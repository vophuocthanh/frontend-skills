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

## 📂 Repository Structure

```text
.agents/skills/
├── react-client-conversion-skills/    # Client-Side Rendering
│   └── SKILL.md
└── nextjs-server-conversion-skills/   # Server-Side Rendering
    └── SKILL.md
```

---
*Created and maintained by [vophuocthanh](https://github.com/vophuocthanh).*
