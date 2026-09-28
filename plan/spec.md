# Spec — Search Deduplication System

## Behaviors

- The system normalizes incoming URLs before crawling by stripping tracking parameters (utm_*, fbclid, etc.), normalizing http/https protocols, removing trailing slashes, and converting to lowercase to detect URL-based duplicates.
- When a document is crawled, the system generates a SHA-256 content hash from cleaned body text and computes a SimHash/MinHash fingerprint for near-duplicate detection at 95-99% similarity threshold.
- During document ingestion, the system compares the normalized URL and content fingerprints against existing documents and blocks storage if a duplicate is detected.
- When duplicates are detected, the system selects the canonical version based on metadata completeness (presence of title, author, publication date, description) and stores the duplicate with a hidden flag and reference to the canonical document.
- The system exposes a dashboard interface showing detected duplicates with similarity scores, canonical selections, and duplicate metadata for admin review and approval.
- Administrators can approve or override canonical selections from the dashboard, manually merge duplicate entries, or mark false positives as unique documents.
- The system processes existing crawled documents in the background to identify and consolidate duplicates already in the index without blocking new crawls.
- Search results exclude documents marked as hidden duplicates and only return canonical versions.
- Users can report duplicate results via a "Report Duplicate" button on search result pages, which adds entries to a manual review queue with priority scoring.
- The system maintains a fingerprint cache (in-memory or Redis) storing the last 10,000 document fingerprints for sub-millisecond duplicate lookups during high-volume crawling.

## APIs / Data Changes

### New Module: `search_engine/deduplicator/`

**URLNormalizer class**
- `normalize(url: str) -> str` — Returns normalized URL with tracking params removed, protocol standardized, trailing slashes removed
- `extract_canonical_from_meta(html: str) -> Optional[str]` — Extracts canonical URL from HTML meta tags

**ContentFingerprinter class**
- `generate_hash(content: str) -> str` — Returns SHA-256 hash of cleaned body text
- `generate_simhash(content: str) -> int` — Returns SimHash fingerprint for near-duplicate detection
- `calculate_similarity(hash1: int, hash2: int) -> float` — Returns similarity percentage (0-100)

**DuplicateDetector class**
- `check_duplicate(url: str, content: str, metadata: dict) -> DuplicateResult` — Returns detection result with canonical document ID if duplicate found
- `select_canonical(docs: List[dict]) -> str` — Returns document ID of canonical version based on metadata completeness
- `mark_as_duplicate(doc_id: str, canonical_id: str, similarity: float) -> None` — Stores duplicate relationship

**DuplicateCache class**
- `get_fingerprint(doc_id: str) -> Optional[str]` — Retrieves cached fingerprint
- `set_fingerprint(doc_id: str, fingerprint: str, ttl: int = 86400) -> None` — Stores fingerprint with TTL
- `search_similar(fingerprint: str, threshold: float) -> List[Tuple[str, float]]` — Returns list of (doc_id, similarity) pairs

### Modified: `search_engine/crawler/crawler.py`

**Crawler class changes**
- Add `deduplicator: DuplicateDetector` instance variable
- Modify `_save_page()` method to call `deduplicator.check_duplicate()` before saving
- Add `skip_duplicate: bool` configuration option (default: True)
- Add `duplicate_action: str` configuration option (values: "block", "mark", "alert")

### Data Schema Changes

**Crawled page JSON structure (existing files in `data/crawled_pages/*.json`)**
```json
{
  "url": "string",
  "normalized_url": "string (NEW)",
  "title": "string",
  "meta_description": "string",
  "content": "string",
  "content_hash": "string (NEW)",
  "simhash": "integer (NEW)",
  "is_duplicate": "boolean (NEW, default: false)",
  "canonical_doc_id": "string | null (NEW)",
  "duplicate_similarity": "float | null (NEW)",
  "crawled_at": "ISO datetime (NEW)",
  "author": "string | null (NEW)",
  "publication_date": "string | null (NEW)"
}
```

**Document map structure (in `data/optimized_index/document_map.json`)**
```json
{
  "doc_id": {
    "url": "string",
    "normalized_url": "string (NEW)",
    "title": "string",
    "description": "string",
    "content_snippet": "string",
    "is_duplicate": "boolean (NEW)",
    "canonical_doc_id": "string | null (NEW)",
    "duplicate_of": "array[string] (NEW - list of doc_ids that are duplicates of this canonical)",
    "metadata_score": "float (NEW - completeness score 0-1)"
  }
}
```

