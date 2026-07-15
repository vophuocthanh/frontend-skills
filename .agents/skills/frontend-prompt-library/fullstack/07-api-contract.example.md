---
title: Example — Define the API Contract (Contract-First)
type: example
pairs_with: 07-api-contract.md
---

# 📘 Example — Define the API Contract (Contract-First)

> A fully filled-in run of [`07-api-contract.md`](07-api-contract.md) on a real scenario.
> Copy the prompt block below and paste it into your agent as-is.

---

```text
Follow `react-client-mastery` + `nextjs-server-mastery`. Define the API contract for User and Invitation.

SOURCE OF TRUTH: the Acme Console backend OpenAPI/Swagger spec at https://api.acme-console.dev/openapi.json (looser than the UI needs: `role` typed as a free-form string, `email` declared `nullable: true`)
ENDPOINTS: GET /users, GET /users/:id, POST /invitations, GET /invitations, POST /invitations/:id/revoke
AUTH: Bearer JWT in an httpOnly cookie

REPO CONTEXT: `src/shared/api/` already holds the billing contract in exactly this shape
  (`billing-schema.ts` + `billing-errors.ts` + `billing-contract.ts`) — mirror it for
  user and invitation. The typed error union already lives in `src/shared/api/errors.ts`
  (AppError + status→error mapping); EXTEND it, do not define a second error type.
  Zod is already a dependency and is used on both the client and the server.

NON-GOALS: Do NOT implement the endpoints or any UI — this is the contract only.
  No GraphQL migration. No changes to the backend OpenAPI spec itself.

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
     - UserSchema / InvitationSchema                     → the full entity as the API returns it
     - UserListItemSchema / InvitationListItemSchema     → the trimmed shape the list endpoint returns
     - CreateUserSchema / CreateInvitationSchema         → the request body for POST (no id, no timestamps)
     - UpdateUserSchema / UpdateInvitationSchema         → usually Create...Schema.partial()
     - UserQuerySchema / InvitationQuerySchema           → query params (page, search, sort, filters)
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
       User.role is z.enum(['admin', 'member', 'viewer']).
       Invitation.status is z.enum(['pending', 'accepted', 'revoked']).

7. Single location, shared by BOTH sides:
     src/shared/api/user/
     ├── user-schema.ts             (schemas + inferred types)
     ├── user-errors.ts             (typed error union + HTTP mapping)
     └── user-contract.ts           (endpoint map: method, path, input, output schema)
     src/shared/api/invitation/
     ├── invitation-schema.ts       (schemas + inferred types)
     ├── invitation-errors.ts       (typed error union + HTTP mapping)
     └── invitation-contract.ts     (endpoint map: method, path, input, output schema)

DELIVERABLES
- The schema files above.
- A markdown table of the contract:
  | Endpoint | Method | Input schema | Output schema | Errors | Auth |
- A list of every ASSUMPTION you had to make because the Acme Console OpenAPI spec was ambiguous.
  I will confirm these with the backend before we build on top of them.

The source is an OpenAPI spec, so derive the schemas from it faithfully and
flag any place where the spec is looser than what the UI actually needs
(e.g. `nullable: true` on the `email` field, which the UI treats as required,
and `role` typed as a free-form string where the UI supports only admin|member|viewer).

WHEN THE IMPLEMENTATION IS DONE (not before the gate above):
run typecheck, lint, and the tests, and paste the real output. If something fails, say so plainly
and show it. If you could not run them, say that explicitly. NEVER report "done" on code you have
not executed.
```
