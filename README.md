# OpenForge 🔥

Open source AI app builder. Describe your app, get production-ready Next.js + Supabase code.

**BYOK** — Bring your own API key (Claude, OpenAI, etc.)

## What is this?

You describe what you want → OpenForge generates a complete, exportable codebase → Deploy anywhere.

No vendor lock-in. The code is yours.

## Quick Start

```bash
git clone https://github.com/yourusername/openforge.git
cd openforge
pnpm install
cp .env.example .env.local
# Add your Supabase credentials to .env.local
pnpm dev
```

## How it works

1. Describe your app in plain language
2. OpenForge generates the schema, components, and API routes
3. Preview live in the browser
4. Export and deploy wherever you want

## Stack

- **Builder:** Next.js 14 + Tailwind + shadcn/ui + Turborepo
- **Generated apps:** Next.js + Supabase
- **AI:** Claude or OpenAI (BYOK)

## Documentation

See [CLAUDE.md](./CLAUDE.md) for full architecture and development guide.

## License

MIT