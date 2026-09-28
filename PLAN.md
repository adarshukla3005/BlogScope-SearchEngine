# Search Deduplication system — OpenSpec Plan


---

# Proposal

Title: Proposal — Search Deduplication System
Type: openspec-proposal
Version: 1

# Proposal — Search Deduplication System

## Title
Add document deduplication system to detect and prevent duplicate or highly similar blog content from being stored and indexed multiple times.

## Motivation
The current search engine crawler and indexer store every discovered page without detecting duplicates, leading to redundant storage, inflated index sizes, and degraded search result quality when identical content appears under multiple URLs (e.g., with different query parameters, protocols, or syndicated copies). Users receive confusing results with the same article repeated, while the system wastes resources storing and indexing near-identical content. Based on intake answers, the system needs to normalize URLs, compare content hashes at 95-99% similarity threshold, block duplicates immediately during ingestion, and surface duplicate alerts in a dashboard for review.

## Proposed Change
The deduplication system will be integrated into the document ingestion pipeline within the crawler module. Core components include:

1. **URL Normalizer** — Strips tracking parameters (utm_*, fbclid, etc.), normalizes http/https protocols, removes trailing slashes, and canonicalizes URLs before crawling.

2. **Content Fingerprinting** — Generates SHA-256 hashes of cleaned body text and implements SimHash or MinHash for near-duplicate detection at 95-99% similarity threshold.

3. **Duplicate Detection Module** — Compares incoming documents against existing fingerprints, evaluates meta tags (canonical links, titles, descriptions), and checks publication dates and author metadata.

4. **Canonical Selection Logic** — When duplicates are detected, selects the document with the most complete metadata (title, author, publication date, description) as the canonical version.

5. **Duplicate Storage Strategy** — Stores full duplicate content but marks as hidden with metadata linking to canonical version, preventing appearance in search results while preserving information.

6. **Dashboard Alert System** — Creates review interface showing detected duplicates, their similarity scores, and canonical selections for admin approval.

7. **Background Processor** — Gradually processes existing crawled pages to identify and consolidate duplicates already in the index.

8. **Cache Layer for Fingerprints** — Implements an in-memory Redis cache storing recent document fingerprints (last 10,000 documents) to enable sub-millisecond duplicate lookups during high-volume crawling, with automatic cache invalidation after 24 hours and fallback to database queries for older content.

9. **User Feedback Loop** — Adds a "Report Duplicate" button on search result pages allowing end users to flag duplicate results that the system missed, feeding these reports into a manual review queue with priority scoring based on frequency of reports and search result prominence.

The system will modify `search_engine/crawler/crawler.py` to check for duplicates before storage, add a new `search_engine/deduplicator/` module with fingerprinting and detection logic, extend the data schema to track duplicate relationships, and integrate alerts into the existing Flask web interface.

## Success Criteria
1. **Duplicate Detection Rate** — System successfully identifies and blocks ≥95% of exact URL duplicates (same content, different query params/protocols) during ingestion.

2. **Near-Duplicate Detection** — Accurately detects documents with 95-99% content similarity and consolidates them to a single canonical version, validated against a test set of known duplicate blog posts.

3. **Zero False Blocking** — No legitimate unique documents are incorrectly flagged as duplicates and blocked from indexing (false positive rate <0.1%).

4. **Storage Reduction** — Reduces redundant document storage by at least 20% when processing existing crawled pages through background deduplication.

5. **Dashboard Functionality** — Admin dashboard displays duplicate alerts with similarity scores, proposed canonical selections, and provides approve/reject actions that execute within 2 seconds.

6. **Performance Impact** — Deduplication checks add no more than 100ms latency to document ingestion pipeline, maintaining current crawl throughput.

## Out of Scope
1. **Cross-Domain Duplicate Detection** — Initial version focuses on detecting duplicates within the same domain or blog; syndicated content across different domains will not be deduplicated.

