# Architectural Directive: Shopify Merchant Copilot (WAT Framework)

## 1. THE MISSION

We are building a production-grade Shopify "Merchant Copilot" — an AI chat interface embedded inside Shopify admin. Merchants type requests in plain English and the AI executes them, always pausing for human approval before touching live store data.

The owner is a non-technical merchant/entrepreneur. Explain technical decisions in plain language. Every feature should be described in terms of what the merchant can *do* with it, not what the code does.

---

## 2. THE WAT FRAMEWORK

### A. Workflows (The "What" — Business Logic SOPs)

- **Definition:** Non-executable Markdown files that act as Standard Operating Procedures.
- **Location:** `docs/workflows/*.md`
- **Responsibility:** Define business rules, step-by-step procedures, and edge-case handling (e.g., `bulk-discounting.md`, `product-description.md`, `inventory-audit.md`).
- **Claude's Rule:** Before starting a complex task, check the relevant workflow file. If one doesn't exist, propose creating one before writing code.
- **Who writes them:** The merchant can write these in plain English. Claude reads them as constraints before acting.

### B. Agents (The "How" — Orchestration & Reasoning)

- **Definition:** The brain of the application — manages state, memory, and decision-making.
- **Location:** `app/lib/agent/`
- **Responsibility:**
  - Parse user intent and retrieve the correct Workflow.
  - Maintain conversation thread and memory context.
  - Manage the human-in-the-loop approval flow for all write operations.
- **Constraint:** Agents do not call Shopify APIs directly. They only decide *which* Tool to call and pass inputs to it.

### C. Tools (The "Action" — Deterministic Execution)

- **Definition:** Pure, single-purpose TypeScript functions that interact with external systems.
- **Location:** `app/lib/shopify/` (Shopify API) and `app/lib/memory/` (memory operations)
- **Responsibility:**
  - Execute specific mutations (e.g., `updateProductPrice`, `createDiscount`).
  - Validate inputs using Zod schemas before execution.
  - Return standardized result or error objects — never throw raw errors to the agent.
- **Constraint:** Tools are stateless and dumb. They execute; they do not decide.

---

## 3. TECH STACK (Do Not Deviate Without Explicit Discussion)

| Layer | Choice | Why |
|---|---|---|
| Framework | React Router 7 + Vite | Shopify's officially maintained package (`@shopify/shopify-app-react-router`). Remix v2 merged into React Router 7 — this is the current official Shopify template. |
| Language | TypeScript strict mode | Type safety across all layers |
| AI SDK | `@anthropic-ai/sdk` raw only | Approval flow requires manual tool_use interception |
| AI Model (default) | `claude-sonnet-4-6` | Cost vs capability balance |
| AI Model (memory extraction) | `claude-haiku-4-5` | Lightweight post-turn fact extraction |
| Database | PostgreSQL 16 + Prisma 7 | Multi-tenant, JSONB for messages, production from day 1 |
| Session storage | `@shopify/shopify-app-session-storage-prisma` | Replaces template default SQLite |
| Validation | Zod | All tool inputs and API response shapes |
| UI | Polaris | Ships with Shopify template, native admin look |
| Testing | Vitest + MSW (unit/integration), Playwright (E2E) | |

**Never use:** SQLite (even locally), Vercel AI SDK (`ai` package), LangChain, LangGraph, or any other agent framework library. We ARE WAT — we build it ourselves with the raw SDK.

---

## 4. ABSOLUTE RULES

1. **Every Shopify write requires human approval.** No exceptions. `update_product_price`, `update_product_description`, `create_product_draft`, `create_discount` all go through: `PendingAction(PENDING)` → ApprovalCard → user confirms → execute → `AuditLog`.

2. **Every database query must be scoped by `storeId`.** Never query across tenants. Every Prisma `findMany`/`findFirst` must include `where: { storeId }`.

