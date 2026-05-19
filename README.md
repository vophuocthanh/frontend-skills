# Frontend Engineering AI Skills (Agent Ruleset)

> The ultimate production-ready AI coding ruleset for modern React, scalable architecture, strict TypeScript, and performance-first patterns.

## 🌟 Overview
This repository provides a comprehensive, production-grade ruleset (Agent Skill) designed specifically for Frontend Engineers and AI Coding Assistants (like Cursor, Cline, GitHub Copilot, and Antigravity).

Instead of repeatedly prompting AI about file organization, state management, or performance optimization, you can simply load this skill into your project. The AI will automatically adopt a "Senior Frontend Engineer Mindset", ensuring:
- **Performance-first:** Eliminating network waterfalls (`Promise.all`), optimizing bundle sizes (no barrel files).
- **Clean Architecture:** Strict separation between Logic (Hooks) and UI (Components), colocating related state to prevent desync.
- **Scalability:** Feature-Sliced Design (Domain-driven) with strict Public/Private module boundaries.
- **Strict Typing:** Utilizing Zod for API Contracts & Forms, relying on Discriminated Unions instead of boolean flags.

## 🚀 Key Features (Included Rules)
1. **Boundary Architecture:** Explicit separation between Server and Client code (`server-only` / `client-only`).
2. **Logic-in-Hook:** Prohibits complex logic inside components; all business logic must reside in Custom Hooks.
3. **Colocate Related State:** Prevents scattering tightly-coupled state across multiple hooks to avoid UI desync.
4. **Composition over Configuration:** Encourages Compound Components over component monolithic blocks with dozens of boolean props.
5. **API & Data Layer:** Enforces Zod Schemas for API responses. Never trust network data.
6. **React Query / Caching:** Centralized Query Keys factory and a robust 3-layer caching strategy.
7. **DX & Git Conventions:** Standardized folder structures (`kebab-case`), file naming (`PascalCase` for UI, `kebab-case` for logic).

## 📦 Installation

You don't need to clone the entire repository or manually copy-paste files. You can use the official `skills` CLI to pull this ruleset directly into your project's `.agents/skills` folder.

Run the following command in the root directory of your target project:

```bash
npx skills add vophuocthanh/frontend-skills
```

*(This command will instantly download and install the frontend engineering skill from this repository).*

### 🛠 Quick Setup Alias (macOS/Linux Optional)
To save time, you can create a bash alias to install this ruleset into any project with a single word.
Open your `~/.zshrc` (or `~/.bashrc`) and add:

```bash
alias install-fe-skills="npx skills add vophuocthanh/frontend-skills && echo '✅ Frontend AI Skills installed successfully!'"
```
Run `source ~/.zshrc`. From now on, just type `install-fe-skills` in any new project.

## 🤖 Usage with AI Assistants

### 1. Cursor / Cline (Recommended)
The most powerful way to enforce these rules is to set them as your project's default system instructions:
- Run the installation command above.
- Copy the contents of the `SKILL.md` file (located in `.agents/skills/frontend-engineering-ruleset/`) and paste it into a `.cursorrules` (or `.clinerules`) file at the root of your project.
- The AI will automatically read and apply these standards on every prompt.

### 2. Antigravity / Other AI Agents
- After installation, the ruleset will be located at `.agents/skills/frontend-engineering-ruleset/`.
- When prompting the AI, simply say:
  > *"Please build feature X, and strictly follow the **frontend-engineering-ruleset**."*
- Or use file mentions (depending on the editor):
  > *"Create the XYZ component using the standards defined in `@[.agents/skills/frontend-engineering-ruleset/SKILL.md]`"*

---
*Created and maintained by [vophuocthanh](https://github.com/vophuocthanh).*