2. **Content Similarity Below 95%** — Documents with <95% similarity (related articles, series posts, partial rewrites) will not be flagged as duplicates in this phase.

3. **Manual Duplicate Merging** — No UI for manually merging or un-merging documents after automatic deduplication; administrators can only approve/reject automatic suggestions.

4. **Historical Analytics** — No reporting on duplicate trends over time, most-duplicated sources, or deduplication effectiveness metrics beyond basic counts.

5. **Real-Time Re-Crawl on Changes** — If a canonical document is updated or deleted, duplicate relationships will not trigger automatic re-evaluation until next scheduled crawl.

---

# Spec

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

---

# Design

# Design — Search Deduplication System

## Architecture Overview

The deduplication system will be integrated into the existing crawler pipeline at `search_engine/crawler/crawler.py`, intercepting documents before they are saved to `data/crawled_pages/`. A new `search_engine/deduplicator/` module will provide URL normalization, content fingerprinting (SHA-256 + SimHash), and duplicate detection logic. The Flask application (`app.py`) will be extended with new dashboard routes to display duplicate alerts and allow admin review. Document metadata in the JSON files will be augmented with deduplication fields (`normalized_url`, `content_hash`, `simhash`, `is_duplicate`, `canonical_doc_id`), and a new `data/duplicates/` directory will store duplicate relationship mappings and fingerprint caches for fast lookups.

## Key Decisions

1. **Use SimHash for near-duplicate detection** — SimHash provides fast hamming-distance-based similarity checks ideal for 95-99% threshold without requiring full pairwise comparisons, unlike MinHash which requires more complex LSH bucketing.

2. **Store fingerprints in JSON files instead of Redis** — The system already uses file-based storage for all data; adding Redis introduces operational complexity and the 10K fingerprint cache can be efficiently managed in-memory with periodic JSON persistence.

3. **Inject deduplication into `Crawler._save_page()` method** — This provides a single interception point that catches all documents regardless of crawl source, ensuring no duplicates bypass detection.

4. **Select canonical based on metadata completeness score** — Computing a simple score (1 point per field: title, author, publication_date, meta_description) provides deterministic, explainable canonical selection without requiring ML models.

5. **Use MD5 URL hashes as document IDs** — The crawler already generates MD5 hashes of URLs as filenames (`_get_url_hash()`), providing stable identifiers that work across the crawler, indexer, and deduplicator.

6. **Implement gradual background processing** — Processing existing documents in batches during idle periods (configurable batch size, sleep intervals) prevents system overload while allowing ongoing crawls to continue.

## Module Boundaries

### New Modules

- `search_engine/deduplicator/__init__.py` — Package initialization
- `search_engine/deduplicator/url_normalizer.py` — URLNormalizer class for URL canonicalization and meta tag extraction
- `search_engine/deduplicator/fingerprinter.py` — ContentFingerprinter class for SHA-256 and SimHash generation
- `search_engine/deduplicator/detector.py` — DuplicateDetector class orchestrating duplicate checks and canonical selection
- `search_engine/deduplicator/cache.py` — FingerprintCache class for in-memory LRU cache with JSON persistence
- `search_engine/deduplicator/background_processor.py` — BackgroundProcessor class for batch processing existing documents

### Modified Modules

- `search_engine/crawler/crawler.py` — Add `DuplicateDetector` instance, modify `_save_page()` to check duplicates, add configuration options (`enable_deduplication`, `duplicate_action`)
- `app.py` — Add `/admin/duplicates` route for dashboard, `/api/duplicates/approve` and `/api/duplicates/override` endpoints for admin actions
- `requirements.txt` — Add `simhash` library for near-duplicate detection

### Data Changes

- Existing crawled page JSON files in `data/crawled_pages/*.json` — Add new fields: `normalized_url`, `content_hash`, `simhash`, `is_duplicate`, `canonical_doc_id`, `duplicate_similarity`, `duplicate_detected_at`
- New directory `data/duplicates/` — Store `fingerprint_cache.json` (last 10K fingerprints), `duplicate_log.json` (audit trail), `canonical_mappings.json` (doc_id → canonical_id index)