3. **Never parse partial JSON from Claude tool inputs.** Always call `stream.finalMessage()` after the stream ends to get complete `tool_use` blocks. Never process `input_json_delta` events mid-stream.

4. **Use raw `fetch()` for the streaming chat endpoint.** `useFetcher` buffers until complete. `EventSource` is GET-only. Neither works for streaming chat — use `fetch()` with a `ReadableStream` reader.

5. **Prisma runs on Node.js only.** Never use Edge runtime adapters (Cloudflare Workers, etc.). Deploy to Railway, Render, or Fly.io.

6. **Secrets in `.env` only.** Never hardcode API keys or tokens. `Store.accessToken` is AES-256-GCM encrypted at rest using `ENCRYPTION_KEY` env var.

7. **All Shopify scopes declared upfront in `shopify.app.toml`.** Scope changes require app re-install on the dev store.

8. **Sequential approval for multiple tool calls.** If Claude returns 3 `tool_use` blocks in one response, show cards one at a time. Never approve in parallel — it creates audit log race conditions.

9. **Prompt caching on system prompt + store memory block.** Always add `cache_control: { type: "ephemeral" }` to the system prompt and memory injection block. Never cache the message history (changes every turn).

10. **Audit log every store mutation.** Every `PendingAction` that reaches `EXECUTED` must produce an `AuditLog` row with `before` and `after` snapshots.

---

## 5. THE APPROVAL FLOW (Core Architecture — Never Bypass)

```
User types request
  → POST /api/chat
  → Load last 40 messages from DB + StoreMemory
  → Build system prompt with memory injected (cached)
  → Call anthropic.messages.stream() with tools
  → Stream text deltas to client as SSE "text_delta" events
  → stream.finalMessage() → get complete tool_use blocks
  → READ tools (read_products, get_analytics, read_collections):
      → Execute immediately against Shopify
      → Append tool_result to messages
      → Continue Claude stream
  → WRITE tools (price, description, draft, discount):
      → Create PendingAction(status: PENDING)
      → Emit SSE "tool_use_start" event to client
      → Client renders ApprovalCard (shows before/after)
      → User clicks Approve → POST /api/tool-approve
          → Shopify mutation executes (synchronous, awaited)
          → PendingAction: PENDING → APPROVED → EXECUTED
          → AuditLog row written (before + after)
          → Return result to client
      → Client POSTs /api/chat again with tool_result in history
      → Claude generates final human-readable summary
      → User clicks Reject → PendingAction: REJECTED
          → Claude acknowledges, no change made
```

**SSE event format (server → client):**
```
event: text_delta
data: {"delta": "Here is what I'll do..."}

event: tool_use_start
data: {"tool_call_id": "toolu_01...", "tool_name": "update_product_price", "tool_input": {...}}

event: done
data: {}
```

---

## 6. TOOL CLASSIFICATION

Defined in `app/lib/agent/tool-classifier.ts`.

| Tool | Type | Approval Required? |
|---|---|---|
| `read_products` | read | No |
| `get_analytics` | read | No |
| `read_collections` | read | No |
| `update_store_memory` | write | No (safe, no store mutation) |
| `update_product_price` | **WRITE** | **Yes** |
| `update_product_description` | **WRITE** | **Yes** |
| `create_product_draft` | **WRITE** | **Yes** |
| `create_discount` | **WRITE** | **Yes** |

---

## 7. DATABASE MODELS (Summary)

Full schema in `prisma/schema.prisma` — that file is the source of truth.

| Model | Purpose |
|---|---|
| `Store` | Tenant root. `shopDomain` unique. `accessToken` encrypted. |
| `Session` | Shopify OAuth sessions (managed by Prisma session adapter). |
| `Conversation` | One chat thread per store+user. Has `storeId`, `userId`, `userRole`. |
| `Message` | `content: Json` stores exact Anthropic `ContentBlock[]` array — no translation. |
| `PendingAction` | One row per write tool call. `toolCallId @unique` prevents double-execution. |
| `StoreMemory` | Persistent brand facts. `@unique([storeId, key])` makes upserts safe. |
| `AuditLog` | Immutable. `before` + `after` snapshots. Never deleted. |

