# Frontend Engineering AI Skills (Agent Ruleset)

> The ultimate production-ready AI coding ruleset for modern React, scalable architecture, strict TypeScript, and performance-first patterns.

## 🌟 Overview
This repository provides production-grade rulesets (Agent Skills) designed for Frontend Engineers and AI Coding Assistants (Cursor, Cline, GitHub Copilot, Antigravity).

Load a skill into your project and the AI will automatically adopt a "Senior Frontend Engineer Mindset".

## 📦 Available Skills

| Skill | Use Case | Size |
|-------|----------|------|
| `react-client-conversion-skills` | **Client-Side** — React hooks, state management, forms, styling, testing, a11y, performance | ~500 lines |
| `nextjs-server-conversion-skills` | **Server-Side** — RSC, Server Actions, boundary architecture, caching, env management, security | ~400 lines |
| `frontend-prompt-library` | **Executable tasks** — 14 copy-paste prompts (+ filled examples, 🇻🇳 VN) that operationalize the two rulesets | 14 prompts |

### Which one should I use?

```text
Client-side only (Vite, CRA, Next.js Pages Router)?
  → Install react-client-conversion-skills

Next.js App Router with RSC + Server Actions?
  → Install both react-client-conversion-skills AND nextjs-server-conversion-skills
```

## 🚀 Installation

```bash
npx skills add vophuocthanh/frontend-skills
```

### Install a specific skill only

```bash
# Client-side only
npx skills add vophuocthanh/frontend-skills --skill react-client-conversion-skills

# Server-side only
npx skills add vophuocthanh/frontend-skills --skill nextjs-server-conversion-skills

# Prompt library only (14 executable prompts + examples)
npx skills add vophuocthanh/frontend-skills --skill frontend-prompt-library
```

### 🛠 Quick Setup Alias (macOS/Linux)

```bash
alias install-fe-skills="npx skills add vophuocthanh/frontend-skills && echo '✅ Frontend AI Skills installed!'"
```

## 🤖 Usage with AI Assistants

### Cursor / Cline
- Copy the `SKILL.md` content into `.cursorrules` (or `.clinerules`) at your project root.

### Antigravity / Other AI Agents
- Say: *"Follow the **react-client-conversion-skills** rules."*
- Or mention: `@[.agents/skills/react-client-conversion-skills/SKILL.md]`

## 🎯 Prompt Library

The skills define **the rules**. The `frontend-prompt-library` skill turns them into **executable tasks** — 14 detailed, copy-paste prompts with placeholders, stop-gates, and a Definition of Done.

| Group | Prompts |
|---|---|
| **Frontend** | UI component · feature page · form · SOLID refactor · performance audit · testing |
| **Frontend + Backend** | API contract · REST integration · RSC data fetching · Server Action CRUD · auth flow · full-stack slice |
| **Review** | code review · debug FE/BE integration |

Start at [`frontend-prompt-library/README.md`](.agents/skills/frontend-prompt-library/README.md) for the index and the typical flows.
🇻🇳 Bản tiếng Việt đầy đủ: [`frontend-prompt-library/vi/README.md`](.agents/skills/frontend-prompt-library/vi/README.md)

## 📂 Repository Structure

```text
.agents/skills/
├── next-client-conversion-skills/     # Client-Side Rendering + SOLID
│   └── SKILL.md
├── nextjs-server-conversion-skills/   # Server-Side Rendering + SOLID
│   └── SKILL.md
└── frontend-prompt-library/           # 14 executable prompts (travels with the skill)
    ├── SKILL.md                       # skill entry + index
    ├── README.md                      # full index + how to use + typical flows
    ├── fe/                            # 01-06  Frontend tasks
    ├── fullstack/                     # 07-12  FE + BE (API integration)
    ├── review/                        # 13-14  Review & debug
    └── vi/                            # 🇻🇳 Vietnamese translation (same 14 prompts)
        ├── README.md
        ├── fe/
        ├── fullstack/
        └── review/
```

---
*Created and maintained by [vophuocthanh](https://github.com/vophuocthanh).*