## External Dependencies

- **simhash** (latest stable, ~3.4.x) — Pure Python implementation for near-duplicate detection using locality-sensitive hashing; chosen over alternatives because it's lightweight and doesn't require C extensions
- **beautifulsoup4** (already in requirements.txt) — Used for extracting canonical link tags from HTML meta tags
- **Flask** (already in requirements.txt) — Extended with new admin dashboard routes

No external services (databases, caches, APIs) are required; the system remains self-contained and file-based to match the existing architecture.

## Trade-offs & Risks

**Performance risk**: Computing SimHash for every document during crawling adds ~50-100ms per page; mitigated by in-memory fingerprint cache that enables fast lookups for recent documents without full comparison. If crawl throughput becomes a bottleneck, fingerprint computation can be moved to async background task after initial save. **False positive risk**: Aggressive similarity thresholds (>95%) may incorrectly flag legitimately different content as duplicates (e.g., template-heavy blogs with minimal unique content per post); mitigated by admin dashboard review and manual override capability, plus the ability to adjust threshold via configuration. **Storage trade-off**: Retaining full duplicate content (instead of just metadata) doubles storage for duplicate documents but enables future re-evaluation if canonical selection logic improves or administrators want to restore hidden duplicates. Alternative considered was delta compression, but rejected due to implementation complexity and uncertain retrieval performance.

---

# Technical

# Technical Tasks — Search Deduplication System

## Implementation Tasks

### Phase 1: Core Deduplication Module

1. Create `search_engine/deduplicator/__init__.py` that exports `URLNormalizer`, `ContentFingerprinter`, `DuplicateDetector`, `FingerprintCache`, and `BackgroundProcessor` classes.

2. Create `search_engine/deduplicator/url_normalizer.py` that implements `URLNormalizer` class with `normalize(url)` method to strip tracking parameters (utm_*, fbclid, gclid, etc.), normalize http/https protocols, remove trailing slashes, convert to lowercase, and sort query parameters; add `extract_canonical_from_meta(html)` method to parse `<link rel="canonical">` tags from HTML using BeautifulSoup.

3. Create `search_engine/deduplicator/fingerprinter.py` that implements `ContentFingerprinter` class with `generate_hash(content)` method returning SHA-256 hash of cleaned/normalized text, `generate_simhash(content)` method using simhash library to create 64-bit fingerprint, and `calculate_similarity(hash1, hash2)` method computing hamming distance and converting to similarity percentage (0-100).

4. Create `search_engine/deduplicator/cache.py` that implements `FingerprintCache` class with in-memory LRU cache (maxsize=10000) storing document fingerprints; add `get_fingerprint(doc_id)`, `set_fingerprint(doc_id, fingerprint, metadata)`, and `search_similar(fingerprint, threshold)` methods that iterate cache to find documents above similarity threshold; implement `save_to_disk()` and `load_from_disk()` methods to persist cache to `data/duplicates/fingerprint_cache.json`.

5. Create `search_engine/deduplicator/detector.py` that implements `DuplicateDetector` class orchestrating URL normalization, fingerprinting, and cache lookups; add `check_duplicate(url, content, metadata)` method that normalizes URL, generates fingerprints, searches cache for matches above 95% threshold, and returns `DuplicateResult` dataclass with `is_duplicate`, `canonical_id`, `similarity`, and `reason` fields; add `select_canonical(docs)` method scoring documents based on metadata completeness (title, author, publication_date, meta_description) and selecting highest score; add `mark_as_duplicate(doc_id, canonical_id, similarity)` method writing relationship to `data/duplicates/duplicate_mappings.json`.

