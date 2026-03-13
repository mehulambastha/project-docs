# Social Media & Content Platform Backend Architecture Research

> Researched 2026-03-13 — How major platforms handle content storage, distribution, and management at scale.

---

## Table of Contents

1. [Twitter/X](#1-twitterx)
2. [Instagram](#2-instagram)
3. [LinkedIn](#3-linkedin)
4. [TikTok / ByteDance](#4-tiktok--bytedance)
5. [YouTube](#5-youtube)
6. [Content Scheduling SaaS (Buffer, Hootsuite, etc.)](#6-content-scheduling-saas)
7. [CMS Platforms (WordPress, Ghost, Strapi, Contentful, Sanity)](#7-cms-platforms)
8. [Email Newsletter Platforms](#8-email-newsletter-platforms)
9. [Blogging Platforms (Medium, HubSpot)](#9-blogging-platforms)
10. [Notable Backend Patterns](#10-notable-backend-patterns)
11. [Cross-Cutting Architectural Patterns Summary](#11-cross-cutting-architectural-patterns-summary)

---

## 1. Twitter/X

### Core Content Entity: The Tweet Object

The Tweet object is a rich, hierarchical entity with root-level fields:

- `id` (Snowflake ID — 64-bit), `text`, `created_at`, `author_id`, `conversation_id`
- `in_reply_to_user_id`, `referenced_tweets` (for retweets/quotes/replies)
- `attachments` (linking to `media_keys` and `poll_ids`)
- `entities` object containing arrays of: hashtags, mentions, URLs, cashtags
- `extended_entities` for polymorphic media (type field: `photo`, `video`, `animated_gif`)
- `public_metrics` (retweet_count, reply_count, like_count, quote_count)

**Polymorphic content** is handled via the `attachments` pattern — tweets can reference media, polls, or places through ID-based associations rather than embedding. Up to 4 photos OR 1 video/GIF per tweet.

### Snowflake ID Structure (64-bit)

```
| 1 bit (sign=0) | 41 bits (timestamp ms since epoch 2010-11-04) | 5 bits (datacenter) | 5 bits (worker) | 12 bits (sequence) |
```

- ~69 years of timestamps, 1024 nodes, 4096 IDs/ms/node
- IDs are roughly time-sorted, enabling efficient range queries
- Adopted by Discord, Instagram (modified), and many others

### Database Evolution: MySQL → Manhattan

Twitter maintained one of the world's largest MySQL installations, originally using the **Gizzard** distributed datastore framework for sharding. In 2014, they launched **Manhattan**, a custom real-time, multi-tenant distributed key-value database.

**Manhattan Architecture (4 layers):**

- **Interfaces**: Client-facing APIs (key/value, graph)
- **Storage Services**: Multi-tenancy, quota management
- **Storage Engines**: Pluggable (LSM-based engines)
- **Core**: Failure handling, eventual consistency, topology (via ZooKeeper), intra/inter-datacenter replication, conflict resolution

Manhattan stores tweets, DMs, account details, and handles tens of millions of QPS.

### Fan-Out Architecture

Twitter uses a **hybrid fan-out** approach:

- **Fan-out-on-write** for normal users: When a user tweets, the tweet ID is written into a cached timeline object (backed by Redis/Haplo) for every follower. This amplifies writes (e.g., 4.6K tweets/sec → ~345K timeline writes/sec) but makes reads trivially fast.
- **Fan-out-on-read** for high-follower accounts (celebrities): Tweets from users with millions of followers are NOT fanned out at write time. Instead, they are merged into the timeline at read time to avoid write amplification.

### Additional Storage Systems

| System | Purpose |
|--------|---------|
| **Blobstore** | Media storage (images, videos) — hundreds of thousands of RPS |
| **Twemcache** | Custom Memcached fork, 20TB+ in-memory, ~2 trillion queries/day |
| **FlockDB** | Graph database for social relationships (follower graph) |
| **MetricsDB** | Time-series DB ingesting 5 billion metrics/minute (Gorilla compression, 95% space reduction) |
| **Hadoop** | 500+ PB across tens of thousands of instances for analytics |

### Media Processing

Media files stored in Blobstore. URLs and metadata stored alongside tweet entities. CDN layer handles global distribution.

---

## 2. Instagram

### Database Architecture

- **PostgreSQL (structured data)**: User profiles, posts, comments, relationships, authentication. Leader-follower replication for read scaling.
- **Cassandra (distributed/write-heavy data)**: User feeds, activity logs, analytics data. Eventual consistency, high write throughput.

### Sharding Strategy (PostgreSQL)

Instagram evaluated NoSQL but chose to shard PostgreSQL instead:

- Thousands of **logical shards** mapped to fewer **physical shards** (PostgreSQL schemas)
- Each logical shard is a Postgres schema containing all sharded tables (likes, comments, photos, etc.)
- As load grows, logical shards are remapped to new physical servers

**ID Generation (Instagram's modified Snowflake):**

- 41 bits: timestamp (ms since custom epoch)
- 13 bits: logical shard ID
- 10 bits: auto-incrementing sequence (per-shard, per-table PostgreSQL sequences)

This embeds the shard ID directly in the primary key, making routing trivial.

### Content Types and Polymorphic Handling

Instagram handles: Posts (photos/carousels), Stories (ephemeral), Reels (short video), Live streams, IGTV.

Typical entity structure:

- **Posts table**: `post_id`, `user_id`, `caption`, `media_type` (enum: photo/video/carousel), `location_id`, `created_at`
- **Media table**: `media_id`, `post_id`, `media_url`, `thumbnail_url`, `width`, `height`, `size_bytes`, `media_order` (for carousels)
- **Stories**: Separate metadata table with `expiry_time`, viewer tracking, sticker/interaction overlays
- **Reels**: Video-specific metadata (`duration`, `encoding_status`, `audio_track_id`)

### Media Storage Pipeline

1. User uploads media → synchronous **Media Service** uploads to **Facebook Haystack** (optimized blob storage minimizing filesystem operations)
2. Asynchronous job generates multiple resolutions/thumbnails
3. Processed media pushed to **CDN** edge locations
4. Metadata persisted to graph data storage
5. **Feed Service** triggered asynchronously to precompute followers' feeds

**Haystack** stores many small images in large consolidated files, requiring only one disk read per image.

### Feed Generation

- For non-celebrity users: **precomputed feed** (fan-out-on-write), results stored in CDN
- **Memcached** layer caches frequently accessed data with a lease mechanism to prevent thundering herd

### Supporting Stack

Django (web framework), RabbitMQ (async messaging), Celery (background tasks), Gearman (distributed task queue for media uploads)

---

## 3. LinkedIn

### Espresso: Document Store for Content

LinkedIn built **Espresso**, a distributed, fault-tolerant NoSQL document store powering ~30 applications.

**Data Model (hierarchical):**

```
Database → Table → Collection → Document
```

- Document schemas defined in **Avro** format; DB/table schemas in JSON
- Multi-part keys: leading key = partition key, creating hierarchical key-space
- Documents stored as Avro serialized binary blobs with metadata (checksum, timestamp, schema version)

**Routing**: Stateless HTTP proxy examines URL, hashes partition key, routes to correct storage node. Routing table cached locally, synced via ZooKeeper.

**Storage**: Pluggable engines (MySQL/InnoDB in production). Synchronous secondary indexes, transactional support within single partitions.

**Schema Evolution**: Zero-downtime changes via Avro evolution rules. New optional fields maintain backward compatibility without migrations.

**Scale**: Hundreds of terabytes, millions of records/sec at peak, dozens of clusters.

### Feed Architecture

**Multi-stage ranking pipeline:**

1. **First Pass Rankers (FPRs)**: Multiple sources create preliminary rankings — job recommendations, news articles, connection updates, sponsored content
2. **Second Pass Ranker (SPR)**: Logistic regression model scoring P(viewer interacts with update)
3. **Re-ranker**: Modifies ranked list for diversity and engagement properties

**Heterogeneous/Polymorphic Content**: The feed handles articles, shared posts, job recommendations, news, connection suggestions, and sponsored updates — all ranked by the same pipeline but with type-specific features.

**Tech Stack**: Kafka, Rest.li, Espresso, RocksDB, Caffeine cache, Galene (search), Apache Lucene.

---

## 4. TikTok / ByteDance

### Internal Database Systems

**ByteGraph** — Distributed graph database:

- Three-layer architecture: BGE (query engine), BGS (graph shard), persistent storage (RocksDB-backed KV stores)
- Vertex storage: `<key: <vID, vType>, value: <properties>>`
- Edge storage: B-tree-like "edge-trees" indexed by `<vID, vType, eType, direction>`
- Handles super vertices with billions of edges
- Manages **tens of billions of vertices, trillions of edges**
- Within-region: single-leader replication. Cross-region: multi-leader with eventual consistency via Hybrid Logical Clocks

**ByteKV** — Strongly consistent, distributed KV store with MVCC-based snapshot consistency.

**ByteHouse/ByConity** — Cloud-native data warehouse for real-time multimodal analytics.

### Video Upload & Processing Pipeline

1. Video uploaded to object storage (chunked uploads for mobile resilience)
2. Transcoded into multiple formats/resolutions (H.264, H.265). 720p prioritized for immediate availability
3. AI feature extraction (30+ models): content analysis, thumbnail generation, safety/moderation scoring
4. Content moderation: AI detection + human review
5. Metadata written to databases; video chunks distributed to CDN

### Content Distribution

- **ByteDance Edge Nodes (BEN)**: Caching, streaming, and lightweight ML inference at the edge
- Combination of third-party CDN providers and own edge network
- Real-time transcoding with GPUs at edge nodes

### Recommendation Pipeline

- Deep learning models analyze: watch time, likes, comments, shares, completion rate
- **Apache Spark + Flink** for real-time stream processing of user behavior
- Features stored in feature stores for model serving
- Microservices: User Service, Content Service, Feed Service, Interaction Service, Search Service, Analytics Service, Notification Service

---

## 5. YouTube

### Database Architecture: Vitess + MySQL

YouTube created **Vitess** (now CNCF open-source) to horizontally scale MySQL. Tens of thousands of MySQL nodes.

**Vitess Architecture (4 layers):**

- **VTGate**: Stateless query router — applications send SQL without knowing which shard to target
- **VTTablet**: Per-shard agent managing individual MySQL instances, connection pooling, query rewriting
- **VTctld**: Control plane for cluster management
- **Topology Service**: Metadata store (ZooKeeper/etcd/Consul)

**Scaling Strategies:**

- **Vertical splitting**: Separate related table groups into different databases (e.g., user profiles vs. video metadata)
- **Horizontal sharding**: Divide single tables across databases by key (user ID, video ID, date range)

### Additional Storage

| System | Purpose |
|--------|---------|
| **Bigtable** | User watch history, creator analytics, ML feature data, warehouse data |
| **Spanner** | Globally distributed, strongly consistent transactions (user sessions, watch history) |
| **Memcache** | Caching layer |
| **Google Cloud Storage** | Transcoded video renditions |

### Video Processing Pipeline

1. Upload received, assigned unique video ID
2. Video split into chunks (e.g., 2GB 1080p video → 100-200 segments)
3. Parallel transcoding: H.264, VP9, AV1 codecs; 144p to 8K resolutions
4. Hardware acceleration: TPUs + proprietary encoding ASICs for AV1
5. Automated: thumbnail generation, metadata extraction, transcript generation, monetization evaluation
6. Transcoded renditions → Google Cloud Storage; metadata → Bigtable
7. Distribution via Google's custom CDN

### Analytics Pipeline

- **Creator Analytics**: Privacy-compliant viewership data from service logs
- **Data Warehouse (Bigtable-backed)**: Three-stage pipeline — raw data → cleaning/transformation → efficient representation
- **Change Streams**: Upstream data changes invalidate entity rows; streaming pipelines fetch latest data

---

## 6. Content Scheduling SaaS

### The "One Post → Many Platforms" Problem

The industry has converged on two primary patterns:

#### Pattern A: Platform-Agnostic Post + Provider Abstraction (dominant)

Used by **Postiz** (open source) and reflected in Buffer's architecture:

- A single `Post` record stores canonical content (text, media references)
- An `Integration` model represents a connected social media account (channel)
- At publish time, an `IntegrationManager` factory instantiates the correct provider (e.g., `XProvider`, `LinkedInProvider`)
- Each provider implements `post()`, `authenticate()`, `refreshToken()`
- Platform-specific transformations happen in the provider layer, not the data layer

#### Pattern B: Post Variants / Category Queues

Tools like **SocialBee** use "post variants" — multiple variations stored and requeued at intervals. Posts organized into "content categories" (promotional, curated, RSS), each with independent scheduling rules.

### Post Status State Machines

Common across all platforms:

```
DRAFT → SCHEDULED → PUBLISHED
                 \→ ERROR (with retry)
```

Postiz uses: `DRAFT`, `SCHEDULED`, `COMPLETED`, `ERROR`.

### Scheduling Mechanisms

Three patterns emerged:

| Pattern | Used By | How It Works |
|---------|---------|-------------|
| **Cron-based polling** | Buffer (original) | Cron every minute, query DB for due posts, trigger publication |
| **Redis + BullMQ workers** | Postiz (original), Mixpost | Posts with scheduled times in Redis, BullMQ manages queue, horizontal scaling via workers |
| **Temporal.io durable workflows** | Postiz (current) | `startWorkflow()` initiates durable scheduled workflow, built-in retry/timeout/failure handling |

### Hootsuite's Universal Polling System

- **Shared scheduler**: Handles registrations, polling frequencies, scheduling, retrying, rate limit management
- **Platform-specific workers**: Make actual API calls, publish results to **Kafka event bus**
- Runs inside Kubernetes, scales to millions of jobs per second

### Buffer's Stack

- Node.js, TypeScript, GraphQL, MongoDB
- **BUDA (Buffer's Unified Data Architecture)**: AWS Kinesis Streams as event log → S3 as JSON → Redshift via Firehose → dbt for transformation → Looker for dashboarding
- Scale: 2.7 billion tracked user actions, 1.145 billion social media posts created

---

## 7. CMS Platforms

### WordPress: Single Table Inheritance

The `wp_posts` table stores ALL content types in a single table, discriminated by `post_type`:

- `post_type` values: `post`, `page`, `attachment`, `revision`, `nav_menu_item`, plus custom post types
- **Revisions** stored as rows in the same table with `post_type = 'revision'` and `post_status = 'inherit'`, linked via `post_parent`
- **Metadata** uses Entity-Attribute-Value (EAV) via `wp_postmeta`: `meta_id`, `post_id`, `meta_key`, `meta_value`

Extremely flexible (zero schema changes for new content types) but has performance issues with large datasets across the EAV table.

### Ghost CMS: Document Storage

- RESTful JSON API split into **Content API** (read-only, cacheable) and **Admin API** (read-write, role-based auth)
- Bookshelf.js ORM supporting SQLite3 (dev) and MySQL (production)
- Content stored in **Lexical** format (React-based editor by Meta, replaced MobileDoc)
- Pluggable storage adapters (local filesystem, S3, Azure)

### Strapi 5: Document-Based Versioning

- Each entry has a `document_id` (24-char alphanumeric) persisting across all versions
- Draft and published versions stored as **separate database rows** sharing the same `document_id`
- `published_at` field discriminates: `NULL` = draft, `[date]` = published
- Three content states: **Draft** (never published), **Modified** (published but with pending changes), **Published**

```sql
-- With versioning enabled, a single post has 2 rows:
SELECT title, published_at, document_id FROM posts;
-- Row 1: title="My Post", published_at=NULL, document_id="abc123"     (draft)
-- Row 2: title="My Post", published_at="2024-01-15", document_id="abc123"  (published)
```

### Contentful: Structured Content Graph

- **Spaces** → **Environments** → **Content Types** → **Entries**
- Content types have up to 50 fields mapped to JSON types
- Relationships via `Link` type fields forming a **content graph**
- Patterns: **Topics and Assemblies** (content vs. structural containers), **Microcopy**, **Navigation**, **Localization**
- GraphQL schema auto-generated from content model at request time

### Sanity: Content Lake + GROQ

- Content models **defined in code** (TypeScript schema files)
- All content in the **Content Lake** (cloud-hosted, real-time JSON datastore)
- Queried with **GROQ** (Graph-Relational Object Queries), not GraphQL
- Real-time collaboration with live editing, cursors, and automatic conflict resolution

---

## 8. Email Newsletter Platforms

### Subscriber Architecture

| Platform | Model | Notes |
|----------|-------|-------|
| **Mailchimp** | Multi-list with tags/groups | Subscribers can exist in multiple lists (duplicates possible) |
| **ConvertKit** | Single-subscriber-database | Every subscriber exists once, segmentation is tag-based. 24% higher conversion rates |

### Campaign Data Model

- ConvertKit: **Sequences** (automated email sets sent on preset schedules)
- Mailchimp: Multiple builder interfaces (Classic for code-aware, New for visual/collaborative)

### A/B Testing Data Model

- Up to 8 variants per campaign
- Independent random selection per message send (not per-user)
- Variant percentages must add to 100
- Control groups supported as separate segment
- Winner selection via statistical significance across open rate, click rate, conversion rate

### Mailchimp/Mandrill Scale

- Handles up to **157 billion emails per month** with median delivery under 1 second
- Sharding by user ID for data storage
- Database layer is primary bottleneck under load

---

## 9. Blogging Platforms

### Medium: Block-Based Architecture

- Articles stored as **JSON documents** broken into blocks (paragraphs, images, code snippets, embeds), not monolithic HTML
- **Polyglot persistence**: document DBs for article content, PostgreSQL for user profiles, Elasticsearch for search
- Every edit creates a new version for revision history
- On publish: content validation → spam/policy checking → SEO metadata generation → search index updates → CDN caching → social preview image generation
- Published articles aggressively cached

### HubSpot CMS

- Proprietary closed architecture — infrastructure, runtime, database, caching, edge delivery all HubSpot-managed
- Pages server-rendered and cached to CDN
- Dynamic content from **HubDB** (structured data source) using HubL templating

---

## 10. Notable Backend Patterns

### Event Sourcing for Content State Changes

- Domain events: `Page.Created` (with title, slug, body), `Page.BodyUpdated` (using diffs, not full replacement)
- Diffs applied in correct order with exactly-once semantics
- CQRS separation: write model (ContentManagementApplication) and read model (SearchIndexApplication)
- Search index rebuilt by replaying all events
- **Snapshots** optimize loading: periodically save current state, then only replay events since last snapshot

### CMS Versioning with Linked Lists

- **Static ID** (immutable UUID from first version) + **Version ID** (unique per version)
- Versions form a linked list: `previous_version ↔ next_version`
- Four states: `Draft → Submitted → Published → Archived`
- **Reference Map pattern**: content references assets via Static IDs; a Reference Map maintains `Static ID → Version ID` mappings, enabling version swaps without modifying content
- Storage split: graph DB (Dgraph) for relations/metadata, Cassandra for actual content

### CQRS for Read-Heavy Content Feeds

- Separate **write model** (normalized, consistency-optimized) from **read model** (denormalized, query-optimized)
- Read model uses **materialized views** prepopulated from write events
- Caveat: eventual consistency adds complexity; unnecessary for most systems

### Polymorphic Content: Three Database Patterns

| Pattern | Example | Pros | Cons |
|---------|---------|------|------|
| **Single Table Inheritance** | WordPress | Simple queries, easy to add types | Sparse columns, no type-specific constraints |
| **Class Table Inheritance** | GTM Dashboard (our approach) | Clean schema, proper constraints | Joins for every query |
| **Concrete Table Inheritance** | Some microservices | No joins, clear separation | No unified querying, field duplication |

### Media Asset Processing Pipelines

- **Three-layer pipeline**: Generation (raw asset) → Control (parameter/brand consistency) → Refinement (upscaling, format conversion)
- Platform-specific formatting: Instagram square, TikTok vertical, LinkedIn landscape
- Video: preprocessing → encoding (H.264, H.265, VP9) → delivery
- Modern trend: **URL-based real-time transformations** (resize, crop on the fly) rather than pre-generating all variants

---

## 11. Cross-Cutting Architectural Patterns Summary

| Pattern | Used By | Description |
|---------|---------|-------------|
| **Snowflake IDs** | Twitter, Instagram, Discord | 64-bit time-sorted distributed IDs embedding machine/shard info |
| **Fan-out-on-write (hybrid)** | Twitter, Instagram | Precompute timelines at write time; fan-out-on-read for celebrities |
| **Sharded SQL** | Instagram (PG), YouTube (MySQL/Vitess) | Logical shards mapped to physical; shard key in primary key |
| **Custom KV stores** | Twitter (Manhattan), ByteDance (ByteKV) | Purpose-built for specific consistency/latency tradeoffs |
| **Graph databases** | ByteDance (ByteGraph), Twitter (FlockDB) | Social graphs with billions of edges |
| **Document stores** | LinkedIn (Espresso) | Avro schemas, hierarchical key-space, zero-downtime evolution |
| **Chunked video processing** | YouTube, TikTok | Split uploads into segments for parallel transcoding |
| **Edge computing** | TikTok (BEN), YouTube (Google CDN) | ML inference and transcoding at edge nodes |
| **Multi-stage feed ranking** | LinkedIn, TikTok, Instagram | First-pass retrieval → ML scoring → re-ranking for diversity |
| **Event streaming** | Buffer (Kinesis), LinkedIn (Kafka) | Decouple producers/consumers; enable replay and backfill |
| **Cron + queue scheduling** | Buffer, Hootsuite | Minute-level cron checks + message queues for reliable publishing |
| **Polymorphic content** | Twitter (entities), Instagram (media_type enum) | Type discriminator + type-specific metadata |
| **Platform-agnostic post + provider abstraction** | Postiz, Buffer | Canonical content record + per-platform adapter at publish time |
| **Durable workflow engines** | Postiz (Temporal.io) | Scheduled workflows with built-in retry, timeout, failure handling |

---

## Key Takeaways for GTM Dashboard

1. **Our `content_items` + `content_platform_posts` fan-out model aligns with industry best practice** — this is essentially the "platform-agnostic post + provider abstraction" pattern used by Buffer and Postiz.

2. **Snowflake-style IDs** are worth considering if we expect to shard later. Instagram's approach of embedding shard ID in the primary key is elegant.

3. **Strapi 5's dual-row versioning** (draft + published sharing a `document_id`) is a cleaner alternative to our `content_versions` snapshot JSONB approach — worth evaluating.

4. **Temporal.io for scheduling** is the most robust pattern (used by Postiz) — superior to cron polling. But BullMQ is a solid middle ground for v1.

5. **URL-based real-time media transformations** (Cloudflare/ImageKit style) could eliminate the need to pre-generate all `processed_variants` — reducing storage costs significantly.

6. **ConvertKit's single-subscriber model** (tag-based segmentation vs. Mailchimp's multi-list) is worth noting for our newsletter features.

7. **Medium's block-based content storage** (JSON documents, not HTML) is the modern standard for rich content — aligns with our JSONB approach for `video_scripts.script_body`.

---

## Sources

- Twitter: [Snowflake Announcement](https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake), [Infrastructure at Scale](https://blog.x.com/engineering/en_us/topics/infrastructure/2017/the-infrastructure-behind-twitter-scale), [Manhattan DB](https://blog.x.com/engineering/en_us/a/2014/manhattan-our-real-time-multi-tenant-distributed-database-for-twitter-scale)
- Instagram: [Sharding & IDs](https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c), [Scaling Infrastructure (ByteByteGo)](https://blog.bytebytego.com/p/how-instagram-scaled-its-infrastructure)
- LinkedIn: [Espresso](https://engineering.linkedin.com/espresso/introducing-espresso-linkedins-hot-new-distributed-document-store), [Feed Infrastructure](https://engineering.linkedin.com/teams/data/data-infrastructure/feed-infrastructure)
- ByteDance: [ByteGraph (VLDB)](https://vldb.org/pvldb/vol15/p3306-li.pdf), [TikTok System Design](https://grokkingthesystemdesign.com/guides/tiktok-system-design/)
- YouTube: [Vitess + MySQL (ByteByteGo)](https://blog.bytebytego.com/p/how-youtube-supports-billions-of), [Bigtable (Google Cloud)](https://cloud.google.com/blog/products/databases/youtube-runs-on-bigtable)
- Buffer: [Data Architecture](https://buffer.com/resources/evolving-buffers-data-architecture/)
- Postiz: [DeepWiki](https://deepwiki.com/gitroomhq/postiz-app)
- Hootsuite: [Universal Polling System](https://medium.com/hootsuite-engineering/the-universal-polling-system-aa25cb84b38d)
- WordPress: [Database Structure](https://wp-staging.com/docs/the-wordpress-database-structure/)
- Strapi: [Draft & Publish in Strapi 5](https://prototypr.io/note/draft-publish-cms-strapi)
- Contentful: [Data Model](https://www.contentful.com/developers/docs/concepts/data-model/)
- Ghost: [Architecture](https://docs.ghost.org/architecture)
- Medium: [Architecture Design](https://dev.to/matt_frank_usa/designing-medium-blogging-platform-architecture-2a7n)
