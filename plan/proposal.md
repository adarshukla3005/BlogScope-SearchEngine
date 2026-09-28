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