6. Create `search_engine/deduplicator/background_processor.py` that implements `BackgroundProcessor` class to scan existing crawled pages in `data/crawled_pages/`; add `process_batch(batch_size, sleep_interval)` method that loads documents in batches, runs duplicate detection, updates JSON files with new deduplication fields, and logs progress; add `process_all()` method to iterate all existing documents with configurable concurrency.

7. Create `data/duplicates/` directory structure with `.gitkeep`, initialize empty `fingerprint_cache.json`, `duplicate_mappings.json` (format: `{"doc_id": {"canonical_id": "...", "similarity": 0.98, "detected_at": "..."}}`), and `duplicate_log.json` for audit trail.

### Phase 2: Crawler Integration

8. Modify `search_engine/crawler/crawler.py` in `__init__()` method to instantiate `DuplicateDetector` as `self.deduplicator`, add `enable_deduplication` config option (default: True), and add `duplicate_action` config option (values: "block", "mark", "alert", default: "block").

9. Modify `search_engine/crawler/crawler.py` in `_save_page()` method (around line 180) to call `self.deduplicator.check_duplicate(url, content, metadata)` before saving; if duplicate detected and `duplicate_action == "block"`, log the duplicate and return early without saving; if `duplicate_action == "mark"`, save with `is_duplicate=True` and additional fields.

10. Update `search_engine/crawler/crawler.py` to modify page metadata JSON structure by adding fields: `normalized_url`, `content_hash`, `simhash`, `is_duplicate` (default: false), `canonical_doc_id` (null if not duplicate), `duplicate_similarity` (float 0-100), and `duplicate_detected_at` (ISO timestamp).

11. Modify `search_engine/crawler/crawler.py` in `_get_url_hash()` method to use normalized URL instead of original URL for generating document ID, ensuring duplicate URLs with different query params/protocols map to same ID.

### Phase 3: Web Dashboard

12. Create `web/templates/admin_duplicates.html` template that displays table of detected duplicates with columns: original URL, canonical URL, similarity score, detection date, and action buttons (Approve, Override, Mark Unique); add filters for similarity threshold, date range, and approval status; include pagination for large result sets.

13. Modify `app.py` to add `/admin/duplicates` route (GET) that loads duplicate mappings from `data/duplicates/duplicate_mappings.json`, enriches with document metadata (titles, URLs), and renders `admin_duplicates.html` with paginated results.

14. Modify `app.py` to add `/api/duplicates/approve` endpoint (POST) that accepts `doc_id` and confirms canonical selection, updating document JSON file with `duplicate_approved=True` field and logging action to `duplicate_log.json`.

15. Modify `app.py` to add `/api/duplicates/override` endpoint (POST) that accepts `doc_id` and `new_canonical_id` to manually change canonical selection; updates `duplicate_mappings.json`, recalculates fingerprint cache, and logs override action.

16. Modify `app.py` to add `/api/duplicates/mark-unique` endpoint (POST) that accepts `doc_id` and removes duplicate marking, setting `is_duplicate=False` in document JSON and removing entry from `duplicate_mappings.json`.

17. Modify `web/templates/search_results.html` to add "Report Duplicate" button next to each result that posts to `/api/duplicates/report` endpoint with `doc_id` and `query`; create simple modal for user feedback about why they think it's a duplicate.

18. Modify `app.py` to add `/api/duplicates/report` endpoint (POST) that accepts user reports, stores in `data/duplicates/user_reports.json` with priority scoring based on frequency, and adds to manual review queue displayed in admin dashboard.

19. Create `web/static/js/duplicates.js` with JavaScript functions for approve/override/mark-unique actions, AJAX calls to API endpoints, toast notifications for success/error, and dynamic table row updates without page reload.

### Phase 4: Search Integration

20. Modify `search_engine/indexer/optimized_indexer.py` in `build_index()` method to skip documents where `is_duplicate == True` when loading from `data/crawled_pages/`, ensuring hidden duplicates don't appear in inverted index.