**New: Duplicate tracking database (`data/duplicates/duplicate_registry.json`)**
```json
{
  "canonical_doc_id": {
    "url": "string",
    "duplicates": [
      {
        "doc_id": "string",
        "url": "string",
        "similarity": "float",
        "detected_at": "ISO datetime",
        "status": "pending | approved | rejected",
        "detection_method": "url | exact_hash | simhash"
      }
    ],
    "created_at": "ISO datetime",
    "updated_at": "ISO datetime"
  }
}
```

**New: User reports database (`data/duplicates/user_reports.json`)**
```json
[
  {
    "report_id": "string",
    "doc_id_1": "string",
    "doc_id_2": "string",
    "reported_at": "ISO datetime",
    "query": "string",
    "user_session_id": "string (optional)",
    "status": "pending | reviewed | resolved",
    "priority_score": "float"
  }
]
```

### New API Endpoints (Flask routes in `app.py`)

**GET `/admin/duplicates`**
- Returns dashboard page listing detected duplicates awaiting review
- Query params: `?status=pending|approved|rejected&page=1&limit=50`

**GET `/api/duplicates/list`**
- Returns JSON array of duplicate groups with canonical and duplicate document details
- Response: `{ "groups": [...], "total": int, "page": int }`

**POST `/api/duplicates/approve`**
- Body: `{ "canonical_id": "string", "duplicate_ids": ["string"] }`
- Approves canonical selection and hides duplicates from search results
- Response: `{ "success": bool, "message": string }`

**POST `/api/duplicates/reject`**
- Body: `{ "canonical_id": "string", "duplicate_id": "string" }`
- Marks documents as unique (false positive)
- Response: `{ "success": bool, "message": string }`

**POST `/api/duplicates/override`**
- Body: `{ "old_canonical_id": "string", "new_canonical_id": "string" }`
- Changes which document is canonical in a duplicate group
- Response: `{ "success": bool, "message": string }`

**POST `/api/report-duplicate`**
- Body: `{ "doc_id_1": "string", "doc_id_2": "string", "query": "string" }`
- Accepts user report of duplicate search results
- Response: `{ "success": bool, "report_id": string }`

**GET `/api/duplicates/stats`**
- Returns deduplication statistics
- Response: `{ "total_duplicates": int, "pending_review": int, "storage_saved_mb": float, "duplicate_rate": float }`

## Acceptance Criteria

### AC1: URL Normalization
**Given** a crawler encounters multiple URLs pointing to the same content (e.g., `http://example.com/post?utm_source=twitter`, `https://example.com/post/`, `https://example.com/post`)  
**When** the URLs are processed by the URL normalizer  
**Then** all three URLs are normalized to the same canonical form (`https://example.com/post`)  
**And** only one document is stored in the crawled pages directory  
**And** the normalized URL is saved in the document JSON

### AC2: Exact Duplicate Detection via Content Hash
**Given** two documents with identical body content but different URLs  
**When** the second document is crawled and processed  
**Then** the system generates matching SHA-256 content hashes for both documents  
**And** the second document is marked as `is_duplicate: true`  
**And** the second document references the first document's ID in `canonical_doc_id`  
**And** the duplicate is stored but flagged as hidden  
**And** a duplicate entry is created in the duplicate registry

### AC3: Near-Duplicate Detection via SimHash
**Given** two documents with 95-99% similar content (e.g., same article with minor edits or different ad sections)  
**When** both documents are processed through the fingerprinting system  
**Then** SimHash fingerprints are generated for both documents  
**And** the similarity score is calculated as ≥95%  
**And** the document with less complete metadata is marked as duplicate  
**And** the canonical document is selected based on highest metadata completeness score  
**And** both documents remain stored but only the canonical appears in search results

### AC4: Canonical Selection Based on Metadata
**Given** three duplicate documents with varying metadata completeness:
  - Doc A: title, content only
  - Doc B: title, author, content
  - Doc C: title, author, publication date, description, content  
**When** the deduplication system evaluates the duplicate group  
**Then** Doc C is selected as canonical (metadata score = 1.0)  
**And** Docs A and B are marked as duplicates with references to Doc C  
**And** Doc C's `duplicate_of` array contains IDs for Docs A and B

### AC5: Dashboard Display and Admin Review
**Given** the system has detected 5 duplicate groups awaiting review  
**When** an administrator navigates to `/admin/duplicates?status=pending`  
**Then** the dashboard displays all 5 groups with:
  - Canonical document (title, URL, metadata score)
  - List of duplicate documents with similarity scores
  - Timestamp of detection
  - Action buttons (Approve, Reject, Override Canonical)  
