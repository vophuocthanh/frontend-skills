---
title: Example — Code Review Against the Skills
type: example
pairs_with: 13-code-review.md
---

# 📘 Example — Code Review Against the Skills

> A fully filled-in run of [`13-code-review.md`](13-code-review.md) on a real scenario.
> Copy the prompt block below and paste it into your agent as-is.

---

```text
Review `git diff develop...HEAD` (PR #142 — `feat(team): invite + revoke teammates`) against the `react-client-mastery` and `nextjs-server-mastery` skills.
CONTEXT: the Team Invitations feature — Server Actions + the RSC `/team` page + the client invitation list; it touches authorization and the RSC boundary, and it is the first feature in this codebase to use Server Actions
DEPTH: full audit — blocking issues first

REPO CONTEXT: PR #142 is the FIRST feature in this repo to use Server Actions — every other
  feature still uses REST + TanStack Query through `src/lib/http-client.ts`. The conventions
  it must be judged against are the ones the billing feature established: ports/adapters +
  `container.ts` under `lib/server/`, the `ActionResult<T>` contract in
  `src/shared/api/action-result.ts`, typed errors in `src/shared/api/errors.ts`, and the
  `src/features/*` api/hooks/components layering.

NON-GOALS: Do NOT fix anything — report only. Do NOT review the unrelated billing refactor
  that happens to sit in the same branch.

0. MATCH THE REPO BEFORE YOU MATCH THIS PROMPT.
   If REPO CONTEXT shows an existing convention — folder layout, an http client, an error type,
   UI primitives, a test setup — REUSE IT instead of creating a parallel structure beside it.
   Creating a second way to do something that already exists is a failure, even if the new way
   follows every rule below. Judge the code against the conventions this repo actually uses,
   not against a greenfield ideal.

Review as the engineer who will be on call for this code. Be specific and
falsifiable: every finding needs a file:line, a concrete failure scenario
("if the user does X while offline, Y happens"), and a fix. No vague advice
like "consider improving readability" — if you cannot describe how it breaks,
it is not a finding.

═══ SEVERITY — report in this order ═══

🔴 BLOCKER (security / data loss / broken contract)
  - Server Action without an auth check inside it (middleware ≠ auth)
  - Authz by role only, missing an OWNERSHIP check (user can mutate another's data)
  - Unvalidated input (`formData.get(x) as string`) → privilege escalation
  - Whole DB row / secret crossing the RSC boundary → leaked in view-source (ISP)
  - Token in localStorage
  - `dangerouslySetInnerHTML` on unsanitized input
  - Unvalidated API response (`as User`) that can corrupt state downstream
  - Missing `revalidatePath` after a mutation → user sees stale data and retries
  - Secret exposed via `NEXT_PUBLIC_*`

🟠 MAJOR (bug or a design flaw that will hurt within a sprint)
  - SRP: a file with 2+ reasons to change (fetch + logic + JSX, or auth + SQL + email)
  - DIP: `axios`/`db`/`stripe`/`localStorage` imported directly in UI/pages/actions
  - LSP: a wrapper swallowing `ref`/`disabled`/`type`/`aria-*`;
         actions with inconsistent return contracts; a fake repo failing differently
         from the real one
  - OCP: `if (variant === ...)` / `if (provider === ...)` chains in shared code
  - `useEffect` used to derive/sync state
  - Optimistic update with no rollback in `onError`
  - Network waterfall: sequential `await` on independent calls
  - Missing error/empty/loading state
  - `'use client'` on a page/layout, dragging the tree into the bundle
  - Library barrel import killing tree-shaking

🟡 MINOR (quality — fix now, cheap)
  - Boolean prop explosion; god-object props; whole-store subscription (no selector)
  - Missing `staleTime`/`gcTime`; string-literal query keys
  - Missing debounce/throttle; scroll listener without `{ passive: true }`
  - `console.log` instead of structured logging
  - Naming convention violations; `any`; missing a11y attributes
  - Hardcoded user-visible strings (i18n)

═══ REQUIRED OUTPUT FORMAT ═══

1) VERDICT: APPROVE / APPROVE WITH NITS / REQUEST CHANGES — in one sentence.

2) FINDINGS TABLE, sorted by severity:
   | # | Sev | Principle | file:line | What breaks (concrete scenario) | Fix |

3) THE SOLID PASS — answer each explicitly, with evidence:
   - SRP: name the file with the most reasons to change. List them.
   - OCP: what happens when the next variant/provider is added? Which file must be reopened?
   - LSP: do all wrappers forward ref/rest? Do all actions/hooks share one return contract?
   - ISP: exactly which fields cross the RSC boundary / get passed as props? Any god-objects?
   - DIP: list every direct import of a concrete dependency (axios/db/SDK) from a
     high-level module.

4) WHAT'S GOOD — name 2-3 things done right. Reviews that only criticize get ignored.

5) TESTS: which findings would have been caught by a test that doesn't exist?
   Name the missing test.

Do NOT rewrite the code. Report findings. I will decide what to fix.

BEFORE YOU REPORT: re-open each file you cite and confirm the line still says what you claim.
A finding with a wrong file:line is worse than no finding — it burns the reader's trust.
If you could not verify a claim, mark it UNVERIFIED rather than dropping or asserting it.
```