21. Modify `search_engine/indexer/optimized_indexer.py` in search methods to filter results excluding documents with `is_duplicate == True`, preventing duplicates from appearing in any search mode (BM25, vector, hybrid).

22. Create `utils/deduplication_config.py` with configuration constants: `SIMILARITY_THRESHOLD = 0.95`, `CACHE_SIZE = 10000`, `CACHE_TTL_SECONDS = 86400`, `TRACKING_PARAMS = ["utm_source", "utm_medium", "utm_campaign", "fbclid", "gclid", "ref"]`, `BATCH_SIZE = 100`, `BATCH_SLEEP_INTERVAL = 0.5`.

### Phase 5: Background Processing & Migration

23. Create `scripts/run_deduplication.py` CLI script that imports `BackgroundProcessor`, accepts command-line arguments for batch size and dry-run mode, processes all existing crawled pages, and outputs summary statistics (duplicates found, canonical selections, storage saved).

24. Modify `scripts/run_deduplication.py` to add `--reprocess` flag that clears existing duplicate mappings and re-runs detection on all documents, useful for adjusting similarity thresholds or fixing false positives.

25. Create `.env.example` file documenting new environment variables: `ENABLE_DEDUPLICATION=true`, `DUPLICATE_ACTION=block`, `SIMILARITY_THRESHOLD=0.95`, `CACHE_SIZE=10000`.

## Tests to Add

### Unit Tests

26. Create `tests/test_url_normalizer.py` with tests for URL normalization covering: tracking parameter removal (utm_*, fbclid), protocol normalization (http→https), trailing slash removal, lowercase conversion, query parameter sorting, and canonical tag extraction from HTML samples.

27. Create `tests/test_fingerprinter.py` with tests for content fingerprinting covering: SHA-256 hash stability (same input = same hash), SimHash generation, similarity calculation returning correct percentages for known similar/dissimilar texts (95%, 98%, 50%, 0%).

28. Create `tests/test_detector.py` with tests for duplicate detection logic covering: exact URL duplicates (same content, different query params), near-duplicate detection (95-99% similar content), canonical selection based on metadata completeness, false negative prevention (different content = not duplicate).

29. Create `tests/test_cache.py` with tests for fingerprint cache covering: LRU eviction when exceeding 10K entries, similarity search returning matches above threshold, persistence to/from JSON file, cache invalidation.

### Integration Tests

30. Create `tests/test_crawler_deduplication.py` with integration tests covering: crawler blocking duplicate URLs during ingestion, duplicate metadata fields correctly populated in saved JSON files, normalized URLs used as document IDs, background processor updating existing documents.

31. Create `tests/test_search_filtering.py` with integration tests verifying: search results exclude documents with `is_duplicate=True`, indexer skips duplicates during index building, canonical documents appear in results while duplicates are hidden.

### Acceptance Tests (from Spec)

32. Create `tests/test_acceptance_duplicate_detection.py` testing: ≥95% detection rate for exact URL duplicates with different query params/protocols, accurate detection of 95-99% similar content, false positive rate <0.1% (no unique documents blocked), canonical selection chooses document with most complete metadata.

33. Create `tests/test_acceptance_dashboard.py` with Selenium/Playwright tests for: admin dashboard displays duplicate pairs with similarity scores, approve/override/mark-unique actions update backend state, "Report Duplicate" button on search results submits user feedback, manual review queue displays user reports with priority.

34. Create `tests/test_acceptance_storage.py` with tests verifying: storage reduction ≥20% when processing existing documents, duplicate documents stored with hidden flag and canonical reference, background processor completes without blocking new crawls.

## Deployment Notes

### Environment Variables

