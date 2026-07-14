---
title: Build a Form (React Hook Form + Zod)
category: FE
skills: [react-client-mastery]
principles: [SRP, DIP, LSP]
---

# 📝 Prompt — Build a Form

> Use for any form: login, signup, create/edit entity, multi-step wizard, filter panel.
> Core rule: **uncontrolled by default** (RHF), one Zod schema as the single source of truth.

---

```text
Follow the `react-client-mastery` skill. Build `{{FORM}}`.

SPEC
- Fields: {{FIELDS}}
- Submit target: {{SUBMIT_TARGET}}
- On success: {{ON_SUCCESS}}

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

1. Schema first — ONE Zod schema is the single source of truth.
   - `export const {{FORM}}Schema = z.object({...})`
   - `export type {{FORM}}Values = z.infer<typeof {{FORM}}Schema>` — never hand-write the type.
   - Reuse this exact schema on the server when the target is a Server Action.

2. Uncontrolled by default:
   - `useForm({ resolver: zodResolver({{FORM}}Schema), defaultValues })` + `register`.
   - Use `watch`/`useWatch` ONLY where a live reaction is genuinely required
     (character counter, dependent field, live preview) — scope it to the smallest subtree.
   - NEVER `useState` per field with manual onChange. That re-renders on every keystroke.

3. SRP — split the three concerns:
   - `{{FORM}}.schema.ts`  → validation contract
   - `use-{{FORM}}.ts`     → form instance, submit handler, mutation, side effects
   - `{{FORM}}.tsx`        → JSX only; it calls the hook and renders fields

4. DIP — the hook depends on an injected API/action function, not on `axios`/`fetch`
   inline. This lets tests run the form with a fake submit and no network.

5. LSP — field wrappers (`<TextField>`, `<Select>`) MUST forward `ref` and spread
   `...rest` so `register()` works. A wrapper that swallows `ref` silently breaks RHF.
   Use `forwardRef` + `ComponentPropsWithoutRef<'input'>`.

6. Submit state must be a discriminated union or RHF's own flags — never
   `{ isLoading, error?, success? }` booleans that allow impossible states.

7. Error handling (three layers, all required):
   - Field-level: `formState.errors.<field>` rendered next to the input,
     linked via `aria-describedby` + `aria-invalid`.
   - Server-side field errors: map them back with `setError(field, ...)`.
   - Form-level: a `role="alert"` region for non-field errors (409, 500, network).

8. A11y:
   - Every input has a real `<label htmlFor>` — placeholder is NOT a label.
   - `aria-invalid` + `aria-describedby` on errored fields.
   - Focus the first invalid field on failed submit.
   - Disable the submit button while `isSubmitting`, and announce success politely
     (`aria-live="polite"`).

9. Never store raw file objects or tokens in global state. No `any`.

DELIVERABLES
  src/features/{{FEATURE}}/
  ├── schemas/{{FORM}}-schema.ts
  ├── hooks/use-{{FORM}}.ts
  └── components/{{FORM}}.tsx

TESTS (RTL + userEvent, mock at the network boundary with msw)
- submits valid data and calls the API once with the parsed payload
- shows field errors for invalid input and does NOT call the API
- renders a server error (e.g. 409 email taken) in the alert region
- disables submit while pending

Print the Zod schema first and ask me to confirm the validation rules before
writing the rest.

WHEN THE IMPLEMENTATION IS DONE (not before the gate above):
run typecheck, lint, and the tests, and paste the real output. If something fails, say so plainly
and show it. If you could not run them, say that explicitly. NEVER report "done" on code you have
not executed.
```
