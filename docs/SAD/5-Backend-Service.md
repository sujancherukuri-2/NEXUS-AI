# SAD Chapter 5 — Backend Service Architecture

**Document:** NexusAI Workspace — Software Architecture Document (SAD)
**Chapter:** 5
**Status:** Final
**Depends on:** SAD Chapters 1–4 (D44–D68), PRD Chapters 1–8 (D1–D43)
**Source:** SAD draft Chapter 5 (preserved in substance, corrected and completed below)

> **Good news first:** this chapter is the first in the SAD to correctly avoid the recurring Worker/AI-Service overlap mistake that appeared in three prior chapters (PRD D37, SAD D45, SAD D56/D57) — nowhere here does the Worker claim to perform extraction, chunking, or embedding. That pattern appears to be resolved going forward. The issues below are different in kind: the folder structure is significantly incomplete relative to the data model Chapter 4 just defined, one component name collides with an already-established term, and one section contradicts this same chapter's own stated service boundary.

---

## Purpose

This chapter is explicitly meant to be "the blueprint for writing clean, maintainable backend code" — which means completeness matters more here than in most chapters. A folder structure that's missing controllers for half the confirmed functional requirements isn't a style gap, it's a blueprint someone could actually start coding from and hit a wall on features 6 chapters proved were in scope.

## Scope

Covers: backend philosophy, layered architecture, complete folder structure, request lifecycle, each layer's responsibilities (routes, controllers, services, repositories, middleware), the Node-side AI communication component, queue/worker architecture, event-driven communication, error handling, validation, API versioning, dependency direction, service communication boundaries, logging, and production readiness.

## Objectives of This Chapter

1. Complete the folder structure against Chapter 4's confirmed collections and the PRD's confirmed FR list — every collection needs a model, repository, and (where user-facing) a service/controller/route.
2. Resolve a naming collision: "AI Service (Node Side)" as described here is not the same thing as the LLM Gateway (Module 13) already established in the Python AI Service — using overlapping names for two different components invites exactly the kind of confusion this whole review process has been catching.
3. Fix an internal contradiction: §5.7 implies the Node-side Search Service performs hybrid/semantic search directly, while §5.18 (this same chapter) explicitly states Node never talks to Qdrant.

---

## 5.1 Backend Philosophy (Unchanged)

Accurate and well-stated — the explicit non-goals (no LLM inference, embedding generation, OCR, or heavy parsing in Node) match every prior chapter's service boundary. No issues.

## 5.2 Backend Layered Architecture (Unchanged)

Standard, correct layered architecture (Routes → Controllers → Services → Repositories → MongoDB) with appropriate cross-cutting concerns. No issues.

## 5.3 Complete Folder Structure (Expanded — significant gaps closed)

**Gap found (D69):** the original structure lists routes for `chat` and `search` with no corresponding controllers, lists only three services (auth, workspace, document, ai, queue) against Chapter 4's ten confirmed collections, and lists only three models against Chapter 4's ten confirmed schemas. Completed:

```
backend/
src/
 ├── app.ts
 ├── server.ts
 │
 ├── config/
 │     database.ts
 │     redis.ts
 │     env.ts
 │
 ├── routes/
 │     auth.routes.ts
 │     workspace.routes.ts
 │     document.routes.ts
 │     chat.routes.ts
 │     search.routes.ts
 │     memory.routes.ts              ← added (FR-9.4–9.8)
 │     generation.routes.ts          ← added (Module 14)
 │     dashboard.routes.ts           ← added (FR-10.1–10.8)
 │     notification.routes.ts        ← added (FR-11.1–11.4)
 │
 ├── controllers/
 │     auth.controller.ts
 │     workspace.controller.ts
 │     document.controller.ts
 │     chat.controller.ts            ← added
 │     search.controller.ts          ← added
 │     memory.controller.ts          ← added
 │     generation.controller.ts      ← added
 │     dashboard.controller.ts       ← added
 │     notification.controller.ts    ← added
 │
 ├── services/
 │     auth.service.ts
 │     workspace.service.ts
 │     document.service.ts
 │     chat.service.ts               ← added
 │     search.service.ts             ← added (keyword-search + AI-service forwarding, see §5.7)
 │     memory.service.ts             ← added (FR-9.4–9.8)
 │     generation.service.ts         ← added (Module 14, forwards to AI Service)
 │     ai-service.client.ts          ← renamed from "ai.service.ts" (see §5.10, D70)
 │     queue.service.ts
 │
 ├── repositories/
 │     user.repository.ts
 │     workspace.repository.ts
 │     document.repository.ts
 │     conversation.repository.ts    ← added
 │     message.repository.ts         ← added
 │     memory.repository.ts          ← added
 │     generatedArtifact.repository.ts ← added
 │     job.repository.ts             ← added
 │     notification.repository.ts    ← added
 │     auditLog.repository.ts        ← added
 │
 ├── middleware/
 │     auth.ts
 │     validate.ts
 │     upload.ts
 │     error.ts
 │     rateLimit.ts
 │
 ├── models/
 │     User.ts
 │     Workspace.ts
 │     Document.ts
 │     Conversation.ts               ← added
 │     Message.ts                    ← added
 │     Memory.ts                     ← added
 │     GeneratedArtifact.ts          ← added
 │     Job.ts                        ← added
 │     Notification.ts               ← added
 │     AuditLog.ts                   ← added
 │
 ├── workers/
 ├── queues/
 ├── events/
 ├── utils/
 ├── validators/
 └── types/
```

