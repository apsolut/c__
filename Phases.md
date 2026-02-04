# Apsolut CI - Implementation Phases

This document outlines the phased roadmap for building **Apsolut CI**, a scalable competitive intelligence platform. The architecture mimics the "C__" philosophy—high performance, unified data flow, and modern AI integration—using best-in-class tools available in 2026.

## Phase 1: Core Data Foundation (The "Structs")

**Goal:** Establish a strictly typed, scalable data layer that supports both relational data (competitors) and semantic data (AI embeddings).

*   **Infrastructure:**
    *   Initialize **Supabase** project (PostgreSQL).
    *   Enable extensions: `pgvector` (for semantic search) and `pg_cron` (for scheduling tasks).
*   **Schema Design:**
    *   `organizations` (Tenants): `id`, `name`, `plan_tier`.
    *   `competitors`: `id`, `org_id`, `name`, `domain`, `sitemap_url`.
    *   `intel_items`: `id`, `competitor_id`, `source_url`, `content_hash` (deduplication), `summary`, `embedding` (vector).
    *   `battle_cards`: `id`, `competitor_id`, `category` (Pricing, Strengths), `content_markdown`.
*   **Security:**
    *   Configure **Supabase Auth** for user login/signup.
    *   Implement **Row Level Security (RLS)** policies immediately to ensure tenants can only see their own competitors.

## Phase 2: The "Intel Engine" (Data Acquisition)

**Goal:** Build the autonomous backend that discovers, scrapes, and processes competitor data without manual input.

*   **Orchestration (n8n / Temporal):**
    *   Deploy **n8n** (self-hosted on Coolify/Hetzner) to manage workflows.
    *   Create `Workflow: Daily Discovery` – runs every 24h for each active competitor.
*   **Crawling & Scraping:**
    *   **Service:** Deploy a **Firecrawl** or **Browserbase** instance (or use a Python `Scrapy` container) to handle JavaScript-heavy sites.
    *   **Logic:**
        1.  Fetch `sitemap.xml` or crawl homepage links.
        2.  Filter out irrelevant pages (Login, TOS).
        3.  Compare strictly against `last_crawled` hash to detect changes.
*   **Visual Intelligence:**
    *   Integrate a screenshot service.
    *   Send screenshots of key pages (Pricing, Home) to a multimodal model (GPT-4o/Gemini) to extract structured data (e.g., "Extract pricing tiers into JSON").

## Phase 3: The "C__" Dashboard (High-Performance UI)

**Goal:** A zero-lag interface that feels like a native app, minimizing "loading spinners" via Server Components.

*   **Framework:** **Next.js 15 (App Router)**.
*   **Data Fetching:**
    *   Use **React Server Components (RSC)** to fetch data directly from the DB on the server.
    *   Implement **Streaming** (`<Suspense>`) for slow data parts (like generating AI summaries).
*   **Visualization:**
    *   **Charts:** Use **Recharts** or **Tremor** for standard analytics (Activity over time).
    *   **Performance:** For large datasets (10k+ points), use a WebGPU-accelerated library (like **Deck.gl**) to keep frame rates high.
*   **Battle Card UI:**
    *   Build a "Live Card" component that pulls the latest `intel_items` for a specific category.

## Phase 4: AI Integration & RAG Layer

**Goal:** Connect the raw data to "Brains" that generate insights, battle cards, and answers.

*   **Unified AI Gateway:**
    *   Install **Vercel AI SDK**.
    *   Configure providers (OpenAI, Anthropic, DeepSeek) via a single config interface.
*   **RAG Pipeline (Retrieval Augmented Generation):**
    *   **Ingest:** When new intel is saved, generate vector embeddings and store in `intel_items`.
    *   **Query:** When a user asks "What is their pricing strategy?", perform a vector similarity search to find relevant snippets.
    *   **Generate:** Pass snippets + prompt to the LLM to write the answer.
*   **Agentic Workflows:**
    *   Build simple agents (e.g., "Competitor Auditor") that can actively browse a site to answer a specific question if the answer isn't in the database.

## Phase 5: Scale & Optimization

**Goal:** Prepare the system for 100k+ users and massive data volume.

*   **Queue System:**
    *   Implement **Redis** (via BullMQ) to decouple the UI from heavy scraping jobs.
    *   User clicks "Scan Now" -> Job added to Queue -> UI shows "Scanning..." -> Worker processes job -> UI updates via WebSocket/Polling.
*   **Database Scaling:**
    *   Set up **Read Replicas** in Supabase for dashboard read operations.
    *   Implement partitioning on the `intel_items` table (e.g., by year or tenant) if it grows beyond millions of rows.
*   **Caching:**
    *   Aggressive caching of "Battle Cards" and "Summaries" to reduce AI API costs and load times.