- `ENABLE_DEDUPLICATION` (default: `true`) — Master switch to enable/disable deduplication system
- `DUPLICATE_ACTION` (default: `block`) — Action on duplicate detection: `block` (don't save), `mark` (save but flag), `alert` (log only)
- `SIMILARITY_THRESHOLD` (default: `0.95`) — Minimum similarity (0-1) to consider documents as near-duplicates
- `CACHE_SIZE` (default: `10000`) — Number of fingerprints to keep in memory cache
- `DEDUP_BATCH_SIZE` (default: `100`) — Number of documents to process per batch in background processor
- `DEDUP_SLEEP_INTERVAL` (default: `0.5`) — Seconds to sleep between batches to avoid overload

### Data Migration

1. Run `python scripts/run_deduplication.py --dry-run` to preview duplicate detection on existing documents without making changes
2. Backup `data/crawled_pages/` directory: `cp -r data/crawled_pages data/crawled_pages.backup`
3. Run `python scripts/run_deduplication.py` to process all existing documents and populate duplicate mappings
4. Verify duplicate mappings in `data/duplicates/duplicate_mappings.json` and check logs
5. Rebuild search index with `python -m search_engine.indexer.optimized_indexer` to exclude newly marked duplicates

### New Dependencies

- Add `simhash==2.1.2` to `requirements.txt` for near-duplicate detection (pure Python, no C dependencies)
- Run `pip install -r requirements.txt` to install new dependency

### Directory Structure

- Create `data/duplicates/` directory with proper permissions (writable by app user)
- Ensure `fingerprint_cache.json`, `duplicate_mappings.json`, `duplicate_log.json`, and `user_reports.json` are writable

### Monitoring

- Monitor `data/duplicates/duplicate_log.json` for duplicate detection activity and admin actions
- Track false positive reports in `data/duplicates/user_reports.json` to tune similarity threshold
- Monitor crawler logs for "Duplicate blocked" messages to verify deduplication is active
- Check dashboard at `/admin/duplicates` weekly to review and approve pending duplicates

### Performance Considerations

- Fingerprint cache limited to 10K entries to prevent memory bloat (configurable via `CACHE_SIZE`)
- Background processor runs in batches with sleep intervals to avoid CPU spikes
- SimHash similarity checks are O(1) hamming distance calculations, very fast
- Consider running background processing during off-peak hours for large datasets

## Rollback Plan

If the deduplication system causes issues in production:

1. **Immediate Mitigation**: Set `ENABLE_DEDUPLICATION=false` in environment variables and restart the Flask app and any running crawlers. This disables duplicate checking but preserves existing data.

2. **Restore Search Results**: If legitimate documents were incorrectly marked as duplicates and hidden from search:
   - Run `python scripts/run_deduplication.py --clear-flags` to set `is_duplicate=False` on all documents
   - Rebuild the search index with `python -m search_engine.indexer.optimized_indexer` to re-include all documents
   - This takes 5-15 minutes depending on corpus size but restores full search functionality

3. **Data Recovery**: If documents were blocked from ingestion (lost):
   - Restore `data/crawled_pages/` from backup: `rm -rf data/crawled_pages && cp -r data/crawled_pages.backup data/crawled_pages`
   - Clear duplicate mappings: `rm data/duplicates/duplicate_mappings.json && echo '{}' > data/duplicates/duplicate_mappings.json`
   - Re-crawl missing URLs from seed list with deduplication disabled

4. **Code Rollback**: If bugs require removing the feature entirely:
   - Revert commits introducing `search_engine/deduplicator/` module
   - Restore original `search_engine/crawler/crawler.py` from version control
   - Remove simhash dependency from `requirements.txt` and reinstall: `pip install -r requirements.txt`
   - Remove admin routes from `app.py` (optional, they will just 404 if deduplicator doesn't exist)

5. **Verification**: After rollback, verify crawler saves new pages normally, search returns expected results, and no errors appear in `crawler.log` or `web.log`.

The system is designed to be safe to disable without data loss — setting `ENABLE_DEDUPLICATION=false` reverts to original behavior immediately. The only risk is blocked duplicates during the window when deduplication was active, which can be recovered by re-crawling or restoring from backup.