**And** the administrator can approve a group to permanently hide duplicates from search  
**And** the administrator can reject to mark as false positive (documents remain searchable)

### AC6: Search Result Exclusion
**Given** a user searches for a query matching both a canonical document and its hidden duplicates  
**When** the search is executed  
**Then** only the canonical document appears in search results  
**And** hidden duplicates are filtered out before ranking  
**And** the search response does not include documents where `is_duplicate: true` and status is "approved"

### AC7: Background Processing of Existing Documents
**Given** the system has 5,000 existing documents in `data/crawled_pages/` without deduplication metadata  
**When** the background deduplication processor is started  
**Then** the system processes documents in batches (e.g., 100 at a time)  
**And** each document receives `normalized_url`, `content_hash`, and `simhash` fields  
**And** duplicate relationships are identified and stored  
**And** the process completes without blocking new crawls  
**And** a progress log shows percentage complete and duplicates found

### AC8: User Duplicate Reporting
**Given** a user sees two duplicate results in their search  
**When** the user clicks "Report Duplicate" on one result and selects the other duplicate  
**Then** a report is submitted to `/api/report-duplicate` with both document IDs and the search query  
**And** the report is stored in `user_reports.json` with status "pending"  
**And** the priority score is calculated based on query frequency and result ranking  
**And** the report appears in the admin dashboard review queue  
**And** a confirmation message is shown to the user

### AC9: Fingerprint Cache Performance
**Given** the system is crawling 100 documents per minute during peak load  
**When** each document is checked for duplicates  
**Then** fingerprint lookups complete in <10ms for cached entries  
**And** the cache stores the most recent 10,000 fingerprints  
**And** cache hits account for ≥80% of lookups during active crawling  
**And** cache misses trigger a database/file lookup without blocking the crawler

### AC10: False Positive Prevention
**Given** the system processes 1,000 unique documents  
**When** deduplication runs on all documents  
**Then** fewer than 1 unique document (0.1% false positive rate) is incorrectly flagged as duplicate  
**And** administrators can review and reject false positives via the dashboard  
**And** rejected duplicates are marked as unique and re-indexed for search

## Non-Functional Requirements

### Performance
- **Duplicate check latency**: URL normalization and fingerprint generation must complete in <100ms per document to avoid slowing crawler throughput
- **Cache lookup performance**: Fingerprint cache queries must return in <10ms for 80th percentile requests
- **Background processing throughput**: Existing document deduplication must process ≥50 documents per second to handle large backlogs
- **Search query impact**: Filtering duplicates from search results must add <20ms to query latency

### Accuracy
- **Exact duplicate detection**: 100% accuracy for identical content (SHA-256 hash match)
- **Near-duplicate detection**: ≥95% recall for documents with 95-99% similarity using SimHash
- **False positive rate**: <0.1% of unique documents incorrectly flagged as duplicates
- **URL normalization accuracy**: ≥99% of common URL variations correctly normalized to canonical form

### Storage
- **Fingerprint cache size**: Maximum 10,000 entries in memory (~1-2 MB depending on implementation)
- **Duplicate metadata overhead**: <5% storage increase for tracking duplicate relationships, hashes, and status fields
- **Expected storage savings**: ≥20% reduction in total document storage after processing existing duplicates

### Scalability
- **Cache eviction**: LRU eviction policy when cache exceeds 10,000 entries with 24-hour TTL
- **Batch processing**: Background deduplication processes documents in configurable batches (default 100) to manage memory
- **Concurrent access**: Support multiple crawler workers checking duplicates simultaneously without race conditions

### Reliability
- **Graceful degradation**: If cache is unavailable, system falls back to direct database/file lookups without failing
- **Transaction safety**: Canonical selection and duplicate marking operations are atomic to prevent inconsistent states
- **Data persistence**: All duplicate relationships and user reports are persisted to disk immediately

### Usability
- **Dashboard clarity**: Duplicate groups display side-by-side content previews for easy visual comparison
- **One-click actions**: Approve/reject/override actions complete with single button click and immediate feedback
- **Report feedback**: Users receive confirmation within 2 seconds after submitting duplicate report

### Security
- **Admin authentication**: Dashboard routes require authentication (implementation depends on existing auth system)
- **Rate limiting**: User duplicate reports limited to 10 per session to prevent abuse
- **Input validation**: All API endpoints validate document IDs exist before processing actions
