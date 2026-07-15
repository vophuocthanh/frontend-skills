---
name: frontend-prompt-library
description: 14 executable, copy-paste prompts that operationalize the frontend SOLID rulesets — reusable UI components, feature pages, forms, refactors, performance, testing, API contracts, RSC/Server Actions, auth, code review, and debugging. Each prompt ships with a filled-in example. English + Vietnamese.
---

# Frontend Prompt Library

> The rules live in `next-client-conversion-skills` and `nextjs-server-conversion-skills`.
> This skill turns those rules into **executable tasks**: 14 detailed prompts with `{{PLACEHOLDER}}` slots, stop-gates, and a Definition of Done. Every prompt has a `.example.md` twin with every placeholder pre-filled for a real scenario.

## How to use

1. **Load a ruleset skill first** so the agent has the rules:
   ```text
   Follow the rules in @.agents/skills/next-client-conversion-skills/SKILL.md
   (and @.agents/skills/nextjs-server-conversion-skills/SKILL.md for server work).
   ```
2. **Open the full index** → [`README.md`](README.md) (🇻🇳 [`vi/README.md`](vi/README.md)) for the table, typical flows, and how-to.
3. **Pick a prompt**, replace every `{{PLACEHOLDER}}`, paste the block as-is, and answer the stop-gate.

## Index

### 🎨 Frontend only — `fe/`
- 01 [Build a Reusable UI Component](fe/01-ui-component.md) · [example](fe/01-ui-component.example.md)
- 02 [Build a Client-Side Feature Page](fe/02-feature-page.md) · [example](fe/02-feature-page.example.md)
- 03 [Build a Form](fe/03-form.md) · [example](fe/03-form.example.md)
- 04 [Refactor a Legacy Component to SOLID](fe/04-refactor-solid.md) · [example](fe/04-refactor-solid.example.md)
- 05 [Performance Audit & Optimization](fe/05-performance.md) · [example](fe/05-performance.example.md)
- 06 [Write Tests](fe/06-testing.md) · [example](fe/06-testing.example.md)

### 🔌 Frontend + Backend — `fullstack/`
- 07 [Define the API Contract](fullstack/07-api-contract.md) · [example](fullstack/07-api-contract.example.md)
- 08 [Integrate a REST API (Client-Side)](fullstack/08-csr-api-integration.md) · [example](fullstack/08-csr-api-integration.example.md)
- 09 [Build an RSC Page with Server Data Fetching](fullstack/09-rsc-data-fetching.md) · [example](fullstack/09-rsc-data-fetching.example.md)
- 10 [Build CRUD with Server Actions](fullstack/10-server-action-crud.md) · [example](fullstack/10-server-action-crud.example.md)
- 11 [End-to-End Authentication Flow](fullstack/11-auth-flow.md) · [example](fullstack/11-auth-flow.example.md)
- 12 [Ship a Complete Full-Stack Feature Slice](fullstack/12-fullstack-slice.md) · [example](fullstack/12-fullstack-slice.example.md)

### 🔍 Review & Debug — `review/`
- 13 [Code Review Against the Skills](review/13-code-review.md) · [example](review/13-code-review.example.md)
- 14 [Debug an FE/BE Integration Issue](review/14-debug-integration.md) · [example](review/14-debug-integration.example.md)

## Typical flows

- **New full-stack feature:** 07 → 12 → 13
- **CSR feature against existing backend:** 07 → 08 → 02 → 03 → 06 → 13
- **Inherited legacy code:** 04 → 06 → 05 → 13
- **Something is broken:** 14 → 06 → 13

🇻🇳 Bản tiếng Việt đầy đủ của cả 14 prompt: [`vi/`](vi/README.md).
