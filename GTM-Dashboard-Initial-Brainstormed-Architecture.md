# GTM Dashboard — Initial Brainstormed Architecture

> Social Media & Content Management System for GTM / Marketing Teams
> Goal: Help businesses generate more inbound leads through managed, AI-assisted content across social media, blogs, and email newsletters.

---

## Table of Contents

1. [Product Scope & Supported Content](#1-product-scope--supported-content)
2. [Identity & Tenancy Layer](#2-identity--tenancy-layer)
3. [Connected Accounts & Credential Storage](#3-connected-accounts--credential-storage)
4. [Content Model (Core Tables)](#4-content-model-core-tables)
5. [AI Content Ideation Pipeline](#5-ai-content-ideation-pipeline)
6. [Video Scripting & B-Roll](#6-video-scripting--b-roll)
7. [Scheduling & Platform Rules Engine](#7-scheduling--platform-rules-engine)
8. [Content Lifecycle & Approval Workflow](#8-content-lifecycle--approval-workflow)
9. [Trend Analysis & Niche Research](#9-trend-analysis--niche-research)
10. [Analytics Data Model](#10-analytics-data-model)
11. [Queues, Jobs & Async Pipelines](#11-queues-jobs--async-pipelines)
12. [Rule Engine Orchestrator Design](#12-rule-engine-orchestrator-design)
13. [Supporting Tables](#13-supporting-tables)
14. [High-Level System Diagram](#14-high-level-system-diagram)
15. [Indexes](#15-indexes)
16. [Priority Phasing (v1 → v2 → v3)](#16-priority-phasing-v1--v2--v3)
17. [Open Questions & Future Considerations](#17-open-questions--future-considerations)

---

## 1. Product Scope & Supported Content

### MUST-Support Content Types

| Medium | Sub-types | Platforms |
|--------|-----------|-----------|
| **Social — Photo** | Single image, Carousel | Instagram, Twitter/X, Threads, LinkedIn |
| **Social — Short-form Video** | Reel, TikTok, YouTube Short | Instagram, TikTok, YouTube, Twitter/X |
| **Social — Long-form Video** | Standard video | YouTube, TikTok, LinkedIn |
| **Blog Posts** | Article, Landing page | HubSpot, WordPress, Ghost, Medium |
| **Email Newsletters** | Broadcast, Drip sequence | Mailchimp, SendGrid, ConvertKit, Beehiiv |

### Core Feature Set

- **Draft content** — Write/upload content, get AI feedback, iterate
- **Capture ideas** — Input a raw text idea → AI agent researches and fleshes it out across multiple mediums
- **Schedule content** — Platform-aware scheduling with best-practice rules per medium
- **Trend analysis** — Describe a niche, analyze what's trending (TikTok search APIs, X trending, internet research)
- **Video scripting** — AI-generated short-form and long-form scripts, thumbnail ideas, B-roll suggestions
- **Analytics** — Pull metrics per post, per account, track growth
- **Multi-platform publish** — One piece of content → fan-out to N platforms with platform-specific adaptations

---

## 2. Identity & Tenancy Layer

### `organizations`

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| name | VARCHAR(255) | |
| slug | VARCHAR(100) UNIQUE | URL-safe identifier for routing |
| plan_tier | ENUM('free','pro','enterprise') | Gates feature access (AI calls, # connected accounts, etc.) |
| settings | JSONB | See niche config below |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

**`settings` JSONB structure:**

```jsonc
{
  "timezone": "America/New_York",
  "niche": "B2B SaaS for HR tech",
  "target_audience": "VP of People / HR Directors at 200-2000 employee companies",
  "brand_voice": "professional but approachable, data-driven",
  "competitors": ["Rippling", "BambooHR"],
  "content_pillars": ["future of work", "HR automation", "employee experience"],
  "regions_of_interest": ["US", "UK"],
  "default_approval_required": true
}
```

> The `niche`, `target_audience`, `brand_voice`, and `content_pillars` fields become **system prompt context** for every AI agent call — ideation, scripting, trend analysis, etc.

### `users`

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| org_id | UUID FK → organizations | |
| email | VARCHAR UNIQUE | |
| password_hash | VARCHAR | bcrypt/argon2 |
| mfa_secret | VARCHAR NULL | TOTP for enterprise |
| status | ENUM('active','suspended','invited') | |
| last_login_at | TIMESTAMPTZ | |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

### `user_roles` (RBAC)

| Column | Type | Notes |
|--------|------|-------|
| user_id | UUID FK → users | |
| role | ENUM('owner','admin','editor','analyst','viewer') | |
| scope | JSONB NULL | Optional: restrict to specific connected_account IDs. NULL = full org access |

> **Why scoped roles?** An enterprise might have 40 social accounts. The social media intern should only publish to the brand's TikTok, not the CEO's Twitter.

---

## 3. Connected Accounts & Credential Storage

### `connected_accounts`

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| org_id | UUID FK → organizations | |
| platform | ENUM('instagram','tiktok','threads','twitter','youtube','linkedin','hubspot','wordpress','mailchimp','sendgrid','convertkit','beehiiv') | Extensible |
| platform_account_id | VARCHAR | Platform's native user/page ID |
| platform_username | VARCHAR | Display name / handle |
| account_type | ENUM('personal','business','creator') | IG has different API surfaces per type |
| access_token_enc | BYTEA | AES-256-GCM encrypted (envelope encryption via KMS) |
| refresh_token_enc | BYTEA | AES-256-GCM encrypted |
| token_expires_at | TIMESTAMPTZ | |
| scopes_granted | TEXT[] | What permissions were actually granted during OAuth |
| status | ENUM('active','expired','revoked','reauth_required') | |
| connected_by | UUID FK → users | Audit: who connected it |
| last_synced_at | TIMESTAMPTZ | Last successful API call |
| metadata | JSONB | Platform-specific — see below |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

**Platform-specific `metadata` examples:**

```jsonc
// Instagram (routes through FB Graph API)
{ "facebook_page_id": "...", "ig_business_account_id": "..." }

// YouTube
{ "channel_id": "...", "channel_title": "..." }

// HubSpot
{ "portal_id": "...", "blog_ids": ["..."] }

// Mailchimp
{ "server_prefix": "us19", "list_ids": ["..."] }
```

### Security Design

- **Envelope encryption**: App-level AES-256-GCM with DEK per row, KEK in AWS KMS / HashiCorp Vault
- **Key rotation**: Rotate the KEK without re-encrypting every row
- **Token refresh pipeline**: Cron job every ~15 min queries `WHERE token_expires_at < NOW() + INTERVAL '30 minutes' AND status = 'active'`; failed refreshes → `status = 'reauth_required'` → dashboard notification
- **Scope tracking**: If user declined `publish` scope during OAuth, block scheduling and prompt re-auth

---

## 4. Content Model (Core Tables)

### Design Principle

Content is highly polymorphic across platforms. A single content piece ("New product launch") might become an IG carousel, a Twitter thread, a YouTube Short, a HubSpot blog post, and a newsletter — each with independent publish state, platform-native ID, and platform-specific configuration.

**Approach**: Unified base table + per-platform fan-out via junction table + JSONB for platform-specific metadata.

### `content_items` — The canonical content record

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| org_id | UUID FK → organizations | Tenant isolation |
| created_by | UUID FK → users | |
| content_type | ENUM('post','reel','story','video','short','thread','carousel','blog_article','newsletter') | |
| status | ENUM('idea','draft','pending_approval','approved','scheduled','publishing','published','partially_published','failed','archived') | |
| title | VARCHAR NULL | Optional — relevant for blog, YT, newsletter |
| body_text | TEXT | Primary caption / text / body content |
| hashtags | TEXT[] | Extracted/managed separately for analytics |
| scheduled_at | TIMESTAMPTZ NULL | NULL = publish immediately when approved |
| published_at | TIMESTAMPTZ NULL | |
| campaign_id | UUID FK → campaigns NULL | Group content into campaigns |
| idea_id | UUID FK → content_ideas NULL | Link back to originating idea |
| ai_generated | BOOLEAN DEFAULT false | Flag AI-assisted content |
| metadata | JSONB | Overflow: alt text, first comment, location tag, etc. |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

### `content_media` — Assets attached to content

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| content_id | UUID FK → content_items | |
| media_type | ENUM('image','video','gif','document','audio') | |
| original_url | VARCHAR | S3/CDN path to original upload |
| processed_variants | JSONB | `{ "thumbnail": "url", "1080p": "url", "9_16": "url", "1_1": "url" }` |
| duration_seconds | INT NULL | For video/audio |
| width | INT | |
| height | INT | |
| file_size_bytes | BIGINT | |
| alt_text | VARCHAR NULL | Accessibility |
| sort_order | SMALLINT | Carousel ordering |
| processing_status | ENUM('uploaded','processing','ready','failed') | |
| storage_tier | ENUM('hot','warm','archive') DEFAULT 'hot' | Cost management for video at scale |
| created_at | TIMESTAMPTZ | |

> **Why `processed_variants`?** Instagram wants square/portrait, YouTube wants 16:9, TikTok wants 9:16. The media pipeline must transcode and crop per-platform. All variants stored here.

### `content_platform_posts` — The per-platform fan-out (KEY TABLE)

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| content_id | UUID FK → content_items | |
| connected_account_id | UUID FK → connected_accounts | Which account to post to |
| platform | ENUM(...) | Denormalized for query speed |
| platform_content_type | VARCHAR | Platform-native type (see below) |
| platform_post_id | VARCHAR NULL | Filled after publish — platform's native ID |
| platform_specific_body | TEXT NULL | If caption differs per platform (NULL = use parent body_text) |
| platform_specific_metadata | JSONB | See below |
| status | ENUM('pending','publishing','published','failed','deleted') | Independent of parent |
| error_message | TEXT NULL | On failure |
| published_url | VARCHAR NULL | Permalink on the platform |
| published_at | TIMESTAMPTZ NULL | |
| retry_count | SMALLINT DEFAULT 0 | |
| idempotency_key | UUID UNIQUE | Prevents double-posting on retry |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

**`platform_content_type` values:**

| Platform | Types |
|----------|-------|
| Instagram | `IG_POST`, `IG_CAROUSEL`, `IG_REEL`, `IG_STORY` |
| TikTok | `TT_VIDEO` |
| YouTube | `YT_VIDEO`, `YT_SHORT`, `YT_COMMUNITY_POST` |
| Twitter/X | `TW_TWEET`, `TW_THREAD_PART`, `TW_POLL` |
| Threads | `TH_POST` |
| LinkedIn | `LI_POST`, `LI_ARTICLE` |
| HubSpot | `HS_BLOG_POST`, `HS_LANDING_PAGE` |
| Mailchimp | `MC_CAMPAIGN` |
| Newsletter (generic) | `NL_BROADCAST`, `NL_DRIP` |

**`platform_specific_metadata` JSONB examples:**

```jsonc
// Instagram Reel
{
  "cover_image_media_id": "uuid",
  "share_to_feed": true,
  "collaborators": ["@handle"],
  "first_comment": "#hashtag1 #hashtag2"
}

// YouTube Video
{
  "privacy_status": "public",
  "category_id": 22,
  "tags": ["ai", "productivity"],
  "playlist_ids": ["PLxxx"],
  "thumbnail_media_id": "uuid",
  "made_for_kids": false,
  "default_language": "en"
}

// YouTube Short
{
  "privacy_status": "public"
}

// Twitter Thread (per-part)
{
  "thread_position": 1,
  "thread_group_id": "uuid",
  "reply_to_tweet_id": null,
  "poll": { "options": ["A", "B"], "duration_minutes": 1440 }
}

// HubSpot Blog
{
  "blog_id": "...",
  "slug": "my-post",
  "meta_description": "...",
  "featured_image_media_id": "uuid",
  "content_group_id": "..."
}

// Email Newsletter (Mailchimp)
{
  "subject_line": "This week in HR tech",
  "preview_text": "3 trends you can't ignore",
  "subscriber_list_ids": ["list123"],
  "ab_test_variants": [
    { "subject": "Variant A subject", "weight": 50 },
    { "subject": "Variant B subject", "weight": 50 }
  ],
  "send_type": "broadcast",
  "template_id": "tmpl_xxx"
}
```

---

## 5. AI Content Ideation Pipeline

This is the **core differentiator** — the primary workflow, not a side feature.

### Flow

```
User inputs raw idea ("talk about how AI is changing hiring")
  → content_ideas row (status: new)
  → content-ideation-queue picks it up
  → Step 1: Internet research (Perplexity / Tavily / your research agent)
  → Step 2: Cross-reference with trending_topics for relevance
  → Step 3: Multi-format expansion (using org's niche + brand_voice as context):
      → Blog draft         → content_items (type: blog_article)
      → Social captions    → content_items (type: post) × N platforms
      → Video script       → content_items (type: reel/short/video) + video_scripts
      → Newsletter angle   → content_items (type: newsletter)
  → Step 4: status = drafts_ready
  → User reviews generated drafts in dashboard, edits, approves
```

### `content_ideas`

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| org_id | UUID FK → organizations | |
| created_by | UUID FK → users | |
| raw_input | TEXT | The seed idea — "talk about how AI is changing hiring" |
| niche_context | VARCHAR NULL | Override org niche for this idea |
| research_status | ENUM('pending','researching','complete','failed') | |
| research_output | JSONB | AI research results: sources, key points, angles, statistics |
| generated_content_ids | UUID[] | FK refs to content_items spawned from this idea |
| target_formats | TEXT[] | `['blog', 'ig_reel', 'twitter_thread', 'newsletter']` — which formats to generate |
| status | ENUM('new','researching','drafts_ready','used','discarded') | |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

**`research_output` JSONB structure:**

```jsonc
{
  "summary": "AI is reshaping hiring through...",
  "key_points": [
    "Resume screening AI reduces time-to-hire by 40%",
    "Bias concerns driving demand for explainable AI"
  ],
  "sources": [
    { "title": "McKinsey Report 2025", "url": "...", "snippet": "..." }
  ],
  "suggested_angles": [
    "Hot take: AI hiring tools are more biased than humans",
    "Tutorial: How to implement AI screening in your ATS"
  ],
  "related_trending_topic_ids": ["uuid1", "uuid2"],
  "statistics": [
    { "stat": "67% of recruiters use AI tools", "source": "LinkedIn 2025 Survey" }
  ]
}
```

---

## 6. Video Scripting & B-Roll

### `video_scripts`

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| content_id | UUID FK → content_items | |
| idea_id | UUID FK → content_ideas NULL | |
| format | ENUM('short_form','long_form') | |
| hook | TEXT | First 3 seconds / opening line — critical for short-form |
| script_body | JSONB | Structured script with timestamps (see below) |
| b_roll_suggestions | JSONB | B-roll ideas per segment |
| thumbnail_ideas | JSONB | Thumbnail concepts |
| cta | TEXT | Call to action |
| estimated_duration_seconds | INT | |
| status | ENUM('draft','reviewed','final') | |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

**`script_body` JSONB structure:**

```jsonc
[
  {
    "timestamp": "0:00-0:03",
    "type": "hook",
    "speaker_text": "Did you know 67% of recruiters are already using AI?",
    "visual_direction": "Speaker direct to camera, bold text overlay with stat"
  },
  {
    "timestamp": "0:03-0:15",
    "type": "context",
    "speaker_text": "Here's why that matters for your job search...",
    "visual_direction": "Cut to screen recording of AI tool"
  },
  {
    "timestamp": "0:15-0:45",
    "type": "main_point",
    "speaker_text": "The three things AI looks for in your resume...",
    "visual_direction": "Split screen: resume on left, AI analysis on right"
  },
  {
    "timestamp": "0:45-0:55",
    "type": "cta",
    "speaker_text": "Follow for more hiring tips. Link in bio for the full guide.",
    "visual_direction": "Speaker direct to camera, point-up gesture"
  }
]
```

**`b_roll_suggestions` JSONB structure:**

```jsonc
[
  {
    "timestamp": "0:03-0:15",
    "description": "Screen recording of AI resume screening tool",
    "source_suggestion": "screen_capture",
    "stock_search_terms": ["AI technology", "resume screening software"]
  },
  {
    "timestamp": "0:15-0:45",
    "description": "Aerial office shot or stock footage of people working",
    "source_suggestion": "stock",
    "stock_search_terms": ["modern office aerial", "people working laptop"],
    "recommended_sources": ["pexels", "unsplash"]
  }
]
```

**`thumbnail_ideas` JSONB structure:**

```jsonc
[
  {
    "text_overlay": "AI IS REPLACING RECRUITERS?",
    "visual_description": "Split face: half human, half robot, shocked expression",
    "style": "bold_text_face",
    "color_scheme": ["#FF0000", "#FFFFFF", "#000000"]
  },
  {
    "text_overlay": "67% of recruiters use AI",
    "visual_description": "Clean stat with recruiter silhouette background",
    "style": "stat_highlight",
    "color_scheme": ["#1E90FF", "#FFFFFF"]
  }
]
```

---

## 7. Scheduling & Platform Rules Engine

### `platform_posting_rules`

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| platform | ENUM(...) | |
| content_type | VARCHAR | `IG_REEL`, `TW_TWEET`, `YT_VIDEO`, etc. |
| rule_type | ENUM('best_time','frequency_cap','character_limit','aspect_ratio','hashtag_limit','duration_limit','file_size_limit') | |
| rule_value | JSONB | See below |
| source | ENUM('system_default','ai_learned','user_override') | |
| org_id | UUID FK NULL | NULL = global default; non-NULL = org-specific override |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

**`rule_value` JSONB examples:**

```jsonc
// best_time — when to post for max engagement
{
  "day_of_week": [1, 2, 3, 4, 5],
  "optimal_hours_utc": [14, 15, 16],
  "timezone_adjust": true,
  "reasoning": "B2B audience most active during work hours"
}

// frequency_cap — don't over-post
{
  "max_posts_per_day": 2,
  "min_hours_between": 4,
  "reasoning": "Instagram algorithm penalizes rapid-fire posting"
}

// character_limit — platform constraints
{
  "max_characters": 2200,
  "recommended_characters": 150,
  "reasoning": "IG truncates at 125 chars in feed; full caption requires tap"
}

// aspect_ratio — media format constraints
{
  "required": ["9:16"],
  "recommended": ["9:16"],
  "reasoning": "Reels must be vertical"
}

// hashtag_limit
{
  "max_hashtags": 30,
  "recommended_hashtags": 5,
  "reasoning": "Instagram allows 30 but 3-5 targeted hashtags perform better per Hootsuite 2025 data"
}

// duration_limit — video length constraints
{
  "min_seconds": 3,
  "max_seconds": 90,
  "recommended_seconds": 30,
  "reasoning": "Reels max 90s, but 15-30s performs best for retention"
}

// file_size_limit
{
  "max_bytes": 4294967296,
  "reasoning": "YouTube max upload 256GB but practical limit for most users"
}
```

### How the Scheduler Uses Rules

When a user picks a time slot:

1. Query `platform_posting_rules` for the target platform + content type
2. Check `frequency_cap`: has the account already posted today? How recently?
3. Check `best_time`: is the chosen time in the optimal window? If not, suggest alternatives
4. Validate media against `aspect_ratio`, `duration_limit`, `file_size_limit`
5. Validate text against `character_limit`, `hashtag_limit`
6. Warn on violations; block on hard constraints (file size, aspect ratio)

---

## 8. Content Lifecycle & Approval Workflow

### State Machine

```
idea → draft → pending_approval → approved → scheduled → publishing → published
                    ↓                                         ↓
                rejected (→ draft)                    partially_published
                                                              ↓
                                                        failed (retry or manual fix)

Any state → archived (soft delete)
```

### `content_approval_steps`

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| content_id | UUID FK → content_items | |
| step_order | SMALLINT | 1, 2, 3... for multi-step approvals |
| approver_id | UUID FK → users NULL | Specific user, or NULL for role-based |
| approver_role | ENUM('admin','editor') NULL | "Any admin can approve" |
| decision | ENUM('pending','approved','rejected') | |
| comment | TEXT NULL | |
| decided_at | TIMESTAMPTZ NULL | |

### `content_versions` — Draft history

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| content_id | UUID FK → content_items | |
| version_number | INT | |
| snapshot | JSONB | Full serialized state: body_text, media refs, platform_specific_body, etc. |
| created_by | UUID FK → users | |
| change_summary | TEXT NULL | "Updated caption for IG, added 3rd carousel slide" |
| created_at | TIMESTAMPTZ | |

---

## 9. Trend Analysis & Niche Research

### `trending_topics`

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| platform | ENUM(...) | |
| region | VARCHAR(5) | `US`, `IN`, `GB`, etc. |
| category | VARCHAR NULL | |
| topic | VARCHAR | |
| volume_score | DECIMAL(5,4) | Normalized 0-1 |
| hashtags | TEXT[] | Associated hashtags |
| sample_content_urls | TEXT[] | Example viral posts |
| captured_at | TIMESTAMPTZ | |
| expires_at | TIMESTAMPTZ | TTL — trends are ephemeral |

### `ai_suggestions`

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| org_id | UUID FK → organizations | |
| suggestion_type | ENUM('content_idea','best_time','hashtag_set','trend_hook','repurpose','niche_insight') | |
| payload | JSONB | The suggestion content (varies by type) |
| source_trending_ids | UUID[] | Which trends informed this |
| relevance_score | DECIMAL(5,4) | How relevant to org's niche (0-1) |
| status | ENUM('new','viewed','accepted','dismissed') | |
| created_at | TIMESTAMPTZ | |

**Suggestion `payload` examples:**

```jsonc
// content_idea
{
  "headline": "Why AI hiring tools might be more biased than humans",
  "angle": "Contrarian take backed by MIT study data",
  "formats": ["ig_carousel", "twitter_thread", "blog"],
  "estimated_engagement": "high"
}

// trend_hook
{
  "trending_topic": "AI regulation EU",
  "hook": "Connect to your HR tech niche: what EU AI Act means for hiring tools",
  "urgency": "trend peaks in 48h"
}

// repurpose
{
  "source_content_id": "uuid",
  "suggestion": "Your blog post on AI screening got 4x avg engagement. Repurpose as a 30s IG Reel with the top 3 stats.",
  "target_format": "ig_reel"
}
```

---

## 10. Analytics Data Model

Analytics data is **time-series** and **high-volume**. Separate storage strategy from transactional tables.

### `analytics_snapshots` (append-only, partitioned by time)

| Column | Type | Notes |
|--------|------|-------|
| id | BIGSERIAL PK | |
| platform_post_id | UUID FK → content_platform_posts | |
| captured_at | TIMESTAMPTZ | Partition key |
| metrics | JSONB | Platform-native metrics blob (see below) |
| followers_at_time | INT | Account followers when captured — needed for engagement rate |

**`metrics` JSONB per platform:**

```jsonc
// Instagram
{ "likes": 1200, "comments": 84, "shares": 31, "saves": 210,
  "reach": 45000, "impressions": 52000, "profile_visits": 320 }

// YouTube
{ "views": 85000, "likes": 3200, "dislikes": 45, "comments": 410,
  "watch_time_minutes": 12500, "avg_view_duration_seconds": 142,
  "click_through_rate": 0.078, "impressions": 1100000 }

// Twitter/X
{ "likes": 540, "retweets": 120, "quote_tweets": 18, "replies": 67,
  "impressions": 89000, "profile_clicks": 230, "url_clicks": 1800 }

// Email Newsletter
{ "sent": 5000, "delivered": 4850, "opens": 1940, "unique_opens": 1720,
  "clicks": 380, "unsubscribes": 12, "bounces": 150,
  "open_rate": 0.3546, "click_rate": 0.0784 }
```

> **Storage**: Use TimescaleDB (Postgres extension) or partition by month. Continuous aggregates for dashboard queries.

### `analytics_aggregates` (materialized rollups)

| Column | Type | Notes |
|--------|------|-------|
| platform_post_id | UUID FK | |
| period | ENUM('daily','weekly','monthly') | |
| period_start | DATE | |
| metrics_sum | JSONB | Summed deltas |
| engagement_rate | DECIMAL(8,6) | Computed: interactions / reach |

### `account_analytics_daily` (account-level growth tracking)

| Column | Type | Notes |
|--------|------|-------|
| connected_account_id | UUID FK → connected_accounts | |
| date | DATE | |
| followers_count | INT | |
| followers_delta | INT | +/- since yesterday |
| posts_published | INT | |
| total_reach | BIGINT | |
| total_engagement | BIGINT | |

---

## 11. Queues, Jobs & Async Pipelines

### Queue Architecture

Using BullMQ on Redis (hot path) + DB job table (scheduled jobs source of truth).

```
┌──────────────────────────────────────────────────────────────────────┐
│                           JOB QUEUES                                 │
│                                                                      │
│  ┌───────────────────┐  ┌────────────────────┐  ┌────────────────┐  │
│  │  publish-queue     │  │  media-processing   │  │ analytics-sync │  │
│  │                    │  │                     │  │                │  │
│  │  Scheduled posts   │  │  Transcode/resize   │  │  Pull metrics  │  │
│  │  Fan-out to        │  │  Generate per-plat   │  │  per platform  │  │
│  │  platform APIs     │  │  variants (9:16,     │  │  Aggregate     │  │
│  │  Retry on failure  │  │  16:9, 1:1)         │  │  into rollups  │  │
│  └───────────────────┘  │  Upload to CDN       │  └────────────────┘  │
│                          └────────────────────┘                       │
│  ┌───────────────────┐  ┌────────────────────┐  ┌────────────────┐  │
│  │  token-refresh     │  │  trend-analysis     │  │ webhook-ingest │  │
│  │                    │  │                     │  │                │  │
│  │  Proactive OAuth   │  │  Fetch trending     │  │  Platform event│  │
│  │  token renewal     │  │  per platform/region │  │  callbacks     │  │
│  │  Mark reauth       │  │  Run AI analysis    │  │  (comments,    │  │
│  │  on failure        │  │  Generate suggest.  │  │  mentions, DMs)│  │
│  └───────────────────┘  └────────────────────┘  └────────────────┘  │
│                                                                      │
│  ┌───────────────────┐  ┌────────────────────┐  ┌────────────────┐  │
│  │  content-ideation  │  │  content-delete     │  │  audit-log     │  │
│  │                    │  │                     │  │                │  │
│  │  AI research →     │  │  Platform delete    │  │  Async audit   │  │
│  │  Multi-format      │  │  + local cleanup    │  │  trail writes  │  │
│  │  expansion         │  │  + CDN purge        │  │  (non-blocking)│  │
│  └───────────────────┘  └────────────────────┘  └────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

### `scheduled_jobs` (DB-backed source of truth)

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| queue_name | VARCHAR | |
| payload | JSONB | |
| status | ENUM('pending','locked','running','completed','failed','dead') | |
| run_at | TIMESTAMPTZ | For delayed/scheduled jobs |
| locked_by | VARCHAR NULL | Worker ID |
| locked_at | TIMESTAMPTZ NULL | |
| attempts | SMALLINT DEFAULT 0 | |
| max_attempts | SMALLINT DEFAULT 3 | |
| last_error | TEXT NULL | |
| completed_at | TIMESTAMPTZ NULL | |
| created_at | TIMESTAMPTZ | |

### Key Pipeline Flows

**Flow 1: Content Creation → Multi-Platform Publish**

```
User creates content in dashboard
  → content_items row (status: draft)
  → User attaches media → content_media rows (status: uploaded)
  → media-processing queue:
      → Transcode video (ffmpeg)
      → Generate per-platform variants (9:16, 16:9, 1:1)
      → Upload variants to CDN
      → content_media.processing_status = ready
  → User selects target platforms
      → content_platform_posts rows created (one per platform)
      → Validate against platform_posting_rules (warn/block)
  → User submits for approval (if org requires it)
      → content_approval_steps created
      → Notify approver (email / slack webhook / in-app)
  → Approved → status = approved
  → If scheduled_at is set → publish-queue delayed job
  → If immediate → publish-queue immediate job
  → publish-queue worker:
      → For each content_platform_posts row:
          → Check idempotency_key against platform (prevent double-post)
          → Call platform adapter (existing connector)
          → On success: fill platform_post_id, published_url, status = published
          → On failure: status = failed, error_message, retry_count++
          → If retry_count < max → re-enqueue with exponential backoff
      → Update parent content_items.status:
          → All published → "published"
          → Some published → "partially_published"
          → All failed → "failed"
```

**Flow 2: Idea → AI Research → Multi-Format Drafts**

```
User enters idea text
  → content_ideas row (status: new)
  → content-ideation queue:
      → Research phase:
          → Internet search (Perplexity/Tavily) using org niche + idea
          → Cross-reference trending_topics
          → Aggregate stats, sources, angles
          → Save to content_ideas.research_output
      → Generation phase (per target_format):
          → Blog: Generate full article draft using research + brand_voice
          → Social: Generate platform-optimized captions
          → Video: Generate script + B-roll suggestions → video_scripts table
          → Newsletter: Generate subject line + body
          → Each → content_items row (status: draft, idea_id set)
      → content_ideas.status = drafts_ready
      → Notify user: "3 drafts ready for review"
```

**Flow 3: Analytics Sync**

```
Cron: every 1 hour (configurable per plan tier)
  → For each active connected_account:
      → Enqueue analytics-sync job
  → Worker:
      → Call platform analytics API
      → For recent posts (last 30 days):
          → Upsert analytics_snapshots
      → For account-level metrics:
          → Upsert account_analytics_daily
      → Nightly: run aggregation → analytics_aggregates
```

**Flow 4: Trend Analysis → AI Suggestions**

```
Cron: every 6 hours
  → For each platform + region combo:
      → Fetch trending data via platform API / scraping adapters
      → Upsert trending_topics (with TTL via expires_at)
  → For each org with AI features enabled:
      → Gather: org's content_pillars + niche + recent performance + trending_topics
      → Call AI service
      → Insert ai_suggestions with relevance_score
      → Notify user in dashboard if high-relevance suggestion
```

---

## 12. Rule Engine Orchestrator Design

### Core Concept

The Rule Engine is a **singleton orchestrator** that maintains a registry of **strategies**. Each strategy implements a lifecycle interface with explicit phases. The engine routes content events to the correct strategy, manages state persistence, and handles two-phase commit for reliable execution.

### Why a Rule Engine?

Content management has many conditional workflows:
- "If a post gets > 1000 likes in 24h, suggest repurposing it as a blog post"
- "If posting frequency drops below 3x/week, send a nudge notification"
- "Before publishing, validate all platform rules are met"
- "When a trend matches our niche with score > 0.7, generate a content idea"
- "If a scheduled post's media processing fails, re-route to manual review"

Rather than hardcoding these as if-else chains scattered across services, a rule engine provides a **structured, extensible** way to add new behaviors.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     ContentRuleEngine                           │
│                     (Singleton Orchestrator)                    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Strategy Registry (Map)                     │   │
│  │                                                          │   │
│  │  "post_performance_repurpose" → PostPerformanceStrategy  │   │
│  │  "platform_rule_validation"   → PlatformValidationStrat  │   │
│  │  "trend_match_ideation"       → TrendMatchStrategy       │   │
│  │  "frequency_nudge"            → FrequencyNudgeStrategy   │   │
│  │  "best_time_suggestion"       → BestTimeSuggestionStrat  │   │
│  │  "content_expiry_archival"    → ExpiryArchivalStrategy   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  evaluate(event) ─┬→ getParticipatingStrategies(event)         │
│                   ├→ For each strategy:                         │
│                   │    Phase 0: Check pending trigger (recovery)│
│                   │    Phase 1: prepare(context, state)         │
│                   │    Phase 2: updateState(prepared, state)    │
│                   │    Phase 3: Check shouldTrigger             │
│                   │    Phase 4: execute(prepared, updatedState) │
│                   │    Phase 5: Persist state (two-phase)       │
│                   └→ Return { actions, triggeredStrategies }    │
└─────────────────────────────────────────────────────────────────┘
```

### Strategy Interface

```typescript
/**
 * ContentStrategy Interface — Explicit Lifecycle Phases
 *
 * 
 * Solves the same dual heterogeneity problem:
 *   - INPUT HETEROGENEITY: Different strategies need different data
 *     (analytics API, trending data, content metadata, user behavior logs)
 *   - OUTPUT HETEROGENEITY: Different actions result from triggers
 *     (create suggestion, send notification, auto-archive, validate rules)
 */
interface ContentStrategy<TPrepared = unknown, TState extends Record<string, unknown> = Record<string, unknown>> {
    readonly strategyType: string;

    // ============= LIFECYCLE METHODS =============

    /**
     * Fetch and prepare data needed for decision-making.
     * Handles INPUT HETEROGENEITY — each strategy queries its own sources.
     *
     * Examples:
     *   - PostPerformanceStrategy: fetches analytics_snapshots for the post
     *   - TrendMatchStrategy: fetches trending_topics + org content_pillars
     *   - FrequencyNudgeStrategy: counts recent content_items for the org
     */
    prepare(context: ContentEventContext, currentState: TState): Promise<TPrepared>;

    /**
     * Update state based on the event. Pure function.
     * Returns { state, changed } — skip DB write if unchanged (idempotency).
     *
     * The updated state includes a `shouldTrigger` flag, keeping trigger
     * logic co-located with state update logic.
     */
    updateState(prepared: TPrepared, currentState: TState): {
        state: TState;
        changed: boolean;
    };

    /**
     * Execute the strategy's action. Async — can call AI, send notifications, etc.
     * Handles OUTPUT HETEROGENEITY.
     *
     * Examples:
     *   - PostPerformanceStrategy: creates ai_suggestions row + notification
     *   - PlatformValidationStrategy: returns validation errors array
     *   - TrendMatchStrategy: enqueues content-ideation job
     */
    execute(prepared: TPrepared, updatedState: TState, context: ExecuteContext): Promise<ExecuteResult>;

    /** Default initial state when no persisted state exists */
    getDefaultState(): TState;

    // ============= TWO-PHASE COMMIT =============
    hasPendingTrigger(state: TState): boolean;
    getPendingAction(state: TState): ContentAction | undefined;
    createPendingState(state: TState, eventId: string, action: ContentAction): TState;
    clearPendingTrigger(state: TState): TState;

    // ============= BYPASS =============
    /** Strategy-specific skip logic (e.g., already completed, wrong event type) */
    shouldBypass(context: BypassContext): boolean;
}
```

### Event Types

```typescript
/**
 * Events that flow through the rule engine.
 * Any system event can be routed here for strategy evaluation.
 */
type ContentEventType =
    | 'content.published'      // After successful publish
    | 'content.failed'         // After publish failure
    | 'content.scheduled'      // When content is scheduled
    | 'analytics.updated'      // After analytics sync
    | 'trend.detected'         // After trend analysis finds match
    | 'frequency.check'        // Periodic posting frequency audit
    | 'pre_publish.validate'   // Before publish — validation gate
    | 'content.expiry_check';  // Periodic check for stale content

interface ContentEventContext {
    eventType: ContentEventType;
    orgId: string;
    userId: string;
    /** The entity that triggered this event */
    entityType: 'content_item' | 'content_platform_post' | 'connected_account' | 'trending_topic';
    entityId: string;
    /** Event-specific payload */
    payload: Record<string, unknown>;
    /** Timestamp of the event */
    timestamp: Date;
}
```

### State Persistence

```typescript
/**
 * Rule state table — stores per-org, per-strategy state.
 * 
 */
// Table: content_rule_states
// | Column              | Type          | Notes                                      |
// |---------------------|---------------|--------------------------------------------|
// | id                  | SERIAL PK     |                                            |
// | org_id              | UUID FK       | Scoped to organization                     |
// | strategy_id         | VARCHAR       | e.g., "post_performance_repurpose"         |
// | strategy_type       | VARCHAR       | Strategy class identifier                  |
// | scope_type          | VARCHAR       | "org" | "account" | "content_item"         |
// | scope_id            | VARCHAR       | The scoped entity ID                       |
// | state               | JSONB         | Strategy-specific persisted state           |
// | created_at          | TIMESTAMPTZ   |                                            |
// | updated_at          | TIMESTAMPTZ   |                                            |
```

### Example Strategy: Post Performance → Repurpose Suggestion

```typescript
/**
 * PostPerformanceRepurposeStrategy
 *
 * Trigger: When a post's engagement rate exceeds 2x the account average,
 *          suggest repurposing it into other formats.
 *
 * Scope: per content_platform_post
 */

// State shape
interface PostPerfState {
    lastCheckedAt: string | null;
    accountAvgEngagement: number;
    postEngagement: number;
    triggered: boolean;
    shouldTrigger: boolean;
}

// Prepared data shape
interface PostPerfPrepared {
    currentMetrics: { likes: number; comments: number; shares: number; reach: number };
    accountAvgEngagementRate: number;
    contentItem: { id: string; content_type: string; body_text: string };
}

class PostPerformanceRepurposeStrategy implements ContentStrategy<PostPerfPrepared, PostPerfState> {
    readonly strategyType = 'post_performance_repurpose';

    async prepare(context, currentState) {
        // Fetch latest analytics_snapshots for this post
        // Fetch account_analytics_daily for avg engagement rate
        // Fetch content_item metadata for repurpose context
    }

    updateState(prepared, currentState) {
        const engagementRate = (prepared.currentMetrics.likes + prepared.currentMetrics.comments + prepared.currentMetrics.shares) / prepared.currentMetrics.reach;
        const threshold = prepared.accountAvgEngagementRate * 2;

        return {
            state: {
                ...currentState,
                lastCheckedAt: new Date().toISOString(),
                accountAvgEngagement: prepared.accountAvgEngagementRate,
                postEngagement: engagementRate,
                shouldTrigger: engagementRate > threshold && !currentState.triggered,
            },
            changed: true,
        };
    }

    async execute(prepared, updatedState, context) {
        // Insert ai_suggestions row with type: 'repurpose'
        // Send notification to content creators in the org
        // Return action summary
    }

    getDefaultState(): PostPerfState {
        return {
            lastCheckedAt: null,
            accountAvgEngagement: 0,
            postEngagement: 0,
            triggered: false,
            shouldTrigger: false,
        };
    }

    // ... two-phase commit methods ...
}
```

### When Strategies Run

| Strategy | Triggered By | Cron? | Event-Driven? |
|----------|-------------|-------|---------------|
| `post_performance_repurpose` | `analytics.updated` | — | Yes (after analytics sync) |
| `platform_rule_validation` | `pre_publish.validate` | — | Yes (before publish) |
| `trend_match_ideation` | `trend.detected` | — | Yes (after trend analysis) |
| `frequency_nudge` | `frequency.check` | Every 24h | — |
| `best_time_suggestion` | `content.scheduled` | — | Yes (when scheduling) |
| `content_expiry_archival` | `content.expiry_check` | Every 24h | — |

---

## 13. Supporting Tables

### `campaigns`

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| org_id | UUID FK | |
| name | VARCHAR | "Q1 Product Launch" |
| start_date | DATE | |
| end_date | DATE | |
| status | ENUM('draft','active','completed') | |
| goals | JSONB | `{ "target_reach": 1000000, "target_engagement_rate": 0.05 }` |
| created_at | TIMESTAMPTZ | |

### `content_labels` (tags for organizing)

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| org_id | UUID FK | |
| name | VARCHAR | |
| color | VARCHAR(7) | Hex color for UI |

### `content_item_labels` (many-to-many)

| Column | Type |
|--------|------|
| content_id | UUID FK → content_items |
| label_id | UUID FK → content_labels |

### `audit_log` (immutable, append-only)

| Column | Type | Notes |
|--------|------|-------|
| id | BIGSERIAL | |
| org_id | UUID | |
| user_id | UUID | |
| action | VARCHAR | `content.created`, `content.published`, `account.connected`, `idea.generated` |
| entity_type | VARCHAR | `content_item`, `connected_account`, `content_idea` |
| entity_id | UUID | |
| changes | JSONB NULL | Before/after diff |
| ip_address | INET | |
| created_at | TIMESTAMPTZ | |

### `notifications`

| Column | Type | Notes |
|--------|------|-------|
| id | UUID PK | |
| org_id | UUID FK | |
| user_id | UUID FK | Target user |
| type | ENUM('approval_needed','post_published','post_failed','reauth_required','suggestion','mention','frequency_nudge','drafts_ready') | |
| payload | JSONB | |
| read | BOOLEAN DEFAULT false | |
| created_at | TIMESTAMPTZ | |

### `content_rule_states` (rule engine state persistence)

| Column | Type | Notes |
|--------|------|-------|
| id | SERIAL PK | |
| org_id | UUID FK | |
| strategy_id | VARCHAR | |
| strategy_type | VARCHAR | |
| scope_type | VARCHAR | `org`, `account`, `content_item` |
| scope_id | VARCHAR | |
| state | JSONB | Strategy-specific state |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

---

## 14. High-Level System Diagram

```
                                        ┌─────────────────┐
                                        │   CDN / S3      │
                                        │  (media assets) │
                                        └────────▲────────┘
                                                 │
┌──────────┐     ┌──────────────────┐    ┌───────┴────────┐     ┌──────────────────┐
│ Dashboard │────▶│   API Gateway /  │───▶│   App Service  │────▶│   PostgreSQL     │
│   (SPA)  │◀────│  Auth (JWT+RBAC) │◀───│  (Node/Python) │◀────│   (Primary DB)   │
└──────────┘     └──────────────────┘    └───────┬────────┘     │                  │
                                                 │               │ + TimescaleDB    │
                                                 │               │   (analytics)    │
                                        ┌────────▼────────┐     └──────────────────┘
                                        │     Redis        │
                                        │ (cache + BullMQ) │
                                        └────────┬────────┘
                                                 │
                          ┌──────────────────────┼──────────────────────┐
                          │                      │                      │
                  ┌───────▼──────┐     ┌─────────▼────────┐   ┌───────▼──────┐
                  │   Publish    │     │   Media          │   │  Analytics   │
                  │   Workers    │     │   Workers        │   │  Workers     │
                  └───────┬──────┘     └─────────┬────────┘   └───────┬──────┘
                          │                      │                    │
                  ┌───────▼──────┐     ┌─────────▼────────┐          │
                  │  Ideation    │     │   Trend          │          │
                  │  Workers     │     │   Workers        │          │
                  │  (AI calls)  │     │   (AI + APIs)    │          │
                  └───────┬──────┘     └─────────┬────────┘          │
                          │                      │                    │
                  ┌───────▼──────────────────────▼────────────────────▼───┐
                  │          Platform Adapters (Existing Connectors)       │
                  │   IG │ TikTok │ YT │ X │ Threads │ LI │ HubSpot │ MC │
                  └──────────────────────────────────────────────────────┘
                                                 │
                                        ┌────────▼────────┐
                                        │  Rule Engine     │
                                        │  (Strategies     │
                                        │   evaluate on    │
                                        │   every event)   │
                                        └─────────────────┘
```

---

## 15. Indexes

```sql
-- Tenant isolation (every query scopes by org)
CREATE INDEX idx_content_items_org_status ON content_items(org_id, status);
CREATE INDEX idx_content_items_org_scheduled ON content_items(org_id, scheduled_at)
    WHERE status = 'scheduled';

-- Content calendar view (most-used dashboard feature)
CREATE INDEX idx_content_items_org_date ON content_items(org_id, scheduled_at, published_at);

-- Publish queue lookup
CREATE INDEX idx_platform_posts_status ON content_platform_posts(status)
    WHERE status IN ('pending', 'publishing', 'failed');

-- Idempotency check during publish
CREATE UNIQUE INDEX idx_platform_posts_idempotency ON content_platform_posts(idempotency_key);

-- Analytics queries (time-range scans)
CREATE INDEX idx_analytics_snapshots_post_time
    ON analytics_snapshots(platform_post_id, captured_at DESC);

-- Token refresh job
CREATE INDEX idx_connected_accounts_expiry ON connected_accounts(token_expires_at)
    WHERE status = 'active';

-- Audit log (queried by entity)
CREATE INDEX idx_audit_log_entity ON audit_log(org_id, entity_type, entity_id, created_at DESC);

-- Trend TTL cleanup
CREATE INDEX idx_trending_topics_expiry ON trending_topics(expires_at)
    WHERE expires_at IS NOT NULL;

-- AI suggestions for dashboard
CREATE INDEX idx_ai_suggestions_org_status ON ai_suggestions(org_id, status, created_at DESC);

-- Content ideas for dashboard
CREATE INDEX idx_content_ideas_org_status ON content_ideas(org_id, status);

-- Rule engine state lookup
CREATE UNIQUE INDEX idx_rule_states_lookup
    ON content_rule_states(org_id, strategy_id, scope_type, scope_id);
```

---

## 16. Priority Phasing (v1 → v2 → v3)

### v1 — MVP (Target: $750 budget scope)

**Tables**: organizations, users (simple role), connected_accounts, content_items, content_media, content_platform_posts, content_ideas, video_scripts, scheduled_jobs

**Features**:
- AI ideation pipeline (the differentiator — idea → research → multi-format drafts)
- Basic scheduling + publish to 2-3 platforms (Instagram, Twitter, one blog)
- Static best-practice rules (hardcoded, not rules engine)
- Simple analytics pull (daily cron, no aggregation)
- Simple role model (owner/editor — no scoped RBAC)

**Skip**: Approval workflows, rule engine, TimescaleDB, newsletters, audit log, webhook receivers, media transcoding (validate format client-side)

### v2 — Fast Follow

**Add**: trending_topics, ai_suggestions, platform_posting_rules, video_scripts (full), content_versions, notifications

**Features**:
- Trend analysis + niche-aware suggestions
- Video scripting with B-roll suggestions
- Scheduling rules engine (validate before publish)
- Email newsletter support (Mailchimp/SendGrid adapter)
- Content versioning and draft history
- YouTube + TikTok adapters

### v3 — Enterprise

**Add**: content_approval_steps, audit_log, content_rule_states, analytics_aggregates, account_analytics_daily, campaigns, content_labels

**Features**:
- Full approval workflows (multi-step)
- Full RBAC with scoped permissions
- Audit log
- Rule engine (event-driven strategies)
- Advanced analytics with rollups + reporting
- Webhook-driven real-time updates
- Campaign management
- Multi-region / data residency

---

## 17. Open Questions & Future Considerations

1. **Rate limiting per platform**: Each social API has rate limits (IG: 200 calls/hr, Twitter: tiered). Publish workers need per-account rate limiting — sliding window counter in Redis per `connected_account_id + platform`.

2. **Idempotency for publish**: If a publish worker crashes after calling IG API but before marking `status=published`, the retry must not double-post. The `idempotency_key` on `content_platform_posts` + platform-side duplicate detection addresses this.

3. **Media storage costs**: Video at scale is expensive. Tiered storage (hot → warm → archive) tracked via `content_media.storage_tier`. Move to S3 Glacier after 30 days of inactivity.

4. **Webhook receivers**: Platforms push events (new comments, mentions, deauth). Need `/webhooks/:platform` endpoint → signature validation → `webhook-ingest` queue → async processing.

5. **Multi-region / data residency**: EU enterprise customers may require data stays in EU. DB + media storage strategy should account for this.

6. **Content calendar**: The calendar view is likely the most-used feature. Query on `content_items(org_id, scheduled_at)` with the right index should suffice over a separate `calendar_slots` table.

7. **Competitive inspiration**: Hootsuite, Buffer, Sprout Social, Later, Planable (for approval workflows), Typefully (for Twitter threads), Jasper (for AI content generation). The differentiator here is the **AI ideation → multi-format generation pipeline** being first-class, not bolted on.

8. **AI cost management**: AI ideation calls (research + multi-format generation) are expensive. Gate by plan_tier. Track usage in a `ai_usage_log` table. Consider caching research results for similar ideas within same org.
