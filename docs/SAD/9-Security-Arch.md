# SAD Chapter 9 — Security Architecture

**Document:** NexusAI Workspace — Software Architecture Document (SAD)
**Chapter:** 9
**Status:** Final
**Depends on:** SAD Chapters 1–8 (D44–D100), PRD Chapters 1–8 (D1–D43)
**Source:** SAD draft Chapter 9 (preserved in substance, corrected and expanded below)

> **This chapter was drafted with better awareness of the project's confirmed scope than several earlier ones** — password hashing (§9.5) matches PRD §7.6 exactly, prompt injection handling (§9.9) matches Chapter 6 exactly, no LangGraph/AI-Gateway-naming drift appears anywhere. The issues found are narrower: one terminology risk, one schema/implementation mismatch, and two completeness gaps against endpoints and file types confirmed in later chapters. Also delivering on Chapter 8's own recommendation: this chapter closes with a consolidated view of security decisions already made across Chapters 1, 3, 6, 7, and 8, rather than leaving them scattered.

---

## Purpose

Chapter 8 recommended this chapter be drafted against the document set's existing security content rather than as freestanding best practices — largely followed here. This chapter's job is twofold: correct the few places it drifted from confirmed scope, and actually deliver the consolidation value Chapter 8 asked for.

## Scope

Covers: security philosophy, objectives, authentication, JWT strategy, password storage, authorization, multi-tenant isolation, AI data isolation, prompt injection protection, input validation, file upload security, rate limiting, HTTPS, secure headers, secrets management, logging security, API abuse prevention, AI security, backup security, and future security enhancements. Closes with a consolidated cross-reference to security decisions made in prior chapters.

## Objectives of This Chapter

1. Resolve a terminology risk: "API Gateway" (§9.1) could be misread as the dedicated API Gateway component SAD Chapter 1 explicitly deferred to *future* — this chapter needs to mean Nginx, which exists in MVP.
2. Fix a genuine schema/implementation mismatch: §9.7's "every query includes workspaceId AND userId" doesn't match Chapter 4's actual schemas, which have no per-record `userId` field on sub-resources.
3. Close two completeness gaps against scope confirmed in later chapters: the file-upload allowlist is narrower than FR-4.1, and rate limiting doesn't cover the `/generate` endpoint Chapter 7 gave a distinct limit.
4. Deliver the consolidation Chapter 8 asked for — pull together security decisions already made across five prior chapters into one reference.

---

## 9.1 Security Philosophy (Corrected — terminology risk resolved)

**Terminology risk found and resolved (D101).** The diagram's second step, "API Gateway," risks being read as the dedicated API Gateway component — but **SAD Chapter 1 §1.10 explicitly lists "API Gateway" as a *future* enhancement**, not present in MVP. As written, this diagram could mislead a reader into thinking that component already exists. Corrected:

> Frontend → **Nginx Reverse Proxy** (SAD Ch.8 §8.6) → Authentication → Authorization → Validation → Business Logic → Database → AI Layer

This is the same layer the original diagram meant — just named to match what actually exists in MVP (Nginx) rather than a term already reserved for a distinct future component.

## 9.2 Security Objectives (Unchanged)

Standard, correct CIA-plus-AAA framing (confidentiality, integrity, availability, authentication, authorization, auditability). No issues.

## 9.3 Authentication (Unchanged)

Accurate and consistent with PRD FR-1.1–1.7. No issues.

## 9.4 JWT Strategy (Confirmed — new, welcome specificity)

15-minute access tokens, 7–30 day refresh tokens — the PRD and prior SAD chapters confirmed "JWT + refresh tokens" without specific lifetimes; this is the first place concrete numbers appear, and they're reasonable, standard values. No issues.

## 9.5 Password Storage (Confirmed accurate)

Argon2 preferred, bcrypt as fallback — matches **PRD Chapter 7 §7.6** exactly. No issues; good independent confirmation of an already-locked decision.

## 9.6 Authorization (Unchanged, one clarification)