Every addition traces directly to a Chapter 4 collection or a confirmed PRD functional requirement — nothing speculative was added.

## 5.4 Request Lifecycle (Unchanged)

The Create Workspace example is accurate and generalizes correctly to every other route now listed in §5.3. No issues.

## 5.5 Routes Layer (Unchanged)

Accurate. No business logic in routes is the right constraint. No issues.

## 5.6 Controller Layer (Unchanged)

Accurate — the thin-controller principle (`WorkspaceService.create()` vs. inline logic) is correctly stated and should apply uniformly to the newly added controllers (memory, generation, dashboard, notification) as well.

## 5.7 Service Layer (Expanded — two gaps fixed)

**Gap found (D72):** two services confirmed by the PRD have no entry here. Added:

> **Memory Service:** view workspace memory (FR-9.4), delete individual records (FR-9.5), reset workspace memory (FR-9.6), toggle passive-capture opt-out (FR-9.7). This service operates primarily against MongoDB directly (via `memory.repository.ts`) — reading, deleting, and resetting existing records doesn't require the AI Service, since those are plain CRUD operations. Writing *new* inferred memory happens inside the AI Service's LangGraph pipeline, not here.
>
> **Generation Service:** accepts a Module 14 request (flashcards, quiz, literature-review outline, revision plan), forwards it to the AI Service Client (§5.10), and persists the result via `generatedArtifact.repository.ts`. Handles the Markdown export request (FR-14.3).

**Contradiction found and resolved (D71):** the original Search Service description — "Hybrid search, Ranking, Filters" — implies Node performs semantic/hybrid search directly. **This directly contradicts §5.18 of this same chapter**, which explicitly states the backend does not communicate with Qdrant. Corrected:

> **Search Service:** performs keyword/metadata search directly against MongoDB (conversation and document metadata, per SAD Chapter 1's D46 split — Search Engine is split across two services). For semantic or hybrid search, it forwards the query to the AI Service Client (§5.10), which calls FastAPI's Retrieval Engine — Node never queries Qdrant itself. The Search Service's job is to call the right source and merge/rank the combined results, not to perform vector similarity computation locally.

Auth, Workspace, Document, and Chat Service descriptions are accurate and unchanged (Chat Service's "Call AI service" step is exactly the AI Service Client's job, §5.10).

## 5.8 Repository Layer (Unchanged)

Accurate — the "no business decisions in repositories" principle is correctly stated and applies to all seven newly added repositories in §5.3 as well.

## 5.9 Middleware (Unchanged)

Accurate and complete for MVP scope. No issues.

## 5.10 AI Service Client (Renamed and corrected — resolves a naming collision)

**Naming collision found and resolved (D70):** the original title, "AI Service (Node Side)," and its description of itself as "an AI Gateway" collides directly with the **LLM Gateway (Module 13)** already established in SAD Chapters 1–3 — which lives inside the *Python* AI Service and handles LLM provider abstraction (OpenAI-compatible, fallback per FR-13.4). These are two genuinely different components: this one is a thin Node-side proxy to FastAPI; that one is the provider-adapter pattern inside FastAPI. Using "AI Gateway" for both would make every future reference to "the AI Gateway" ambiguous.

**Renamed: AI Service Client** (`ai-service.client.ts`, per §5.3). Responsibilities unchanged from the original draft and correctly scoped:

- Forward requests to FastAPI (chat, generation, search)
- Handle timeouts
- Retry transient failures
- Stream responses (SSE passthrough)
- Save conversation history (via `conversation.repository.ts`/`message.repository.ts`)

One caution worth adding to the original: **retries on a streaming LLM call need to be idempotent-safe** — a naive retry after a partial stream could produce a duplicate partial response or a duplicate memory write. Recommend retries only apply before the first token is received; once streaming has started, a failure should surface to the client rather than silently retry.

## 5.11 Queue Service (Unchanged)

Accurate. The "Cleanup" job type is the correct home for the cascading-deletion job established in **SAD Chapter 4 §4.17** (FR-3.7/FR-2.6) — worth stating that connection explicitly here since this is where that job actually gets scheduled and monitored.

## 5.12 Worker Architecture (Unchanged — confirmed correct)

Accurate and, notably, correctly generic — it doesn't claim the worker performs extraction/chunking/embedding, avoiding the exact mistake three prior chapters made in similar sections. No changes needed.

## 5.13 Event-Driven Communication (Expanded — two events added)

**Gap found (D73):** the event list omits anything for Memory or Generation, despite both being confirmed v1 features with real UI implications (a user deleting a memory record, or a generation job completing, both benefit from the same event-driven notification pattern already used for documents). Added:

> **Additional events:** `MemoryUpdated`, `MemoryDeleted` (drives real-time UI updates for the memory-transparency screens implementing FR-9.4–9.6), `GenerationCompleted` / `GenerationFailed` (mirrors the existing `JobFailed` pattern, notifies the user when a flashcard/quiz/outline is ready).

## 5.14 Error Handling Strategy (Unchanged)

Accurate and standard. No issues.

## 5.15 Validation Strategy (Unchanged)

Accurate. No issues.

## 5.16 API Versioning (Expanded)

Accurate; extending the example list for completeness against §5.3's routes:

> `/api/v1/auth`, `/api/v1/workspaces`, `/api/v1/documents`, `/api/v1/chat`, `/api/v1/search`, `/api/v1/memory`, `/api/v1/generate`, `/api/v1/dashboard`, `/api/v1/notifications`

## 5.17 Dependency Direction (Unchanged)

Accurate — correct layered-architecture constraint. No issues.

## 5.18 Service Communication Principles (Confirmed — this is the source of truth D71 corrects §5.7 against)

Accurate and precisely stated: the backend talks to MongoDB, Redis, BullMQ, and FastAPI; it does not talk directly to the LLM provider or Qdrant. This section was correct as originally written — it's §5.7's Search Service description that needed to be brought into line with it, not the other way around.

## 5.19 Logging (Unchanged)

Accurate, and the explicit "don't log passwords/tokens/sensitive data" instruction is a good, specific security callout. No issues.

## 5.20 Production Readiness (Expanded — two items added)

**Gap found (D74):** the checklist doesn't reference two mechanisms Chapter 4 just confirmed. Added:

> - Cascading-deletion job (SAD Chapter 4 §4.17) verified working end-to-end (MongoDB + Qdrant + File Storage)
> - Idempotency keys applied to all retryable background jobs (SAD Chapter 4 §4.11)

Everything else in the original checklist (environment config, graceful shutdown, health checks, global error handling, validation, auth middleware, queue monitoring, structured logging) is accurate and unchanged.

## 5.21 Chapter Summary (Unchanged)

Accurate summary, now describing a complete backend rather than one missing half its service/repository/model layer.

---

## Design Decisions & Trade-offs Log (SAD Chapter 5)

| # | Decision Needed | Resolution | Status |
|---|---|---|---|
| D69 | Folder structure missing controllers/services/repositories/models for chat, search, memory, generation, dashboard, notifications, plus 7 of 10 confirmed collections had no model | Completed folder structure (§5.3) | **Resolved** |
| D70 | "AI Service (Node Side)" / "AI Gateway" naming collided with the LLM Gateway (Module 13, Python) | Renamed to **AI Service Client** | **Resolved** |
| D71 | Search Service (§5.7) implied Node performs hybrid/semantic search directly, contradicting §5.18's own stated boundary | Corrected: Node does keyword/metadata search; semantic/hybrid forwarded via AI Service Client | **Resolved** |
| D72 | Memory Service and Generation Service missing from §5.7 despite confirmed FRs | Added both | **Resolved** |
| D73 | Event list (§5.13) missing Memory and Generation events | Added `MemoryUpdated`, `MemoryDeleted`, `GenerationCompleted`, `GenerationFailed` | **Resolved** |
| D74 | Production Readiness checklist didn't reference Chapter 4's cascading-deletion job or idempotency keys | Added both as explicit checklist items | **Resolved** |

## Security Considerations

- The renamed AI Service Client's retry caution (§5.10) is a real security/correctness concern, not just a style note — a naive retry on a streaming call could double-write memory (FR-9.7 territory) or produce duplicate assistant messages. Recommend this be an enforced code-review checklist item, not just documentation.
- Memory Service operating directly against MongoDB (§5.7) for read/delete/reset operations — bypassing the AI Service for those specific operations — is correct and efficient, but means workspace-boundary checks (normally centralized at the AI Service/LLM Gateway per PRD §8.8) must also be explicitly enforced in Memory Service's own authorization logic, since this is one of the few paths that doesn't route through that chokepoint.

## Scalability Considerations

- The completed repository layer (§5.3) means every collection from Chapter 4 now has a clear, single access path — this matters for scalability because it prevents ad-hoc direct MongoDB queries from services, which would otherwise become impossible to optimize or migrate later without finding every call site.

## Performance Considerations

- Search Service's merge/rank step (§5.7, corrected) — combining Node-side keyword results with AI-Service-side semantic results — is a new piece of logic that didn't exist when the Search Service was (incorrectly) described as self-contained. Worth benchmarking this merge step specifically once implemented, since it's now an explicit fan-out/fan-in pattern rather than a single query.

## Best Practices Applied in This Expansion

- Completed the folder structure by deriving it directly from Chapter 4's confirmed collections and the PRD's FR list, rather than guessing at plausible-sounding additions.
- Caught a naming collision (D70) before it could cause confusion between two genuinely different components sharing the term "AI Gateway."
- Caught an internal contradiction within the same chapter (D71) — §5.7 and §5.18 disagreed with each other, not just with a prior chapter — and resolved it in favor of the more architecturally sound section (§5.18's explicit boundary statement).
- Explicitly acknowledged where this chapter got something right (avoiding the Worker/AI-Service overlap mistake) rather than only cataloging problems.

## Implementation Notes for Later Chapters

- Chapter 6 (AI Architecture, previewed next) should treat the AI Service Client (§5.10) as the Node-side contract it calls into — the LangGraph pipelines, Retrieval Engine, and Generation Engine all live behind this client's REST calls, not inside Node.
- The Search Service's fan-out/fan-in pattern (§5.7) needs its Python-side counterpart specified in Chapter 6 — what does FastAPI's search endpoint return, and how does Node merge it with keyword results?

## Future Enhancements (Chapter-Level)

- None beyond what's already logged in prior chapters (roles/permissions, API keys, integrations — all post-v1 per the PRD).

---

## Senior Engineering Review

**Overall assessment:** this chapter correctly internalizes a lesson from three prior chapters (the Worker/AI-Service boundary) — genuinely good sign. Its own issues are different: completeness (a blueprint chapter needs to cover every confirmed feature, and roughly half were missing scaffolding) and one internal contradiction between two of its own sections, not a conflict with an earlier chapter.

**Resolved in this revision:**
1. The folder structure (D69) now has a model, repository, and — where user-facing — service/controller/route for every collection Chapter 4 defined.
2. The naming collision between this chapter's "AI Gateway" and Module 13's "LLM Gateway" (D70) is resolved before it could cause confusion in code or in later chapters.
3. Search Service's self-contradiction with this same chapter's §5.18 (D71) is resolved in favor of the correct, already-stated boundary.

## Summary

This chapter completes the backend blueprint: folder structure now matches the confirmed data model and FR list, the Node-side AI communication component has a name that doesn't collide with an existing one, and the Search Service's description is consistent with this chapter's own stated service boundaries. The Worker/AI-Service overlap mistake that recurred three times in prior chapters does not appear here.

**Next step:** proceed to Chapter 6 (AI Architecture) as previewed — it should specify FastAPI's side of the AI Service Client contract (§5.10) and the Search Service's semantic-search endpoint (§5.7) concretely, since both are now defined from the Node side and need their Python-side counterpart.