---

## 8. PROJECT DIRECTORY STRUCTURE

```
docs/
  workflows/              ← Business logic SOPs (markdown, written by merchant or Claude)
    product-description.md
    bulk-discounting.md
    inventory-audit.md

app/
  shopify.server.ts       ← Central Shopify singleton. Every server file imports from here.
  lib/
    db.server.ts          ← Prisma singleton. Never instantiate PrismaClient elsewhere.
    auth.server.ts        ← requireStoreAccess(request, minimumRole)
    agent/
      claude.server.ts    ← Anthropic streaming wrapper
      tools.ts            ← All Claude tool definitions (JSON schemas)
      system-prompt.ts    ← Builds system prompt, injects StoreMemory with cache_control
      executor.server.ts  ← Routes tool name → shopify/ module
      tool-classifier.ts  ← WRITE_TOOLS / READ_TOOLS sets
    shopify/
      graphql-client.server.ts  ← Rate-limit-aware Shopify GraphQL wrapper
      rate-limiter.server.ts    ← Token bucket per store
      products.server.ts
      pricing.server.ts
      discounts.server.ts
      analytics.server.ts
      collections.server.ts
    memory/
      store-memory.server.ts    ← CRUD for StoreMemory model
      memory-extractor.server.ts ← Post-turn fact extraction via Claude Haiku
    security/
      encrypt.server.ts   ← AES-256-GCM for access tokens
      sanitize.server.ts  ← Prompt injection detection
      rate-limit.server.ts ← Per-store/user request rate limiter

  routes/
    app.copilot.tsx               ← Main chat page
    api.chat.ts                   ← POST streaming action (core of the app)
    api.tool-approve.ts           ← Executes approved write actions
    api.tool-reject.ts            ← Records rejections
    api.webhooks.$.ts             ← Shopify webhook handler (HMAC verified)
    app.settings.memory.tsx       ← View/edit store memory entries
    app.settings.audit.tsx        ← Audit log viewer

  components/chat/
    ApprovalCard.tsx
    MessageBubble.tsx
    ChatInput.tsx
    ConversationSidebar.tsx
    cards/AnalyticsCard.tsx

  hooks/
    useChat.ts            ← useReducer state machine (not useState — prevents tearing)
    useChatStream.ts      ← SSE consumer using raw fetch()

prisma/
  schema.prisma

tests/
  unit/
  integration/
  e2e/
```

---

## 9. SHOPIFY API RATE LIMITS

GraphQL Admin API: 1000-point bucket, 50 points/second refill. Track `extensions.cost.throttleStatus.currentlyAvailable` in every response. On `429`: exponential backoff, max 3 retries, 2s cap. Default query size: `first: 20`, not `first: 250` unless explicitly needed for bulk operations.

Rate limiter lives in `app/lib/shopify/rate-limiter.server.ts` as an in-memory per-store token bucket (Map).

---

## 10. MEMORY SYSTEM

**v1 — category-filtered recency (no vectors).**
Load all `StoreMemory` for the store, `ORDER BY updatedAt DESC`, grouped by category, max 20 entries (~1500 tokens). Inject into system prompt with `cache_control: { type: "ephemeral" }`.

**Memory categories:** `BRAND_VOICE`, `PRICING_RULES`, `PRODUCT_RULES`, `CUSTOMER_RULES`, `STORE_CONTEXT`, `OPERATOR_PREFS`

**Extraction:** After each conversation turn, run a lightweight `claude-haiku-4-5` call (no streaming, no tools) to extract any new facts the merchant mentioned. Upsert into `StoreMemory`.

**v2 (deferred):** Add `pgvector` + `embedding vector(1536)` column when stores exceed ~200 memory entries.

---

## 11. SHOPIFY APP SCOPES