Accurate. "Permission Granted" as the flow's terminal step is fine as written — it describes the *outcome* of an ownership check, not a role-based permission system, so it doesn't trigger the "ownership vs. role" wording issue flagged repeatedly elsewhere (D17/D30/D34/D92). Worth stating that explicitly here so it isn't mistaken for a recurrence: this section is consistent with v1's single-owner model (PRD D1), not implying RBAC.

## 9.7 Multi-Tenant Isolation (Corrected — schema mismatch resolved)

**Gap found (D102).** *"Every database query includes: workspaceId AND userId"* — taken literally, this doesn't match **SAD Chapter 4's actual schemas**: documents, conversations, messages, and memory records carry `workspaceId`, but none carry a separate `userId` field (workspaces themselves carry `ownerId`, per §4.5). Implementing this section literally would require schema changes not otherwise motivated by anything in this document set. Clarified:

> For v1 (single-owner workspaces, PRD D1), isolation is enforced as an **authorization gate**, not a literal per-record filter: before executing any workspace-scoped query, verify `req.user.id === workspace.ownerId`. Once that gate passes, querying by `workspaceId` alone is sufficient and correct — sub-resources (documents, conversations, messages, memory) don't need their own `userId` field, since they're already scoped to a workspace that has exactly one owner in v1. This becomes a genuine multi-field concern only if/when team workspaces (multiple owners per workspace) ship post-v1, at which point sub-resource-level `userId`/`createdBy` filtering would become meaningful.

This preserves the original's actual security intent (never let a request touch another workspace's data) while removing an implication that doesn't match the confirmed schema or single-owner model.

## 9.8 AI Data Isolation (Unchanged)

Accurate and matches SAD Chapter 4 §4.13's confirmed Qdrant payload schema (`workspaceId` field). No issues.

## 9.9 Prompt Injection Protection (Confirmed accurate)

Matches **SAD Chapter 6 §6.24** exactly (treating uploaded content as data, never instructions; system prompts take precedence). No issues — good consistency, no re-explanation needed here beyond the cross-reference in the consolidated section below.

## 9.10 Input Validation (Unchanged)

Accurate and standard. No issues.

## 9.11 File Upload Security (Corrected — allowlist completed)

**Gap found (D103).** The "Allow" list — PDF, DOCX, TXT, Markdown — omits **CSV and ZIP**, both confirmed supported formats since **PRD FR-4.1** and consistently listed in SAD Chapters 4 and 6. Taken literally, this security policy would block functionality the rest of the document set confirms is in scope. Corrected:

> **Allow:** PDF, DOCX, TXT, Markdown, CSV, ZIP.
> **Reject:** EXE, BAT, DLL, JS executables, and any file type not on the allow list.
> **ZIP-specific handling:** per PRD FR-7.9, a ZIP archive is unpacked and **each contained file individually validated** against this same allowlist before processing — a ZIP cannot be used to smuggle a disallowed file type past validation just because the outer container is itself allowed.

## 9.12 Rate Limiting (Corrected — Generate endpoint added)

