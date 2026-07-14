---
title: Example — Build CRUD with Server Actions
type: example
pairs_with: 10-server-action-crud.md
---

# 📘 Example — Build CRUD with Server Actions

> A fully filled-in run of [`10-server-action-crud.md`](10-server-action-crud.md) on a real scenario.
> Copy the prompt block below and paste it into your agent as-is.

---

```text
Follow `nextjs-server-mastery` (+ `react-client-mastery` for the form).
Implement create (invite), revoke, resend for Invitation using Server Actions.

AUTHZ: only admins may invite; an admin may revoke any invitation, a member may revoke ONLY an invitation they themselves sent
SIDE EFFECTS: send the invite email via Resend on create/resend; write an audit-log entry on revoke
REVALIDATE: /team, /team/invitations

REPO CONTEXT: `lib/server/ports/` + `adapters/` + `container.ts` ALREADY exist for billing
  (a Stripe adapter is wired there) — add the invitation port/adapter to the same container,
  do not stand up a second one. `ActionResult<T>` is ALREADY defined in
  `src/shared/api/action-result.ts` — import it, do not redefine it. `withAuth` already
  exists in `src/lib/server/action-wrappers.ts` — reuse it and only add what is missing
  (withRateLimit / withValidation) beside it. Prisma + Resend are already configured.

NON-GOALS: No bulk CSV invite. No SSO / SCIM provisioning. No invite-expiry cron job
  (expired invitations are handled lazily on read for now).

═══ ARCHITECTURE — the dependency arrow points INWARD, always ═══

  InvitationForm.tsx ('use client', useActionState)
        ↓ calls
  actions/invitation-actions.ts     ← THIN. auth → validate → delegate → revalidate
        ↓ calls
  lib/server/use-cases/*.ts         ← business rules. Knows nothing about HTTP/FormData.
        ↓ depends on
  lib/server/ports/*.ts             ← INTERFACES you own (InvitationRepository, Mailer)
        ↑ implemented by
  lib/server/adapters/*.ts          ← Prisma, Resend, Stripe (the replaceable details)
        ↑ wired in
  lib/server/container.ts           ← the ONLY file that knows the concrete classes

═══ RULES ═══

0. MATCH THE REPO BEFORE YOU MATCH THIS PROMPT.
   The paths in DELIVERABLES below are the default for a greenfield repo. If REPO CONTEXT shows
   an existing convention — folder layout, an http client, an error type, UI primitives, a test
   setup — REUSE IT instead of creating a parallel structure beside it. Creating a second way to
   do something that already exists is a failure, even if the new way follows every rule below.
   List every place you deviated from the tree below, and why.

1. SRP — every Server Action does EXACTLY four things, in this order:
     a) authenticate  → `const session = await getSession()`
     b) validate      → Zod `safeParse` on the input
     c) delegate      → call the use case
     d) revalidate    → `revalidatePath` / `revalidateTag`
   NO SQL, NO email, NO Stripe inside the action. If the action imports `db` AND
   `resend` AND `stripe`, it is a god-function — split it.

2. SECURITY — a Server Action is a PUBLIC ENDPOINT:
   - Auth is checked INSIDE the action. Middleware alone is NOT protection.
   - Authorization per the rules above (only admins may invite; an admin may revoke any
     invitation, a member may revoke ONLY an invitation they themselves sent) is checked
     against the RESOURCE, not just the role
     (an authenticated user must not be able to revoke someone else's invitation by
     passing a different id — this is the #1 real-world Server Action bug).
   - EVERY input is Zod-validated. `formData.get('role') as string` is BANNED —
     it is a privilege-escalation vector.
   - Rate-limit writes and auth actions.
   - Never return internal error details, stack traces, or DB errors to the client.

3. LSP — ONE result contract for EVERY action, no exceptions:
     export type ActionResult<T> =
       | { ok: true; data: T }
       | { ok: false; fieldErrors?: Record<string, string[]>; message?: string };
   Expected failures (validation, conflict) RETURN `{ ok: false }`.
   Only truly exceptional cases (unauthorized, infra down) THROW to the error boundary.
   One action must never throw where another returns — that makes `useActionState`
   unusable and forces callers to special-case each action.

4. OCP — cross-cutting concerns are COMPOSED, not copy-pasted:
     export const revokeInvitation = withAuth({ role: 'admin' })(
       withRateLimit({ limit: 5, window: '1m' })(
         withValidation(RevokeInvitationSchema)(async (input, ctx) => { ... })));
   Adding audit logging later must mean adding ONE wrapper, not editing N actions.

5. DIP — the use case depends on PORTS, never on Prisma/Resend/Stripe:
     export async function createInvitationUseCase(
       input: CreateInvitationInput,
       deps = { invitationRepo, mailer, audit },   // injected → unit-testable with fakes
     ) { ... }
   `import 'server-only'` on every port, use case, and adapter.

6. LSP for repositories — the fake and the real implementation must fail IDENTICALLY:
     findById(id): Promise<Invitation | null>   // null when missing — NEVER throws
     create(input): Promise<Invitation>         // throws ConflictError on duplicate
   A fake that returns null where Prisma throws = green tests, 500s in production.

7. ISP — `select` only the needed columns. Return a NARROW DTO from the action,
   not the DB row. Whatever the action returns is serialized to the client.

8. SIDE EFFECTS (send the invite email via Resend on create/resend; write an audit-log
   entry on revoke) live in the use case behind their own port
   (Mailer, AuditLog). They must NOT be able to fail the whole mutation silently —
   decide and state explicitly: is the email transactional (rollback on failure)
   or fire-and-forget (log and continue)?

9. THE FORM (client island):
   - `useActionState` bound to the action; reuse the SAME Zod schema client-side
     with react-hook-form for instant feedback (progressive enhancement:
     it must still work with JS disabled).
   - Map `fieldErrors` from ActionResult back onto the fields via `setError`.
   - `useFormStatus` for the pending state; disable submit while pending.
   - Success/error announced in an `aria-live` region.

10. CACHE — every mutation revalidates /team and /team/invitations. Forgetting this is the most
    common Server Action bug: the write succeeds, the UI shows stale data,
    and the user clicks the button again.

11. LOGGING — structured logs (`logger.info({ action, userId, requestId })`),
    never `console.log`. Capture exceptions with context.

═══ DELIVERABLES ═══
  src/shared/api/invitation/invitation-schema.ts   (shared FE+BE)
  src/lib/server/ports/invitation-repository.ts
  src/lib/server/ports/mailer.ts
  src/lib/server/adapters/prisma-invitation-repository.ts
  src/lib/server/adapters/resend-mailer.ts
  src/lib/server/container.ts
  src/lib/server/use-cases/create-invitation.ts   (+ revoke / resend)
  src/lib/server/action-wrappers.ts               (withAuth, withValidation, withRateLimit)
  src/actions/invitation-actions.ts
  src/features/invitation/components/InvitationForm.tsx
  tests: use cases with FAKE ports (no DB); actions for authz + validation rejection

Start by printing the port interfaces and the `ActionResult<T>` contract,
plus a one-line statement of what each layer's single reason to change is.
Then wait for my "go".

WHEN THE IMPLEMENTATION IS DONE (not before the gate above):
run typecheck, lint, and the tests, and paste the real output. If something fails, say so plainly
and show it. If you could not run them, say that explicitly. NEVER report "done" on code you have
not executed.
```
