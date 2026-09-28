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