In `shopify.app.toml`:
```toml
[access_scopes]
scopes = "read_products,write_products,read_inventory,write_inventory,read_orders,read_analytics,write_discounts,read_discounts"
```
Scope changes require re-installing the app on the dev store.

---

## 12. DEVELOPMENT COMMANDS

```bash
# Start local dev with Shopify tunnel
shopify app dev

# Create and run database migrations
npx prisma migrate dev

# Generate Prisma client after schema changes
npx prisma generate

# Run unit/integration tests
npm test

# Run E2E tests
npx playwright test

# Type check only (no emit)
npx tsc --noEmit

# Open Prisma Studio (DB GUI)
npx prisma studio
```

---

## 13. DEVELOPMENT WORKFLOW (Per Feature)

1. **Check Workflow doc** — does `docs/workflows/<feature>.md` exist? If not, write it first (plain English business rules).
2. **Design the Tool** — create the TS module in `app/lib/shopify/` with a Zod input schema.
3. **Add to tool definitions** — update `app/lib/agent/tools.ts` with the Claude tool JSON schema.
4. **Classify the tool** — add to `WRITE_TOOLS` or `READ_TOOLS` in `tool-classifier.ts`.
5. **Connect the executor** — update `executor.server.ts` to dispatch the new tool name.
6. **Write the test** — Vitest unit test with MSW mocking the Shopify GraphQL endpoint.
7. **Test end-to-end** — use `shopify app dev` against the dev store, trigger the tool via chat.

---

## 14. BUILD PHASES

| Phase | Deliverable | Status |
|---|---|---|
| 1 | Scaffold + infrastructure (React Router 7 template, Prisma, PostgreSQL) | In progress |
| 2 | Auth + tenant bootstrap (afterAuth hook, RBAC) | Pending |
| 3 | Chat UI shell (Polaris layout, SSE stream reader, conversation list) | Pending |
| 4 | Agent engine core (Claude streaming, tools, message persistence) | Pending |
| 5 | Approval flow (ApprovalCard, tool-approve/reject routes, AuditLog) | Pending |
| 6 | Shopify tool implementations (all 7 tools with MSW tests) | Pending |
| 7 | Shopify API rate limiting (token bucket, retry logic) | Pending |
| 8 | Store memory system (extraction, injection, settings UI) | Pending |
| 9 | Analytics dashboard (Polaris DataTable, inline analytics cards) | Pending |
| 10 | Security hardening (HMAC webhooks, encryption, CSP, audit UI) | Pending |

---

## 15. SECURITY CHECKLIST (OWASP ASVS Level 2)

- [ ] Shopify OAuth for all auth (no custom login in v1)
- [ ] Webhook HMAC verification on all `/api/webhooks/*`
- [ ] `Store.accessToken` AES-256-GCM encrypted at rest
- [ ] All DB queries scoped by `storeId` (no cross-tenant leakage)
- [ ] `PendingAction.toolCallId @unique` (idempotency — no double-execution)
- [ ] Input sanitization: strip prompt injection patterns before sending to Claude
- [ ] Rate limiting: 10 messages/minute per store+user on `/api/chat`
- [ ] No secrets in logs or error responses in production
- [ ] CSP headers on all routes
- [ ] RBAC: `STORE_OWNER` / `STORE_ADMIN` / `VIEW_ONLY`

---

## 16. END-TO-END SMOKE TEST (What "Done" Looks Like)

1. `shopify app dev` → install on dev store → embedded app loads inside admin
2. "What products do I have?" → streamed response with real product list
3. "Lower the price of [product] to $19.99" → ApprovalCard with current vs. new price
4. Click Approve → Shopify variant updates → Claude confirms in chat
5. Click Reject → Claude acknowledges, no change made
6. New conversation → state brand voice → next conversation Claude uses it unprompted
7. Audit log shows: request → tool call → approval → mutation result
8. Forged webhook → 401
9. 11 messages in 1 minute → rate limited
