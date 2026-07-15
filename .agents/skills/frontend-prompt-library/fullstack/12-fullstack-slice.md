---
title: Ship a Complete Full-Stack Feature Slice
category: FE + BE
skills: [react-client-mastery, nextjs-server-mastery]
principles: [SRP, OCP, LSP, ISP, DIP]
---

# 🧱 Prompt — Ship a Complete Full-Stack Feature Slice

> The **master prompt**. Use it to take a feature from a ticket to a mergeable PR: schema → server → API → UI → tests.
> It orchestrates the other prompts rather than replacing them. Phase-gated: the agent must stop and get your approval between phases.

---

```text
Follow `react-client-mastery` + `nextjs-server-mastery` (both, including their SOLID sections).
Ship the `{{FEATURE}}` feature end-to-end.

USER STORY: {{USER_STORY}}
STACK: {{RENDERING}}
ACCEPTANCE: {{ACCEPTANCE}}
NON-GOALS: {{NON_GOALS}}   ← do not build these. If you think they're required, ask.

REPO CONTEXT: {{EXISTING}}
  (Existing conventions to reuse: folder layout, http client, error types, UI primitives,
   test setup. Write "greenfield" if there is none.)

Work in PHASES. STOP after each phase and wait for my approval.
Do not write a single line of implementation before Phase 1 is approved.

RULE 0 — MATCH THE REPO BEFORE YOU MATCH THIS PROMPT.
The paths in the deliverables below are the default for a greenfield repo. If REPO CONTEXT shows
an existing convention — folder layout, an http client, an error type, UI primitives, a test
setup — REUSE IT instead of creating a parallel structure beside it. Creating a second way to
do something that already exists is a failure, even if the new way follows every rule below.
List every place you deviated from the tree below, and why.

──────────────────────────────────────────
PHASE 1 — DESIGN (no code)
──────────────────────────────────────────
Produce:
a) The file tree of everything you will create/modify.
b) For EACH file, ONE sentence stating its single reason to change (SRP).
   If a sentence needs the word "and", split the file.
c) The Zod contract: entity, create/update inputs, query params, error union.
d) The port interfaces (repository, mailer, gateway) — the abstractions the UI and
   use cases will depend on (DIP).
e) The RSC/client boundary table:
   | Component | Server or Client | Why | Exact fields crossing the boundary |
   Flag any field that must NOT cross (secrets, whole DB rows) — ISP.
f) Cache & invalidation plan: what is cached, for how long, what invalidates it.
g) Authz matrix: | Operation | Who may do it | Where is it enforced |
   ("Enforced in the UI" is not an answer. It must be enforced at the data layer.)
h) Risks + open questions for me.

──────────────────────────────────────────
PHASE 2 — CONTRACT & SERVER
──────────────────────────────────────────
- Shared Zod schemas (src/shared/api/) — ONE source of truth for FE and BE.
- Ports + adapters + container (DIP). `import 'server-only'` on all of them.
- Use cases: business rules, deps injected, unit-tested with FAKE ports (no DB).
- Server Actions: THIN — auth → validate → delegate → revalidate.
  Cross-cutting concerns via wrappers (withAuth/withValidation/withRateLimit) — OCP.
  ONE `ActionResult<T>` contract for every action — LSP.
- Authz enforced against the RESOURCE (ownership), not just the role.
- Structured logging. No secrets or stack traces in responses.

Gate: use-case unit tests green with fakes, no database needed. Show me them.

──────────────────────────────────────────
PHASE 3 — DATA LAYER (client)
──────────────────────────────────────────
- Gateway interface + implementation; `axios`/`fetch` confined to ONE file (DIP).
- Every response `Schema.parse`d at the boundary. `as T` is banned.
- Query key factory; explicit staleTime/gcTime; invalidateQueries after mutations.
- Optimistic update WITH rollback where {{ACCEPTANCE}} demands instant feedback.

──────────────────────────────────────────
PHASE 4 — UI
──────────────────────────────────────────
- Logic-in-hook, UI-in-component. The .tsx files contain JSX ONLY (SRP).
- Shareable UI state (filters/pagination/tabs) in the URL, not useState.
- Derived state computed during render — no useEffect state syncing.
- Shared components: forwardRef + {...rest} (LSP); cva variants, no boolean props (OCP).
- Narrow props; store slice selectors (ISP).
- ALL FOUR states: loading (skeleton), error (+retry), empty, success.
- A11y: roles, labels, focus management, keyboard paths, aria-live for async updates.
- i18n: zero hardcoded user-visible strings.

──────────────────────────────────────────
PHASE 5 — TESTS & VERIFY
──────────────────────────────────────────
- Unit: use cases (fake ports), hooks (renderHook), Zod schemas.
- Integration (RTL + msw): the full {{USER_STORY}} flow; all four data states;
  mutation failure + optimistic ROLLBACK.
- Security: anonymous caller rejected by the action; non-admin blocked;
  cannot mutate a resource they don't own.
- E2E (only if this is a critical path).
- Then walk {{ACCEPTANCE}} point by point and show me the evidence for each.

──────────────────────────────────────────
PHASE 6 — PR
──────────────────────────────────────────
- Conventional Commits: `feat({{FEATURE}}): ...`. One concern per commit.
- PR < 400 lines of real change. If it's bigger, split it and tell me how.
- PR body: what changed, the boundary/authz decisions, what you did NOT do
  ({{NON_GOALS}}), and how to test it manually.
- Self-review against BOTH skills' PR checklists (including all 5 SOLID items each)
  and paste the filled checklist.

Begin with PHASE 1. Do not write implementation code yet.

WHEN THE IMPLEMENTATION IS DONE (not before the gate above):
run typecheck, lint, and the tests, and paste the real output. If something fails, say so plainly
and show it. If you could not run them, say that explicitly. NEVER report "done" on code you have
not executed.
```
