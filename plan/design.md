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
