# 🎯 Prompt Library

> 🇻🇳 Bản tiếng Việt: [`vi/README.md`](vi/README.md)

> Battle-tested prompts that operationalize the two skills in `.agents/skills/`.
> The skills tell the AI **what the rules are**. These prompts tell it **what to do, in what order, and when to stop and ask you**.

---

## 📚 Index

Every prompt ships with a **`.example.md`** twin: the same prompt with every `{{PLACEHOLDER}}` already filled in for a real scenario (**Acme Console**, a B2B team-admin dashboard). Copy-paste and run.

### 🎨 Frontend only

| # | Prompt | Example | Use it when | Principles |
|---|--------|---------|-------------|-----------|
| 01 | [Build a Reusable UI Component](fe/01-ui-component.md) | [📘](fe/01-ui-component.example.md) | Creating a design-system component used by 3+ features | OCP, LSP, ISP |
| 02 | [Build a Client-Side Feature Page](fe/02-feature-page.md) | [📘](fe/02-feature-page.example.md) | A full CSR screen: list + filters + pagination + states | SRP, ISP, DIP |
| 03 | [Build a Form](fe/03-form.md) | [📘](fe/03-form.example.md) | Any form — login, create/edit, wizard | SRP, DIP, LSP |
| 04 | [Refactor a Legacy Component to SOLID](fe/04-refactor-solid.md) | [📘](fe/04-refactor-solid.example.md) | A 300-line god-component you inherited | all 5 |
| 05 | [Performance Audit & Optimization](fe/05-performance.md) | [📘](fe/05-performance.example.md) | Janky typing, slow LCP, huge bundle, re-render storms | ISP |
| 06 | [Write Tests](fe/06-testing.md) | [📘](fe/06-testing.example.md) | Adding a test suite (Testing Trophy) | DIP, LSP |

### 🔌 Frontend + Backend (API integration)

| # | Prompt | Example | Use it when | Principles |
|---|--------|---------|-------------|-----------|
| 07 | [Define the API Contract](fullstack/07-api-contract.md) | [📘](fullstack/07-api-contract.example.md) | **Run this first.** Zod contract shared by FE + BE | ISP, DIP, LSP |
| 08 | [Integrate a REST API (Client-Side)](fullstack/08-csr-api-integration.md) | [📘](fullstack/08-csr-api-integration.example.md) | React Query / SWR against a REST backend | SRP, DIP, ISP, LSP |
| 09 | [Build an RSC Page with Server Data Fetching](fullstack/09-rsc-data-fetching.md) | [📘](fullstack/09-rsc-data-fetching.example.md) | App Router page fetching on the server, streaming | SRP, ISP, DIP |
| 10 | [Build CRUD with Server Actions](fullstack/10-server-action-crud.md) | [📘](fullstack/10-server-action-crud.example.md) | Full-stack mutations — ports, adapters, authz | all 5 |
| 11 | [End-to-End Authentication Flow](fullstack/11-auth-flow.md) | [📘](fullstack/11-auth-flow.example.md) | Login, session, refresh, logout, route protection | SRP, LSP, DIP |
| 12 | [Ship a Complete Full-Stack Feature Slice](fullstack/12-fullstack-slice.md) | [📘](fullstack/12-fullstack-slice.example.md) | **The master prompt.** Ticket → mergeable PR | all 5 |

### 🔍 Review & Debug

| # | Prompt | Example | Use it when | Principles |
|---|--------|---------|-------------|-----------|
| 13 | [Code Review Against the Skills](review/13-code-review.md) | [📘](review/13-code-review.example.md) | Reviewing a PR / branch / folder | all 5 |
| 14 | [Debug an FE/BE Integration Issue](review/14-debug-integration.md) | [📘](review/14-debug-integration.example.md) | "It works in Postman but not in the app" | LSP, DIP |

---

## 🚀 How to use

1. **Load the skill first.** Every prompt assumes the AI already has the ruleset:
   ```text
   Follow the rules in @.agents/skills/next-client-conversion-skills/SKILL.md
   (and @.agents/skills/nextjs-server-conversion-skills/SKILL.md for server work).
   ```
2. **Pick the prompt** from the index above — or its `.example.md` twin if you want it pre-filled.
3. **Replace every `{{PLACEHOLDER}}`** in the block. An unfilled placeholder is the #1 cause of a bad generation.
4. **Paste the block** into your agent, as-is. Each file is nothing but that block — no preamble to trim.
5. **Answer the gate.** Most prompts deliberately stop after the design phase and ask for your approval. Do not skip it — that pause is where 80% of the rework is prevented.

---

## 🧭 Typical flows

**New full-stack feature**
```text
07 (contract) → 12 (master slice, which orchestrates 09/10 + 02/03) → 13 (review)
```

**New CSR feature against an existing backend**
```text
07 (contract) → 08 (integrate) → 02 (page) → 03 (form) → 06 (tests) → 13 (review)
```

**Inherited legacy code**
```text
04 (refactor to SOLID) → 06 (tests) → 05 (performance) → 13 (review)
```

**Something is broken**
```text
14 (debug) → 06 (regression test) → 13 (review)
```

---

## 🧱 What every prompt enforces

Each prompt is a projection of the same core rules from the skills:

| Principle | On the client | On the server |
|---|---|---|
| **SRP** | Logic-in-hook, UI-in-component. `.tsx` = JSX only. | Thin actions: auth → validate → delegate → revalidate. |
| **OCP** | Compound components, `cva` variant maps. No boolean props. | Action wrappers (`withAuth`), strategy maps. No `if (provider === ...)`. |
| **LSP** | `forwardRef` + `ComponentPropsWithoutRef` + `{...rest}`. | One `ActionResult<T>`. Fake and real repos fail identically. |
| **ISP** | Narrow props. Store slice selectors. | `select` columns. Only primitives cross the RSC boundary. |
| **DIP** | Gateways injected via Context. `axios` in exactly one file. | Ports + adapters. One composition root. |

Plus, in every prompt: **Zod at every boundary** (`as T` is banned), **all four data states**, **a11y**, and **no `any`**.

---

## ✍️ Writing your own prompt

Every file here is the same minimal shape — frontmatter, a one-line "when to use", and **the prompt block, nothing else**:

```markdown
---
title: ...
category: FE | FE + BE | Review
skills: [...]
principles: [...]
---

# Prompt — <name>

> When to use it. When NOT to use it.

---

```text
(the whole prompt — constraints, deliverables, and the stop-gate)
```
```

No inputs table, no checklist, no commentary. The prompt IS the file. Anything else is context you pay for on every run without it changing what the agent does.

Three things make these prompts work — keep them:
- **Constraints, not wishes.** "Wrap in `forwardRef` and spread `...rest`" beats "make it reusable".
- **A stop gate.** Force the agent to show its design before it writes 400 lines you'll throw away.
- **Rules that shape decisions.** If a line only restates what a linter already enforces (`no any`, naming conventions), it's noise — cut it and let the skill file carry it.
