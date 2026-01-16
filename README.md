# Agent Skills Collection

A curated collection of skills for AI coding agents, covering modern web development patterns with Next.js, React 19, Supabase, and more.

## 📦 Available Skills

| Skill | Description |
|-------|-------------|
| [`framer-motion`](./framer-motion/) | Animation patterns and micro-interactions with Framer Motion |
| [`react-19-patterns`](./react-19-patterns/) | New React 19 APIs: `use()`, Server Components, Actions, Compiler |
| [`supabase-auth`](./supabase-auth/) | Authentication flows and session management with Supabase |
| [`api-rate-limiting`](./api-rate-limiting/) | Retry logic, circuit breakers, and request deduplication |
| [`seo-next`](./seo-next/) | SEO patterns for Next.js App Router |
| [`typescript-patterns`](./typescript-patterns/) | Advanced TypeScript: generics, utility types, type guards |
| [`error-handling`](./error-handling/) | Error boundaries, API errors, logging, and recovery patterns |
| [`tailwind-v4`](./tailwind-v4/) | Tailwind CSS v4 features: @theme, container queries, utilities |
| [`git-conventional-commits`](./git-conventional-commits/) | Standardized commit messages and semantic versioning |
| [`zustand-patterns`](./zustand-patterns/) | State management with Zustand 5: slices, persist, devtools |
| [`next-cache-patterns`](./next-cache-patterns/) | Next.js caching: cacheLife, PPR, revalidation strategies |
| [`testing-patterns`](./testing-patterns/) | Testing with Vitest, React Testing Library, and Playwright |
| [`vercel-ai-sdk`](./vercel-ai-sdk/) | Streaming LLM responses, tool calling, and structured outputs |
| [`mcp-server-dev`](./mcp-server-dev/) | Building Model Context Protocol servers: tools, resources, prompts |
| [`pwa-next`](./pwa-next/) | Service workers, offline support, and installability for Next.js |
| [`database-migrations`](./database-migrations/) | Migration patterns for Supabase, Prisma, and Drizzle ORM |
| [`claude-code-plugins`](./claude-code-plugins/) | Building Claude Code plugins: commands, agents, hooks, marketplaces |

## 🚀 Installation

Copy any skill folder to your project's `.agent/skills/` directory:

```bash
cp -r framer-motion /path/to/your/project/.agent/skills/
```

Or clone the entire repository:

```bash
git clone https://github.com/yourusername/agent-skills.git
cp -r agent-skills/* /path/to/your/project/.agent/skills/
```

## 📝 Skill Format

Each skill follows the standard format with a `SKILL.md` file containing:

```yaml
---
name: skill-name
description: Brief description of when to use this skill
---

# Skill Title

Detailed patterns, code examples, and best practices.
```

## 🛠️ Tech Stack Coverage

- **Frontend**: React 19, Next.js 16, Framer Motion
- **Backend**: Supabase, Server Actions
- **Styling**: Tailwind CSS v4
- **State**: Zustand 5
- **Language**: TypeScript 5.9+

## 📄 License

MIT License - see [LICENSE](./LICENSE) for details.

## 🤝 Contributing

Contributions welcome! Please follow the existing skill format and include practical code examples.
