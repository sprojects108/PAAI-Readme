# PAAI Autopilot

### AI-Powered Multi-Tenant Receptionist SaaS for Local Businesses

> **Live Demo:** [paai-ai.vercel.app](https://paai-ai.vercel.app)

PAAI Autopilot is a production-grade, multi-tenant SaaS platform that gives local businesses — salons, auto detailers, plumbers, and **100+ other industries** — their own AI-powered receptionist. Each business gets a personalized chat widget, voice AI, appointment booking, dynamic pricing, SMS confirmations, and a full management dashboard — all powered by a **triple AI engine** (OpenAI + Google Gemini + OpenRouter) with real-time function calling and **automatic runtime failover**.

---

## Why This Project Exists

Small businesses lose customers when they can't answer the phone. PAAI Autopilot solves this by replacing the traditional receptionist with an AI that:

- **Answers questions** about services, hours, and pricing in a warm, human tone
- **Shows real-time prices** pulled directly from the database (never hallucinated)
- **Checks live calendar availability** and presents open time slots
- **Books appointments** with lead capture, deduplication, and UTC-safe scheduling
- **Sends SMS confirmations** to customers via Twilio
- **Notifies the business owner** instantly via browser push notifications
- **Handles voice calls** through VAPI integration with shared knowledge context

Each business gets a unique URL (`/your-business-name`), a tailored AI personality based on their industry, and a dashboard to manage leads, pricing, and AI settings.

---

## Key Technical Highlights

| Area | What I Built |
|---|---|
| **Triple AI Engine with Runtime Failover** | Supports OpenAI (gpt-4o-mini), Google Gemini (1.5-flash), and OpenRouter (Gemini 2.0 Flash via `google/gemini-2.0-flash-001`). Provider-agnostic architecture — same system prompt, same 7 tools, same handlers. **Automatic runtime failover**: if the primary provider fails, the system retries with the configured fallback provider transparently. Zero customer-facing downtime. |
| **Hybrid RAG + Function Calling (8 Tools)** | Combines **retrieval-augmented generation** for knowledge questions (FAQs, policies, preparation instructions) with **deterministic function calling** for actions (booking, pricing, availability). The AI autonomously decides whether to search the vector knowledge base or call an action tool. RAG uses **pgvector** (cosine similarity) with **OpenAI embeddings** (`text-embedding-3-small`), scoped per-tenant for data isolation. Content is auto-chunked and embedded on upload via the dashboard. |
| **Anti-Hallucination System** | Multi-layer defense against price hallucination: data validation (`isValidPrice()`), tool response framing ("RESPOND WITH THIS EXACT TEXT"), system prompt enforcement ("FORBIDDEN: $XX, $TBD, $0"), and graceful degradation for missing prices (Soft Pivot / Expert Pivot). |
| **Dynamic Pricing Engine** | Merges Prisma services (with prices) + knowledge base entries (names only). Filters scraped UI clutter via a 40+ term blacklist. Groups unpriced services into natural sentences. Adds industry-specific upsell suggestions. |
| **Availability Engine** | 3-second timeout with fallback common slots (no error loops). Overlap detection against existing bookings. Morning/afternoon categorization. Slot caching in conversation context to prevent redundant tool calls. |
| **Multi-Tenant Isolation** | Per-tenant data scoping, system prompts, usage metering, API keys, Twilio numbers, and knowledge bases. 100+ industry configs with tailored prompts, emojis, and service suggestions. |
| **Streaming Responses** | Real-time "typing" effect using Web Streams API (`ReadableStream` + `TextEncoder`), character-by-character for both OpenAI and Gemini providers. |
| **Hybrid Database** | Supabase for auth and core tables + Prisma ORM for structured models (Service, Booking) — both hitting the same PostgreSQL instance via `@prisma/adapter-pg` connection pooling. |
| **Push Notifications** | Web Push (VAPID) with no third-party service — subscription management, payload delivery, Confirm/Reject actions, and automatic cleanup of expired endpoints. |
| **System Sentry (AI Health Agent)** | Autonomous monitoring agent at `/api/admin/system-sentry` that checks Prisma/DB connectivity, all 3 AI provider endpoints, and environment variable integrity — every 5 minutes via Vercel Cron. Uses the **Vercel AI SDK** (`generateText` + `tool`) to reason about failures, produce Technical Root Cause + Proposed Fix, and auto-trigger emergency push notifications to the admin. Gracefully degrades to rule-based analysis if the AI itself is down. |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  CLIENT LAYER                                                   │
│                                                                 │
│  Tenant Pages (/[slug])     → Chat Widget + Voice Widget        │
│  Dashboard (/dashboard)     → Leads, Pricing Manager, Settings  │
│  Admin Panel (/(admin))     → Tenant CRUD, Usage Tracking       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  API LAYER (Next.js API Routes — Serverless)                    │
│                                                                 │
│  /api/chat ─────────► AI Engine ──► 8 Function-Calling Tools    │
│  /api/vapi/chat      (OpenAI / Gemini / OpenRouter + Failover)  │
│  /api/knowledge      ──► RAG: Ingest, Embed, Search             │
│  /api/dashboard      ──► Pricing Manager CRUD                   │
│  /api/leads          ──► Lead Management + SMS                  │
│  /api/webhooks       ──► Stripe, Twilio, Voice                  │
│  /api/notifications  ──► Push Subscription                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  DATA LAYER                                                     │
│                                                                 │
│  Supabase  → Auth, tenants, leads, knowledge_base               │
│  Prisma    → Service (prices), Booking (calendar)               │
│  pgvector  → RAG embeddings (knowledge_chunks, cosine search)   │
│  External  → OpenAI, Gemini, Twilio, Stripe, VAPI, Web Push    │
└─────────────────────────────────────────────────────────────────┘
```

### AI Chat Flow (Hybrid RAG + Function Calling)

```
Customer message
    → /api/chat receives message + tenant_id
    → Build dynamic system prompt (persona + business context + pricing rules
      + availability rules + custom instructions + booking context)
    → Send to primary AI provider (OpenAI, Gemini, or OpenRouter)
    → If primary fails → automatic retry with fallback provider
    → AI decides which tool to call:
        ├── ACTION question (book, price, availability)
        │   → Call function tool (e.g., get_available_slots, book_appointment)
        │   → Server executes tool against database
        │   → Tool returns exact, deterministic data
        └── KNOWLEDGE question (FAQ, policy, preparation)
            → Call search_knowledge_base(query)
            → Query embedded into vector via OpenAI Embeddings
            → pgvector cosine similarity search (filtered by tenant_id)
            → Top 3 relevant chunks returned with similarity scores
    → Tool result sent back to AI
    → AI generates final response grounded in retrieved data
    → Response streamed character-by-character to client
```

---

## AI Provider Failover

The platform supports three AI providers with **automatic runtime failover**. If the primary provider fails (API error, rate limit, outage), the system transparently retries with the fallback — the customer never sees an error.

```
Customer sends message
  → Try PRIMARY provider (e.g., openai)
  → If it fails → Try FALLBACK provider (e.g., openrouter)
  → If both fail → Return graceful error
```

### Supported Providers

| Provider | Model | How It Connects |
|---|---|---|
| **OpenAI** | gpt-4o-mini | Direct via OpenAI SDK |
| **OpenRouter** | google/gemini-2.0-flash-001 (configurable) | Via OpenAI-compatible API (`baseURL: openrouter.ai/api/v1`) |
| **Gemini** | gemini-1.5-flash | Direct via Google Generative AI SDK |

OpenRouter uses the OpenAI-compatible API format, so it reuses the same SDK, streaming logic, and tool definitions as OpenAI — no duplicate code paths.

### Configuration Examples

| `AI_SERVICE_PROVIDER` | `AI_FALLBACK_PROVIDER` | Behavior |
|---|---|---|
| `openai` | `openrouter` | OpenAI primary, Gemini 2.0 Flash (via OpenRouter) as backup |
| `openrouter` | `openai` | Gemini 2.0 Flash primary, OpenAI as backup |
| `openai` | `gemini` | OpenAI primary, Gemini 1.5 Flash (native) as backup |
| `openrouter` | _(not set)_ | OpenRouter only, no fallback |

---

## System Sentry — AI-Powered Health Monitoring

The platform includes an autonomous health monitoring agent at `/api/admin/system-sentry` that acts as a proactive technical guardian for the entire PAAI architecture.

### What It Checks (5 parallel health probes)

| Probe | What It Does | Healthy | Degraded | Critical |
|---|---|---|---|---|
| **PostgreSQL / Prisma** | Runs a real DB query (tenant count + 24h leads) | Connected, <3s | Response >3s (pool exhaustion) | Connection failed |
| **OpenAI** | Pings `api.openai.com/v1/models` with API key | Key valid, <5s | Response >5s | Key invalid or API down |
| **Google Gemini** | Pings `generativelanguage.googleapis.com` | Key valid, <5s | Response >5s | Key invalid or API down |
| **OpenRouter** | Pings `openrouter.ai/api/v1/models` | Key valid, <5s | Response >5s | Key invalid or API down |
| **Environment Variables** | Checks 5 critical vars + at least 1 AI provider key | All present | — | Missing critical vars |

### The Reasoning Agent

Built with the **Vercel AI SDK** (`generateText` + `tool` from `ai` + `@ai-sdk/openai` + `zod`):

1. All 5 health probes run in parallel
2. Results are fed to a specialized AI agent with PAAI domain knowledge
3. For every non-healthy check, the agent outputs a **Technical Root Cause** and a **Proposed Fix**
4. If any check is `critical`, the agent calls the `sendAdminAlert` tool — triggering an emergency push notification to the system admin

**Example agent output for a database failure:**

> **Technical Root Cause:** Prisma connection pool exhausted. Supabase free tier limits concurrent connections to 20, and serverless cold starts can cause pool starvation.
>
> **Proposed Fix:** Add `?pgbouncer=true&connection_limit=10` to DATABASE_URL in Vercel environment variables, then redeploy.

### Emergency Mode

When the agent detects a critical failure, it automatically:

1. Calls the `sendAdminAlert` tool with severity, summary, root cause, and fix
2. Sends a push notification to the admin (via `ADMIN_TENANT_ID` or `paai-official` tenant)
3. The admin receives a browser notification with the full diagnostic

### Graceful Degradation

If the AI itself is down (all providers failed), the sentry **does not break** — it falls back to a rule-based analysis engine that still produces a full diagnostic report and sends admin alerts. The monitoring never depends on what it's monitoring.

### Automated Scheduling (Vercel Cron)

The sentry runs automatically via `vercel.json`:

```json
{
  "crons": [{
    "path": "/api/admin/system-sentry",
    "schedule": "0 9 * * *"
  }]
}
```

This runs a full health check **daily at 9 AM UTC**. On the Vercel Pro plan, you can increase this to every 5 minutes (`*/5 * * * *`) for near-real-time monitoring. You can also hit the endpoint manually at any time.

### Manual Access

```
GET https://paai-ai.vercel.app/api/admin/system-sentry
GET https://paai-ai.vercel.app/api/admin/system-sentry?secret=YOUR_SECRET
```

Returns a JSON response with the full health report, AI analysis, and alert status.

---

## RAG — Retrieval-Augmented Generation

The platform implements a **hybrid RAG + function calling** architecture. Action-oriented questions (booking, pricing, availability) use deterministic function calling. Knowledge questions (FAQs, policies, preparation instructions) use semantic vector search via RAG.

### How It Works

1. **Ingest**: Business owner adds knowledge via the dashboard (title, content, category)
2. **Chunk**: Long content is auto-split into ~1500-character paragraphs
3. **Embed**: Each chunk is converted to a 1536-dimensional vector via OpenAI `text-embedding-3-small`
4. **Store**: Chunks + embeddings are stored in PostgreSQL using **pgvector** (`vector(1536)` column)
5. **Search**: When a customer asks a knowledge question, the AI calls `search_knowledge_base`
6. **Retrieve**: The query is embedded and matched against stored chunks using cosine similarity, filtered by `tenant_id`
7. **Generate**: The top 3 most relevant chunks (above a similarity threshold) are injected as context, and the AI generates a grounded answer

### Per-Tenant Isolation

Every knowledge chunk is scoped to a `tenant_id`. Vector searches include a `WHERE tenant_id = $1` filter — Tenant A's knowledge is never returned for Tenant B's customers.

### Knowledge Categories

| Category | Example Content |
|---|---|
| **FAQ** | "Do you accept walk-ins?" → "Yes, but appointments are preferred and get priority." |
| **Policy** | "24-hour cancellation notice required. Late cancellations may incur a $25 fee." |
| **Preparation** | "Arrive with unwashed hair for best color absorption. Avoid conditioner 24 hours before." |
| **Service Detail** | "Our deep conditioning treatment uses Olaplex No. 3 and takes approximately 45 minutes." |
| **General** | "Free parking available behind the building. Wheelchair accessible entrance on Main St." |

### Dashboard Management

Business owners manage their knowledge base from the dashboard at `/dashboard/[slug]/knowledge`:
- Add new entries with title, content, and category
- Content is automatically chunked and embedded on save
- View all entries with chunk counts
- Delete entries (removes all associated chunks and embeddings)

---

## Tech Stack

| Layer | Technology |
|---|---|
| **Framework** | Next.js 16 (App Router, Turbopack, React 19) |
| **Language** | TypeScript (strict mode) |
| **Styling** | Tailwind CSS 4 |
| **Database** | PostgreSQL (Supabase-hosted) with **pgvector** for RAG embeddings |
| **ORM** | Prisma 7 with `@prisma/adapter-pg` for connection pooling |
| **Vector Search** | pgvector (cosine similarity), OpenAI Embeddings (`text-embedding-3-small`, 1536d) |
| **Auth** | Supabase Auth (`@supabase/ssr`) |
| **AI / LLM** | OpenAI SDK v6 (gpt-4o-mini) / Google Generative AI SDK (gemini-1.5-flash) / OpenRouter (google/gemini-2.0-flash-001) |
| **SMS** | Twilio SDK v5 |
| **Voice AI** | VAPI SDK v2 |
| **Payments** | Stripe SDK v20 |
| **Push Notifications** | web-push v3 (VAPID protocol) |
| **Date Parsing** | chrono-node v2 (natural language: "this Thursday", "next week") |
| **AI Agent SDK** | Vercel AI SDK v5 (`ai` + `@ai-sdk/openai`) + Zod for tool schemas |
| **Testing** | Vitest 4, Testing Library, MSW v2 |
| **Hosting** | Vercel (serverless) + Vercel Cron (scheduled health monitoring) |

---

## AI Tools & Function Calling

The AI has **7 callable tools** for real-time actions during conversations:

| Tool | Trigger Example | What It Does |
|---|---|---|
| `get_available_slots` | "Do you have anything Tuesday?" | Queries bookings table, returns open slots with morning/afternoon categorization. 3-second timeout with fallback. |
| `get_price_menu` | "What are your prices?" | Builds formatted Markdown menu from database. Filters UI clutter, groups unpriced services, adds industry upsell. |
| `calculate_price_estimate` | "How much for a haircut?" | Single-service lookup with fuzzy matching and alias resolution. Returns price or Soft Pivot. |
| `book_appointment` | Customer provides name + phone + date | Creates lead (with dedup), Prisma booking (UTC), triggers push notification to owner. |
| `update_booking` | "Can you add deep conditioning?" | Appends notes to an existing booking via server-injected booking_id. |
| `check_availability` | "Are you open Saturday?" | Simpler slot check with business hour filtering. |
| `send_sms` | AI-initiated confirmation | Sends SMS via Twilio to the customer's phone number. |

### System Prompt Design

The system prompt is dynamically assembled per-tenant from **6 layers**:

1. **Persona** — "High-Warmth Relational Concierge" tone (friendly human, not a chatbot)
2. **Business context** — name, industry, services, hours, address, current date
3. **Pricing rules** — no hallucinated prices (FORBIDDEN: $XX/$TBD/$0), verbatim menu output, soft pivot for missing prices
4. **Availability rules** — slot caching, no "one second" loops, slot lock on selection, morning/afternoon awareness
5. **Custom instructions** — business owner's own rules from the dashboard
6. **Booking context** — server-side injection of active `booking_id` for multi-turn booking flows

> Every rule exists because the AI violated it during testing, and a specific constraint was added to prevent it.

---

## Features

- **Multi-Tenant SaaS** — each business gets a unique URL, chat widget, and AI personality
- **Triple AI Engine** — OpenAI, Google Gemini, and OpenRouter (Gemini 2.0 Flash), switchable via environment variable
- **Runtime Failover** — if the primary AI provider fails, automatically retries with the fallback provider (zero downtime for customers)
- **7 AI Function-Calling Tools** — real-time database actions, not hallucinated responses
- **Dynamic Pricing Engine** — database prices + blacklist filtering + industry-aware formatting
- **Real-Time Availability** — 3-second timeout with graceful fallback, slot caching
- **Appointment Booking** — lead capture, dedup, UTC scheduling, push notification
- **SMS Integration** — Twilio send/receive for booking confirmations
- **Voice AI** — VAPI browser-based voice calls with shared knowledge base
- **Push Notifications** — Web Push (VAPID) with Confirm/Reject actions
- **Stripe Billing** — subscription management with checkout sessions and webhooks
- **Pricing Manager Dashboard** — add/edit services with prices, durations, categories
- **100+ Industry Configs** — tailored prompts, emojis, and service suggestions per vertical
- **PWA Support** — installable on mobile with service worker and offline caching
- **Website Scraper** — auto-fills knowledge base from the business's existing website
- **Embeddable Widget** — drop-in `<script>` tag for any website
- **System Sentry Agent** — AI-powered health monitoring that checks DB, all AI providers, and env vars every 5 minutes with auto-alerting on critical failures

---

## Database Design

The project uses a **hybrid database approach** — Supabase for auth and original tables, Prisma for structured models needing typed relations and migrations. Both hit the same PostgreSQL instance.

| Model | Purpose |
|---|---|
| **Tenant** | Business entity (name, slug, industry, usage limits, status) |
| **Service** | Priced offerings (name, base_price, duration, category) — Prisma |
| **Booking** | Calendar slots (start/end UTC, price estimate, lead_id) — Prisma |
| **Lead** | Captured customers (name, phone, service, date, time, status) |
| **KnowledgeBase** | AI training data (services, hours, custom prompt, FAQs) |
| **KnowledgeChunk** | RAG: per-tenant chunks + vector embeddings (pgvector) for semantic search |
| **UserProfile** | Auth users with roles (TENANT_OWNER, SUPER_ADMIN) |
| **Subscription** | Stripe billing (customer_id, subscription_id, plan) |
| **Conversation** | Chat sessions linked to tenant |
| **Message** | Individual messages in conversations |

---

## Project Structure

```
├── app/
│   ├── [slug]/page.tsx              # Dynamic tenant pages with chat + voice
│   ├── dashboard/[slug]/            # Tenant dashboard (leads, pricing, settings)
│   ├── (admin)/admin/               # Super admin panel
│   ├── api/
│   │   ├── chat/route.ts            # Core AI chat endpoint (~750 lines)
│   │   ├── vapi/chat/completions/   # Voice AI (OpenAI-compatible endpoint)
│   │   ├── dashboard/services/      # Pricing Manager CRUD
│   │   ├── admin/system-sentry/     # AI health monitoring agent (Vercel AI SDK)
│   │   ├── leads/                   # Lead management + SMS
│   │   ├── webhooks/                # Stripe, Twilio, Voice webhooks
│   │   └── notifications/           # Push subscription management
│   └── page.tsx                     # Landing page with live demo
├── components/
│   ├── ChatInterface.tsx            # Main chat UI (streaming, quick actions)
│   ├── VoiceWidget.tsx              # VAPI voice call widget
│   └── PushSubscription.tsx         # Push notification opt-in
├── lib/
│   ├── rag.ts                       # RAG pipeline (chunk, embed, pgvector search)
│   ├── availability.ts              # Slot engine (3s timeout, fallback, overlap detection)
│   ├── pricing.ts                   # Price menu + estimates (blacklist, fuzzy match)
│   ├── constants.ts                 # 100+ industry configs + system prompt builder
│   ├── usage.ts                     # Per-tenant usage metering
│   ├── notifications.ts             # Push notification delivery
│   └── system-health.ts             # Health probes + admin alerting
├── prisma/
│   ├── schema.prisma                # Full database schema (10 models)
│   └── migrations/                  # SQL migrations
└── public/
    ├── sw.js                        # Service worker (push + offline caching)
    └── widget.js                    # Embeddable chat widget script
```

---

## Design Decisions & Trade-offs

| Decision | Why | Trade-off |
|---|---|---|
| **Hybrid RAG + Function Calling** | Action-oriented use cases (book, price, check) use deterministic function calling. Knowledge questions (FAQs, policies, preparation) use RAG with pgvector semantic search. The AI autonomously chooses the right path per question. | Two retrieval patterns; embedding cost per knowledge upload (~$0.0001/chunk via `text-embedding-3-small`). |
| **Triple AI Provider with Failover** | Cost flexibility (OpenRouter/Gemini cheaper), redundancy (automatic failover on outage), client preference, access to latest models (Gemini 2.0 Flash via OpenRouter). | Slight behavioral differences between models require extra prompt tuning. |
| **Hybrid Database (Supabase + Prisma)** | Supabase for fast auth/prototyping; Prisma for typed models, relations, and migrations. | Two query patterns in the same codebase. |
| **3-Second Timeout with Fallback Slots** | Supabase free tier can be slow. Fallback common slots keep the conversation flowing instead of erroring. | Fallback slots are approximations, not real-time. |
| **Service Blacklist (40+ terms)** | Scraped websites include UI text ("Contact Us", "Gallery"). Filtering ensures clean price menus. | Needs maintenance as new junk terms appear. |
| **Slot Caching in Conversation Context** | AI was re-calling `get_available_slots` on every follow-up, causing latency and contradictions. | Cached slots may become stale in very long conversations. |
| **Streaming Responses** | Real-time "typing" effect is significantly better UX than waiting for full response. | Slightly more complex client-side state management. |
| **No Middleware** | Auth handled at API route level. Public routes (chat, tenant pages) don't need auth. | Manual auth checks in each protected route. |
| **AI-Powered System Sentry** | Uses Vercel AI SDK agent (not just static checks) so diagnostics include context-aware root cause analysis and actionable fixes. Falls back to rule-based analysis if AI is down — monitoring never depends on what it monitors. | Consumes one AI call per health check run (~every 5 min). |

---

## Getting Started

### Prerequisites

- Node.js 18+
- PostgreSQL database (Supabase recommended)
- At least one AI provider API key (OpenAI or Google Gemini)

### Installation

```bash
git clone https://github.com/sridharr9/paai_ai.git
cd paai_ai

npm install

npx prisma generate

cp .env.example .env.local
# Edit .env.local with your keys (see below)

npx prisma migrate deploy

npm run dev
```

Open [http://localhost:3000](http://localhost:3000) to see the landing page.

### Environment Variables

Create a `.env.local` file in the project root:

```env
# Database
DATABASE_URL=postgresql://user:pass@host:port/dbname

# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key

# AI Provider (pick one or more)
AI_SERVICE_PROVIDER=openai          # Primary: "openai", "gemini", or "openrouter"
AI_FALLBACK_PROVIDER=openrouter     # Fallback: auto-retries with this if primary fails
OPENAI_API_KEY=sk-...               # Required if using OpenAI
GEMINI_API_KEY=AI...                 # Required if using Gemini (native)
OPENROUTER_API_KEY=sk-or-...        # Required if using OpenRouter
OPENROUTER_MODEL=google/gemini-2.0-flash-001  # Optional: defaults to Gemini 2.0 Flash

# Application
NEXT_PUBLIC_ROOT_DOMAIN=localhost:3000
NODE_ENV=development

# Optional: SMS (Twilio)
TWILIO_ACCOUNT_SID=AC...
TWILIO_AUTH_TOKEN=...

# Optional: Payments (Stripe)
STRIPE_SECRET_KEY=sk_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Optional: Voice AI (VAPI)
VAPI_API_KEY=...

# Optional: Push Notifications
NEXT_PUBLIC_VAPID_PUBLIC_KEY=...
VAPID_PRIVATE_KEY=...

# Optional: System Sentry
SENTRY_SECRET=...                    # Protects the /api/admin/system-sentry endpoint
ADMIN_TENANT_ID=...                  # UUID of the admin tenant for emergency push alerts
```

---

## Deployment

### Vercel (Recommended)

```bash
git push origin main
npx vercel --prod
```

**Build configuration:**
- `postinstall`: `prisma generate` (auto-generates Prisma Client)
- `build`: `next build` (Turbopack production build)
- `serverExternalPackages`: `['openai']` (Node.js require compatibility)
- `cpus: 1` (prevents OOM on Vercel free tier)

---

## Testing

```bash
npm test              # Watch mode
npm run test:run      # Single run
npm run test:coverage # Coverage report
npm run test:ui       # Vitest visual UI
```

**Stack:** Vitest 4 + Testing Library + MSW v2 (API mocking) + jsdom

---

## Scripts

| Command | Description |
|---|---|
| `npm run dev` | Start development server |
| `npm run build` | Production build |
| `npm start` | Start production server |
| `npm test` | Run tests (watch mode) |
| `npm run test:run` | Run tests (single run) |
| `npm run test:coverage` | Run tests with coverage |
| `npm run lint` | ESLint check |

---

## License

This project is proprietary software. All rights reserved.

---

Built with Next.js 16, TypeScript, OpenAI, Google Gemini, OpenRouter, Vercel AI SDK, pgvector (RAG), Supabase, Prisma, Twilio, Stripe, VAPI, and Web Push.# PAAI-Readme
