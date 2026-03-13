# Social Media Backend Architecture — Compact Overview

> Researched 2026-03-13. See full version: `Social-Media-Backend-Architecture-Research.md`

---

## 1. Twitter/X

- **Content entity**: Tweet object with Snowflake ID (64-bit, time-sorted, embeds datacenter+worker+sequence)
- **Polymorphism**: `attachments` pattern — tweets reference media/polls/places via ID associations, not embedding
- **Storage**: Migrated from MySQL (Gizzard sharding) → **Manhattan** (custom multi-tenant distributed KV store, tens of millions QPS)
- **Fan-out**: Hybrid — fan-out-on-write for normal users (tweet ID pushed to every follower's cached timeline in Redis), fan-out-on-read for celebrities (merged at read time to avoid write amplification)
- **Media**: Blobstore (hundreds of thousands RPS) → CDN
- **Other systems**: Twemcache (20TB+ in-memory, 2T queries/day), FlockDB (social graph), MetricsDB (5B metrics/min, Gorilla compression)

## 2. Instagram

- **DB**: Sharded PostgreSQL (structured) + Cassandra (feeds/activity, write-heavy)
- **Sharding**: Thousands of logical shards → fewer physical shards (PG schemas). Shard ID embedded in primary key (41-bit timestamp + 13-bit shard ID + 10-bit sequence)
- **Content model**: Posts table (`post_id, user_id, caption, media_type enum, location_id`) + Media table (`media_id, post_id, media_url, thumbnail_url, width, height, media_order`). Stories/Reels have separate metadata tables
- **Media pipeline**: Upload → Facebook Haystack (consolidated blob storage, 1 disk read/image) → async transcode/thumbnail → CDN
- **Feed**: Precomputed (fan-out-on-write) + Memcached with lease mechanism
- **Stack**: Django, RabbitMQ, Celery, Gearman

## 3. LinkedIn

- **Storage**: Built **Espresso** — distributed document store, Avro schemas, hierarchical keys (`DB → Table → Collection → Document`), MySQL/InnoDB engine underneath. Zero-downtime schema evolution. Hundreds of TB, millions of records/sec
- **Feed**: Multi-stage ranking — First Pass Rankers (multiple sources) → Second Pass (logistic regression scoring) → Re-ranker (diversity). Handles polymorphic content (articles, posts, jobs, ads) in single pipeline
- **Stack**: Kafka, Rest.li, RocksDB, Caffeine cache, Galene (Lucene-based search)

## 4. TikTok / ByteDance

- **Storage**: ByteGraph (graph DB — tens of billions vertices, trillions edges, edge-trees for super vertices), ByteKV (consistent KV with MVCC), ByteHouse (real-time analytics warehouse)
- **Video pipeline**: Chunked upload → transcode H.264/H.265 (720p first) → 30+ AI models (content analysis, moderation, thumbnails) → CDN
- **Distribution**: ByteDance Edge Nodes (BEN) — caching, streaming, edge ML inference, GPU transcoding
- **Recommendation**: Spark + Flink for real-time behavior streaming → deep learning models → feature stores

## 5. YouTube

- **DB**: Created **Vitess** (open-source MySQL sharding, now CNCF) — VTGate (stateless query router) + VTTablet (per-shard agent) + topology service. Tens of thousands of MySQL nodes
- **Other storage**: Bigtable (watch history, creator analytics, warehouse), Spanner (globally consistent transactions), GCS (video renditions)
- **Video pipeline**: Upload → chunk into 100-200 segments → parallel transcode (H.264, VP9, AV1; 144p–8K) with TPUs/ASICs → auto thumbnails, transcripts, monetization eval → GCS → Google CDN
- **Analytics**: Bigtable-backed warehouse, three-stage pipeline (raw → clean → efficient), Change Streams for invalidation

## 6. Content Scheduling SaaS (Buffer, Hootsuite, Postiz)

- **Dominant pattern**: Platform-agnostic `Post` record + provider abstraction layer. Platform-specific transforms happen at publish time, not storage time. (This is what our `content_items` + `content_platform_posts` does)
- **State machine**: `DRAFT → SCHEDULED → PUBLISHED / ERROR (retry)`
- **Scheduling approaches**: Cron polling (Buffer original) → Redis+BullMQ workers (Postiz/Mixpost) → **Temporal.io durable workflows** (Postiz current, most robust)
- **Hootsuite**: Shared scheduler + platform-specific workers → Kafka event bus, K8s, millions of jobs/sec
- **Buffer stack**: Node.js, TypeScript, GraphQL, MongoDB. BUDA pipeline: Kinesis → S3 → Redshift → dbt → Looker

## 7. CMS Platforms

- **WordPress**: Single Table Inheritance — `wp_posts` stores all types via `post_type` discriminator. Revisions = same table. Metadata via EAV (`wp_postmeta`). Flexible but slow at scale
- **Ghost**: Lexical JSON content format, Bookshelf.js ORM, SQLite3/MySQL, pluggable storage (S3/Azure)
- **Strapi 5**: Draft + published = **separate rows sharing `document_id`**. `published_at` NULL = draft. Cleaner than snapshot JSONB versioning
- **Contentful**: Spaces → Environments → Content Types → Entries. Link-based content graph. Auto-generated GraphQL
- **Sanity**: Schema-as-code (TypeScript), Content Lake (real-time JSON store), GROQ query language, live collaboration

## 8. Email & Blogging

- **Mailchimp**: Multi-list subscriber model (duplicates possible), sharded by user ID, 157B emails/month
- **ConvertKit**: Single-subscriber DB, tag-based segmentation only — 24% higher conversion than multi-list
- **Medium**: Block-based JSON documents (not HTML), polyglot persistence (doc DB + PG + Elasticsearch), version per edit
- **A/B testing**: Up to 8 variants, independent random selection per send, statistical significance for winner

## 9. Key Architectural Patterns

| Pattern | Who | What |
|---------|-----|------|
| Snowflake IDs | Twitter, Instagram, Discord | 64-bit time-sorted IDs with embedded shard/machine info |
| Hybrid fan-out | Twitter, Instagram | Write for normal users, read for celebrities |
| Sharded SQL | Instagram (PG), YouTube (Vitess/MySQL) | Logical→physical shard mapping, shard key in PK |
| Platform-agnostic post + provider | Buffer, Postiz, Hootsuite | Canonical record + per-platform adapter at publish |
| Durable workflow scheduling | Postiz (Temporal.io) | Built-in retry, timeout, failure handling |
| Block-based content | Medium, Ghost (Lexical) | JSON document blocks, not monolithic HTML |
| Dual-row versioning | Strapi 5 | Draft + published as separate rows, shared document_id |
| Real-time media transforms | Cloudflare, ImageKit | URL-based resize/crop on-the-fly vs pre-generating variants |
| Event streaming | Buffer (Kinesis), LinkedIn (Kafka) | Decouple producers/consumers, enable replay |
| Polymorphic content | Twitter, Instagram, WordPress | Type discriminator + type-specific metadata (JSONB or child tables) |

## 10. Takeaways for GTM Dashboard

1. Our `content_items` + `content_platform_posts` fan-out = industry standard (Buffer/Postiz pattern)
2. BullMQ for v1 scheduling is solid; Temporal.io is the upgrade path
3. Strapi 5's dual-row versioning may be cleaner than our `content_versions` snapshot JSONB
4. URL-based real-time media transforms could replace pre-generated `processed_variants`
5. ConvertKit's single-subscriber + tags model is better for newsletters than multi-list
6. Block-based JSON content (Medium/Lexical) aligns with our JSONB approach for rich content