**Gap found (D104).** The protected-endpoint list (Login, Registration, Password reset, AI chat, Search) omits `/generate` — which **SAD Chapter 7 §7.18** already gave its own, *lower* rate limit (10 req/min/user vs. chat's 60) specifically because it costs more per request. Corrected:

> **Protected endpoints:** Login, Registration, Password reset, AI chat (60 req/min/user), **Generate (10 req/min/user, per SAD Ch.7 §7.18)**, Search, Memory operations (standard authenticated-user limit — these are inexpensive MongoDB CRUD operations, no special throttling needed).

## 9.13 HTTPS (Unchanged)

Accurate and standard. No issues.

## 9.14 Secure Headers (Unchanged, one addition)

Accurate list (CSP, X-Content-Type-Options, X-Frame-Options, Referrer-Policy). Worth naming the implementation library explicitly, consistent with PRD Chapter 6/7's recommendation: **Helmet.js** (Express middleware) is the standard way to apply these headers without hand-rolling each one.

## 9.15 Secrets Management (Unchanged)

Accurate and consistent with SAD Chapters 2 and 8. No issues.

## 9.16 Logging Security (Expanded — one addition)

**Gap found (D105, minor).** The never-log list (passwords, JWTs, API keys, refresh tokens) is correct but doesn't repeat **SAD Chapter 8's D98 addition** — memory and generated-artifact content should also never appear in logs, given how much emphasis this project places on memory privacy specifically. Added: *"Memory record contents, generated-artifact contents (log operation metadata only — IDs, timestamps, operation type — never the content itself)."*

## 9.17 API Abuse Prevention (Unchanged)

Accurate and standard. No issues.

## 9.18 AI Security (Expanded — two cross-references added)

Accurate restatement of Chapter 6's guardrails. Two additions, both genuinely security-relevant AI behaviors that belong in the security chapter, not just the AI architecture chapter:

> - **Memory opt-out enforcement (FR-9.7)** is enforced server-side at the point of write in the AI Orchestration Engine's LangGraph pipeline (SAD Ch.6 §6.15) — not a client-side preference that can be bypassed by a direct API call.
> - **Generation output validation (SAD Ch.6 §6.16a)** — structured output from the Generation Engine is validated server-side before reaching the client; an unvalidated, malformed response is a distinct failure mode from a wrong-but-well-formed one and should never pass through unchecked.

## 9.19 Backup Security (Expanded, one addition)

Accurate. One addition, connecting to SAD Chapter 8 §8.15: backups implicitly include Memory records and Generated Artifacts, both privacy-sensitive — backup access controls and retention should be held to the same standard as the live data, not treated as a lower-sensitivity copy.

## 9.20 Future Security Enhancements (Confirmed accurate)

RBAC correctly deferred (consistent with PRD D1, team workspaces post-v1). MFA/SSO are reasonable future additions, consistent with PRD Chapter 6's already-listed future OAuth providers (Google/GitHub/Microsoft — SSO-adjacent). No issues.

## Chapter Summary (Unchanged)

Accurate. No issues.

---

## Consolidated Security Decisions (New — delivers Chapter 8's requested consolidation)

Chapter 8 specifically recommended this chapter pull together security content already scattered across the document set rather than redraft it. Doing that explicitly here, as a single reference:

| Decision | Established In | Summary |
|---|---|---|
| Centralized workspace-boundary enforcement | PRD §8.8, SAD Ch.1 | Isolation checks happen at the AI Orchestration Engine chokepoint, not duplicated per-engine |
| FastAPI never externally reachable | SAD Ch.8 (D95) | No Nginx route exists to FastAPI under any condition — not "internal if exposed" |
| LLM Gateway as the provider-call chokepoint | SAD Ch.1, Ch.2 | Single point to enforce workspace-boundary checks before any external LLM call |
| Cascading deletion as a privacy mechanism | SAD Ch.4 §4.17 (FR-3.7/FR-2.6) | Workspace/account deletion purges MongoDB, Qdrant, and File Storage together |
| Memory transparency access controls | SAD Ch.6 §6.15, Ch.7 §7.4 (FR-9.4–9.8) | View/delete/reset/opt-out, enforced server-side, traceable to source |
| Rate limiting per endpoint cost | SAD Ch.7 §7.18, this chapter §9.12 | Generate throttled harder than chat, reflecting real per-request cost |
| Document/version integrity | SAD Ch.4 §4.6/§4.13 (D28) | Superseded content deprioritized in retrieval, not silently lost |

This table is the authoritative security cross-reference for the rest of the document set — future chapters touching security should check against this table rather than re-deriving these decisions independently, which is exactly the drift pattern that recurred throughout Chapters 1–7 for other topics (LangGraph, memory transparency, naming).

---

## Design Decisions & Trade-offs Log (SAD Chapter 9)

| # | Decision Needed | Resolution | Status |
|---|---|---|---|
| D101 | "API Gateway" (§9.1) risked confusion with the future dedicated API Gateway component (SAD Ch.1 §1.10) | Renamed to "Nginx Reverse Proxy," matching what actually exists in MVP | **Resolved** |
| D102 | "Every query includes workspaceId AND userId" (§9.7) didn't match Chapter 4's actual schemas (no per-record userId) | Clarified as an authorization gate (verify ownership before query), not a literal per-record filter | **Resolved** |
| D103 | File-upload allowlist (§9.11) omitted CSV and ZIP, narrower than confirmed FR-4.1 | Added both; added ZIP-specific per-contained-file validation | **Resolved** |
| D104 | Rate-limiting list (§9.12) omitted `/generate`, which Ch.7 gave a distinct lower limit | Added, with the specific limit from Ch.7 §7.18 | **Resolved** |
| D105 | Logging-security list (§9.16) didn't include memory/generated-artifact content, unlike Ch.8's D98 | Added, cross-referenced | **Resolved** |
| D106 | AI Security (§9.18) didn't reference FR-9.7 enforcement point or generation output validation | Added both as explicit security-relevant behaviors | **Resolved** |

## Security Considerations

- This entire chapter is security considerations; the consolidated table above is the summary artifact worth keeping current as the document set grows.

## Scalability Considerations

- No new items; rate-limiting decisions (§9.12) are consistent with PRD D33's scalability position (throttle the most expensive endpoint hardest).

## Performance Considerations

- No new items.

## Best Practices Applied in This Expansion

- Fixed a schema/implementation mismatch (D102) rather than letting a security document imply a data model change that nothing else in the project motivates.
- Checked the file-upload and rate-limiting lists against the *actual* confirmed scope (FR-4.1, Chapter 7's endpoint list) rather than treating them as generic security boilerplate.
- Delivered the consolidation value Chapter 8 explicitly asked for, rather than just auditing this chapter's own new content in isolation.

## Implementation Notes for Later Chapters

- If Chapter 10 (Performance & Scalability) proceeds next, it should reference this chapter's rate-limiting table (§9.12) rather than re-deriving limits independently.
- The Consolidated Security Decisions table should be updated whenever a future chapter makes a new security-relevant decision, rather than left as a point-in-time snapshot.

## Future Enhancements (Chapter-Level)

- RBAC, MFA, SSO, audit dashboards, security event monitoring, hardware security keys — all already correctly listed in §9.20, consistent with prior chapters' future-evolution items.

---

## Senior Engineering Review

**Overall assessment:** this chapter shows the benefit of Chapter 8's recommendation to draft against existing project content — password hashing, prompt injection handling, and AI data isolation all matched confirmed decisions exactly, with no drift. The issues found are narrower and more specific than in earlier chapters: a terminology collision with a *named future component* (not a re-derivation of an already-decided architecture choice), a schema mismatch, and two completeness gaps against scope that was confirmed in chapters written *after* wherever this content was originally sourced from.

**Resolved in this revision:**
1. The "API Gateway" terminology risk (D101) — a new kind of issue for this document set: not a reversal of a decision, but a term already reserved for something else being reused ambiguously.
2. The workspace/userId schema mismatch (D102) — caught before it could motivate an unnecessary schema change.
3. Two completeness gaps (D103, D104) against scope this chapter's source material likely predated.

## Summary

This chapter delivers what Chapter 8 asked for: a security architecture consistent with confirmed scope, corrected in six specific places, and closed with a consolidated reference table pulling together security decisions already made across five prior chapters. That table is now the authoritative security cross-reference for the rest of the document set.

**Next step:** proceed to Chapter 10 (Performance & Scalability) as previewed — per Chapter 8's assessment, this is a consolidation opportunity (scaling, caching, bottleneck analysis are already scattered across Chapters 1, 2, 4, 7, 8) with one genuinely open item worth resolving there: the Redis cache-key/TTL specification, flagged as underspecified since PRD Chapter 2.
