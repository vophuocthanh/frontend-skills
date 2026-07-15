---
title: Define the API Contract (Contract-First)
category: FE + BE
skills: [react-client-mastery, nextjs-server-mastery]
principles: [ISP, DIP, LSP]
---

# 📜 Prompt — Define the API Contract

> **Run this BEFORE any integration prompt.** The contract is the seam between FE and BE.
> Everything downstream (client hooks, Server Actions, tests, mocks) is generated from it.
> One Zod schema, two consumers. No hand-written types, ever.

---

```text
Follow `react-client-mastery` + `nextjs-server-mastery`. Define the API contract for {{RESOURCE}}.

SOURCE OF TRUTH: {{SPEC_SOURCE}}
ENDPOINTS: {{ENDPOINTS}}
AUTH: {{AUTH}}

REPO CONTEXT: {{EXISTING}}
  (Existing conventions to reuse: folder layout, http client, error types, UI primitives,
   test setup. Write "greenfield" if there is none.)

NON-GOALS: {{NON_GOALS}}
  (Do NOT build these. If you believe one is actually required, STOP and ask me first.)

RULES

0. MATCH THE REPO BEFORE YOU MATCH THIS PROMPT.
   The paths in DELIVERABLES below are the default for a greenfield repo. If REPO CONTEXT shows
   an existing convention — folder layout, an http client, an error type, UI primitives, a test
   setup — REUSE IT instead of creating a parallel structure beside it. Creating a second way to
   do something that already exists is a failure, even if the new way follows every rule below.
   List every place you deviated from the tree below, and why.

1. ZOD IS THE CONTRACT. Every type is INFERRED, never hand-written:
     export const UserSchema = z.object({...});
     export type User = z.infer<typeof UserSchema>;
   A hand-written `interface User` that duplicates the schema is a bug — it WILL drift.

2. Separate the schemas by direction and by use case (ISP — don't ship one fat type):
     - {{RESOURCE}}Schema          → the full entity as the API returns it
     - {{RESOURCE}}ListItemSchema  → the trimmed shape the list endpoint returns
     - Create{{RESOURCE}}Schema    → the request body for POST (no id, no timestamps)
     - Update{{RESOURCE}}Schema    → usually Create...Schema.partial()
     - {{RESOURCE}}QuerySchema     → query params (page, search, sort, filters)
   Derive with `.pick()` / `.omit()` / `.partial()` / `.extend()` — do NOT copy-paste
   object literals between schemas.

3. NEVER expose secrets in a response schema. Explicitly `.omit()` fields like
   passwordHash, internal ids, stripeCustomerId. Whatever is in the schema is what
   crosses the wire — and, in RSC, what gets serialized into the HTML payload.

4. Envelope + errors (LSP — every endpoint fails the SAME way):
     - Define ONE response envelope and ONE error shape used by every endpoint.
     - Define the paginated envelope once:
         PaginatedSchema(item) → { data: item[], meta: { page, pageSize, total } }
     - Define the app error union: 400 validation / 401 / 403 / 404 / 409 conflict / 5xx.
     - Map HTTP status → a typed domain error class. Callers must never inspect
       raw status codes scattered across the codebase.

5. Boundary validation is MANDATORY on both sides:
     - FE: `Schema.parse(response)` on every read. `as User` is banned.
     - BE (Server Action / route handler): parse the INPUT. Never trust the client.
   If the backend contract is unstable, use `.safeParse()` at the boundary and
   report the parse failure as a first-class error state — do not silently continue.

6. Dates/money/enums:
     - Dates: `z.string().datetime()` on the wire; convert to `Date` only in the app
       layer. Never send a `Date` object across the RSC boundary (not serializable).
     - Money: integer minor units + currency code. Never floats.
     - Enums: `z.enum([...])` — never a bare `z.string()`.

7. Single location, shared by BOTH sides:
     src/shared/api/{{RESOURCE}}/
     ├── {{RESOURCE}}-schema.ts     (schemas + inferred types)
     ├── {{RESOURCE}}-errors.ts     (typed error union + HTTP mapping)
     └── {{RESOURCE}}-contract.ts   (endpoint map: method, path, input, output schema)

DELIVERABLES
- The schema files above.
- A markdown table of the contract:
  | Endpoint | Method | Input schema | Output schema | Errors | Auth |
- A list of every ASSUMPTION you had to make because {{SPEC_SOURCE}} was ambiguous.
  I will confirm these with the backend before we build on top of them.

If {{SPEC_SOURCE}} is an OpenAPI spec, derive the schemas from it faithfully and
flag any place where the spec is looser than what the UI actually needs
(e.g. `nullable: true` on a field the UI treats as required).

WHEN THE IMPLEMENTATION IS DONE (not before the gate above):
run typecheck, lint, and the tests, and paste the real output. If something fails, say so plainly
and show it. If you could not run them, say that explicitly. NEVER report "done" on code you have
not executed.
```
