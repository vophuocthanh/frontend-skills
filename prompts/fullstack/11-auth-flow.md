---
title: End-to-End Authentication Flow
category: FE + BE
skills: [nextjs-server-mastery, react-client-mastery]
principles: [SRP, LSP, DIP]
---

# 🔐 Prompt — End-to-End Authentication Flow

> Login, session, refresh, logout, route protection — across the FE/BE boundary.
> Two rules that are never negotiable: **tokens live in httpOnly cookies**, and **middleware is not authorization**.

---

```text
Follow `nextjs-server-mastery` + `react-client-mastery`. Implement the auth flow.

MODEL: {{AUTH_MODEL}}
PROVIDER: {{PROVIDER}}
PROTECTED: {{PROTECTED}}
PUBLIC: {{PUBLIC}}

REPO CONTEXT: {{EXISTING}}
  (Existing conventions to reuse: folder layout, http client, error types, UI primitives,
   test setup. Write "greenfield" if there is none.)

NON-GOALS: {{NON_GOALS}}
  (Do NOT build these. If you believe one is actually required, STOP and ask me first.)

═══ NON-NEGOTIABLE SECURITY RULES ═══

0. MATCH THE REPO BEFORE YOU MATCH THIS PROMPT.
   The paths in DELIVERABLES below are the default for a greenfield repo. If REPO CONTEXT shows
   an existing convention — folder layout, an http client, an error type, UI primitives, a test
   setup — REUSE IT instead of creating a parallel structure beside it. Creating a second way to
   do something that already exists is a failure, even if the new way follows every rule below.
   List every place you deviated from the tree below, and why.

1. TOKEN STORAGE — httpOnly, Secure, SameSite=Lax cookies, set BY THE SERVER.
   - NEVER localStorage or sessionStorage. Any XSS on any page steals the token.
   - NEVER put the token in a client-readable cookie or expose it via `NEXT_PUBLIC_*`.
   - The client code must never be able to READ the access token. It doesn't need to.

2. MIDDLEWARE IS NOT AUTHORIZATION. It is a redirect optimization, nothing more.
   - `middleware.ts` may redirect unauthenticated users away from {{PROTECTED}}.
   - EVERY Server Action, route handler, and RSC page MUST INDEPENDENTLY verify the
     session with `getSession()`. Server Actions are public endpoints — a middleware
     redirect does not stop a direct POST to them.
   - Authorization (role/ownership) is enforced at the DATA layer, not in the UI.
     Hiding a button is UX. It is not security.

3. VERIFY, DON'T DECODE. Always verify the signature and expiry server-side
   (`jwtVerify`). Never trust a decoded payload from the client.

═══ ARCHITECTURE (SRP + DIP) ═══

  lib/server/auth/session.ts    'server-only' — getSession(), createSession(), destroySession()
  lib/server/ports/auth-gateway.ts   interface AuthGateway { login, refresh, logout }
  lib/server/adapters/*.ts      the {{PROVIDER}} implementation
  actions/auth-actions.ts       thin: validate → delegate → set cookie → redirect
  middleware.ts                 redirect only. No DB calls, no heavy crypto.
  components/LoginForm.tsx      'use client' — useActionState

- DIP: the login use case depends on `AuthGateway`, not on {{PROVIDER}}'s SDK.
  Swapping providers must not touch the UI or the use case.
- SRP: session management, credential verification, and the form are three modules.

═══ FLOWS TO IMPLEMENT ═══

A. LOGIN
   - Zod-validate credentials (shared schema, FE + BE).
   - Rate-limit by IP AND by email (brute-force protection).
   - On success: set httpOnly cookies server-side, then `redirect()`.
   - On failure: return `{ ok: false, message: 'Invalid email or password' }`.
     Use the SAME generic message for wrong-email and wrong-password — never reveal
     which one was wrong (user enumeration).
   - Return the standard `ActionResult<T>` (LSP — same contract as every other action).
   - Timing: failure paths should not be measurably faster than success paths.

B. SESSION READ
   - `getSession()` wrapped in `React.cache()` → called in layout, page, and actions
     but verified ONCE per request.

C. REFRESH
   - Handled server-side. If the client-side HTTP client must handle 401s,
     it does so CENTRALLY: queue concurrent 401s → refresh ONCE → replay the queue →
     hard-logout if the refresh itself fails.
   - Rotate the refresh token on use; detect reuse of a rotated token (theft signal).

D. LOGOUT
   - Server Action: destroy the session server-side (invalidate refresh token),
     clear cookies, `redirect('/login')`.
   - Clear the client cache (`queryClient.clear()`) so the next user does not see
     the previous user's cached data.

E. ROUTE PROTECTION
   - middleware.ts → redirect for {{PROTECTED}}.
   - Every protected page → `const session = await getSession(); if (!session) redirect('/login')`.
   - Every action → its own auth check (use the `withAuth` wrapper — OCP).

F. CLIENT-SIDE USER CONTEXT
   - The session USER (id, name, role, avatar — never the token) is passed from the
     server layout into a client Context provider for DI (low-frequency read).
   - ISP: pass only the fields the UI needs. Never the full user row.

═══ DELIVERABLES ═══
  middleware.ts
  lib/server/auth/session.ts
  lib/server/ports/auth-gateway.ts
  lib/server/adapters/{{PROVIDER}}-auth-gateway.ts
  lib/server/use-cases/login.ts
  actions/auth-actions.ts
  components/LoginForm.tsx
  providers/session-provider.tsx    (client Context — DI, user info only)
  tests: login success/failure, rate limit, protected action rejects anon caller,
         non-admin blocked from an admin action, logout clears the cache

═══ SECURITY CHECKLIST — answer each explicitly before coding ═══
  [ ] Where is the access token stored, and can any client JS read it?
  [ ] What happens if someone POSTs directly to a Server Action, bypassing the UI?
  [ ] Can a logged-in non-admin call an admin action by crafting the request?
  [ ] Does a failed login reveal whether the email exists?
  [ ] Is the refresh token rotated? What happens if an old one is replayed?
  [ ] Are auth errors logged with context but returned generically to the user?

Answer the checklist first. Then wait for my "go".

WHEN THE IMPLEMENTATION IS DONE (not before the gate above):
run typecheck, lint, and the tests, and paste the real output. If something fails, say so plainly
and show it. If you could not run them, say that explicitly. NEVER report "done" on code you have
not executed.
```
