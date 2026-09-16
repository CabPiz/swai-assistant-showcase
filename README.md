# SWAI Assistant — Technical Showcase

> ⚠️ **This repository is a public technical showcase.** The full source code is maintained in a private repository for proprietary reasons. This repo documents the architecture, engineering decisions, and stack used in the project.

[![License](https://img.shields.io/badge/license-Proprietary-red.svg)]()
[![Stack](https://img.shields.io/badge/stack-Next.js%2015%20%7C%20TypeScript%20%7C%20Tauri-blue)]()
[![Monorepo](https://img.shields.io/badge/monorepo-Turborepo-EF4444)]()
[![Quality](https://img.shields.io/badge/quality-SonarCloud-4E9BCD)]()

An AI-powered real-time assistant for Summoners War, delivered as a **web dashboard + native desktop overlay**. Built solo as a product of [Kairos Labs](https://github.com/CabPiz).

---

## Architecture Overview

```
swai-assistant/              ← Turborepo monorepo (pnpm workspaces)
├── apps/
│   ├── web/                 ← Next.js 15 dashboard (Vercel)
│   ├── proxy/               ← Hono cloud proxy (Fly.io)
│   └── desktop/             ← Tauri v2 desktop app (local proxy + RTA overlay)
└── packages/
    ├── types/               ← Shared TypeScript types + Zod schemas
    ├── core/                ← Domain logic (rune analysis engine)
    └── ui/                  ← Shared React component library
```

### Data Flow

```
Game Client
    │
    ▼
[Tauri Desktop App]  ←── local proxy (passive listener — never modifies traffic)
    │                         │
    │                         ▼
    │              [Hono Cloud Proxy — Fly.io]
    │                         │
    │                         ▼
    └──────────────► [Next.js Web App — Vercel]
                              │
                              ▼
                    [Vercel AI SDK + LLM routing]
                    ├── Free tier   → Gemini Flash
                    └── Pro/Guild   → Claude Haiku / Sonnet
```

The proxy is **strictly passive**: it reads traffic to extract game state for AI analysis. It never injects, modifies, or retransmits altered data. The player always acts manually inside the game client.

---

## Stack

| Layer | Technology | Why |
|---|---|---|
| Monorepo | **Turborepo + pnpm workspaces** | Shared packages, parallel builds, task caching |
| Web | **Next.js 15 + App Router** | RSC, server actions, streaming AI responses |
| Styling | **Tailwind CSS + shadcn/ui** | Consistent design system with accessible primitives |
| Language | **TypeScript** (strict mode, project references) | End-to-end type safety across all apps and packages |
| Validation | **Zod** | Runtime schema validation on every API boundary |
| Cloud Proxy | **Hono** on Fly.io | Lightweight, edge-ready, TypeScript-first |
| Desktop | **Tauri v2** | Rust-powered native app with Rust + WebView; lightweight binary |
| Database | **Supabase** (PostgreSQL + Auth + Realtime) | Managed Postgres, RLS, real-time subscriptions |
| AI SDK | **Vercel AI SDK** | Unified streaming interface across providers |
| Testing | **Jest + Testing Library + Playwright** | Unit, integration, and E2E coverage |
| Quality | **SonarCloud** | Coverage gates, security ratings, code smells |
| CI/CD | **GitHub Actions** | Lint → type-check → test → build → quality gate |
| Git hooks | **Husky + lint-staged** | Enforce standards pre-commit |

---

## Engineering Decisions

### Why Turborepo?
The project has three deployment targets (web SaaS, cloud proxy, native desktop) sharing types, UI components, and domain logic. Turborepo provides build caching and dependency-aware task orchestration across them without a heavy framework.

### Why Tauri instead of Electron?
- ~10× smaller binary footprint (Rust runtime vs. bundled Node + Chromium)
- Native OS security model; no bundled browser to update
- Rust backend handles the local proxy listener efficiently

### Why Hono for the cloud proxy?
Hono runs on any runtime (Node, Deno, Bun, edge workers) and has near-zero overhead. The proxy layer is stateless — it only validates, enriches, and forwards game state payloads. Hono's middleware chain makes this pipeline clean and testable.

### Why separate `packages/types`?
All Zod schemas and TypeScript interfaces live in a single package imported by web, proxy, and desktop alike. A schema change fails the type-check on every consumer at once, catching mismatches before runtime.

### Model routing by tier
Calling Claude Sonnet for every free user isn't economically viable. The routing layer selects the model based on the user's subscription:
- **Free** → Gemini Flash (cost-effective, fast)
- **Pro / Guild** → Claude Haiku → Claude Sonnet (by task complexity)

This keeps gross margin positive while still offering premium AI quality to paying users.

---

## Quality & Compliance

### Code Quality (SonarCloud)
Every pull request runs through a Quality Gate:
- Coverage threshold enforced
- Zero new bugs or vulnerabilities allowed to merge
- Security hotspots reviewed before merge

### Legal Compliance (EU AI Act — Article 6)
This product is classified as **Minimal Risk** under the EU AI Act (entertainment tool, no decisions affecting fundamental rights).

Key obligations implemented:
- **AI Disclosure**: every AI-generated output is marked with an `<AIDisclosureBadge />` component
- **Observability**: every LLM call is traced with provider, model, token count, and estimated cost via `tracedLLMCall()`
- **Human-in-the-loop**: the assistant suggests; the player always acts manually
- **Data minimization**: game state is processed in-memory and not stored beyond session scope

---

## CI Pipeline

```
push / PR
  │
  ├── lint (ESLint)
  ├── type-check (tsc --noEmit)
  ├── test (Jest — coverage report uploaded to SonarCloud)
  ├── build (Turborepo — web + proxy)
  └── quality gate (SonarCloud — blocks merge on failure)
```

Merge is squash-merged after CI green. Branch is deleted automatically.

---

## About Kairos Labs

Kairos Labs is an independent software studio focused on AI-powered tools. SWAI Assistant is one of several products under active development.

**Contact:** contact.kairoslabs@gmail.com  
**GitHub:** [github.com/CabPiz](https://github.com/CabPiz)
