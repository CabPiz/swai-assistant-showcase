# SWAI Assistant — Vitrine Técnica

🌐 [English](./README.md) · **Português** · [Español](./README.es.md)

> ⚠️ **Este repositório é uma vitrine técnica pública.** O código-fonte completo é mantido em um repositório privado por razões proprietárias. Este repo documenta a arquitetura, as decisões de engenharia e o stack utilizado no projeto.

[![Licença](https://img.shields.io/badge/licença-Proprietária-red.svg)]()
[![Stack](https://img.shields.io/badge/stack-Next.js%2015%20%7C%20TypeScript%20%7C%20Tauri-blue)]()
[![Monorepo](https://img.shields.io/badge/monorepo-Turborepo-EF4444)]()
[![Qualidade](https://img.shields.io/badge/qualidade-SonarCloud-4E9BCD)]()

Um assistente de IA em tempo real para Summoners War, entregue como **dashboard web + overlay desktop nativo**. Desenvolvido de forma independente como produto da [Kairos Labs](https://github.com/CabPiz).

---

## Visão Geral da Arquitetura

```
swai-assistant/              ← Monorepo Turborepo (pnpm workspaces)
├── apps/
│   ├── web/                 ← Dashboard Next.js 15 (Vercel)
│   ├── proxy/               ← Cloud proxy Hono (Fly.io)
│   └── desktop/             ← App desktop Tauri v2 (proxy local + overlay RTA)
└── packages/
    ├── types/               ← Types TypeScript compartilhados + schemas Zod
    ├── core/                ← Lógica de domínio (motor de análise de runas)
    └── ui/                  ← Biblioteca de componentes React compartilhados
```

### Fluxo de Dados

```
Cliente do Jogo
    │
    ▼
[App Desktop Tauri]  ←── proxy local (listener passivo — nunca modifica o tráfego)
    │                         │
    │                         ▼
    │              [Cloud Proxy Hono — Fly.io]
    │                         │
    │                         ▼
    └──────────────► [App Web Next.js — Vercel]
                              │
                              ▼
                    [Vercel AI SDK + roteamento de LLM]
                    ├── Tier Free   → Gemini Flash
                    └── Pro/Guild   → Claude Haiku / Sonnet
```

O proxy é **estritamente passivo**: lê o tráfego para extrair o estado do jogo para análise de IA. Nunca injeta, modifica ou retransmite dados alterados. O jogador sempre age manualmente dentro do cliente do jogo.

---

## Stack

| Camada | Tecnologia | Por quê |
|---|---|---|
| Monorepo | **Turborepo + pnpm workspaces** | Pacotes compartilhados, builds paralelos, cache de tarefas |
| Web | **Next.js 15 + App Router** | RSC, server actions, streaming de respostas de IA |
| Estilização | **Tailwind CSS + shadcn/ui** | Design system consistente com primitivos acessíveis |
| Linguagem | **TypeScript** (strict mode, project references) | Type safety ponta a ponta em todos os apps e pacotes |
| Validação | **Zod** | Validação de schema em tempo de execução em cada fronteira de API |
| Cloud Proxy | **Hono** no Fly.io | Leve, pronto para edge, TypeScript-first |
| Desktop | **Tauri v2** | App nativo com Rust + WebView; binário leve |
| Banco de Dados | **Supabase** (PostgreSQL + Auth + Realtime) | Postgres gerenciado, RLS, subscriptions em tempo real |
| AI SDK | **Vercel AI SDK** | Interface de streaming unificada entre providers |
| Testes | **Jest + Testing Library + Playwright** | Cobertura unitária, de integração e E2E |
| Qualidade | **SonarCloud** | Gates de cobertura, ratings de segurança, code smells |
| CI/CD | **GitHub Actions** | Lint → type-check → test → build → quality gate |
| Git hooks | **Husky + lint-staged** | Garantia de padrões no pre-commit |

---

## Decisões de Engenharia

### Por que Turborepo?
O projeto tem três targets de deploy (SaaS web, cloud proxy, desktop nativo) compartilhando types, componentes de UI e lógica de domínio. O Turborepo fornece cache de build e orquestração de tarefas com consciência de dependências entre eles, sem um framework pesado.

### Por que Tauri em vez de Electron?
- Binário ~10× menor (runtime Rust vs. Node + Chromium empacotados)
- Modelo de segurança nativo do SO; sem browser empacotado para atualizar
- Backend Rust lida com o listener do proxy local de forma eficiente

### Por que Hono para o cloud proxy?
O Hono roda em qualquer runtime (Node, Deno, Bun, edge workers) e tem overhead praticamente zero. A camada de proxy é stateless — apenas valida, enriquece e encaminha os payloads de estado do jogo. O middleware chain do Hono torna esse pipeline limpo e testável.

### Por que um `packages/types` separado?
Todos os schemas Zod e interfaces TypeScript vivem em um único pacote importado por web, proxy e desktop igualmente. Uma mudança de schema falha o type-check em todos os consumidores de uma vez, detectando incompatibilidades antes do runtime.

### Roteamento de modelo por tier
Chamar Claude Sonnet para cada usuário free não é economicamente viável. A camada de roteamento seleciona o modelo com base na assinatura do usuário:
- **Free** → Gemini Flash (custo-efetivo, rápido)
- **Pro / Guild** → Claude Haiku → Claude Sonnet (por complexidade da tarefa)

Isso mantém a margem bruta positiva enquanto ainda oferece qualidade premium de IA para usuários pagantes.

---

## Qualidade e Conformidade

### Qualidade de Código (SonarCloud)
Cada pull request passa por um Quality Gate:
- Threshold de cobertura obrigatório
- Zero novos bugs ou vulnerabilidades permitidos para merge
- Security hotspots revisados antes do merge

### Conformidade Legal (EU AI Act — Artigo 6)
Este produto é classificado como **Risco Mínimo** sob o EU AI Act (ferramenta de entretenimento, sem decisões que afetam direitos fundamentais).

Obrigações implementadas:
- **Disclosure de IA**: todo output gerado por IA é marcado com o componente `<AIDisclosureBadge />`
- **Observabilidade**: toda chamada de LLM é rastreada com provider, modelo, contagem de tokens e custo estimado via `tracedLLMCall()`
- **Human-in-the-loop**: o assistente sugere; o jogador sempre age manualmente
- **Minimização de dados**: o estado do jogo é processado em memória e não armazenado além do escopo da sessão

---

## Pipeline de CI

```
push / PR
  │
  ├── lint (ESLint)
  ├── type-check (tsc --noEmit)
  ├── test (Jest — relatório de cobertura enviado ao SonarCloud)
  ├── build (Turborepo — web + proxy)
  └── quality gate (SonarCloud — bloqueia merge em caso de falha)
```

O merge é feito via squash após CI verde. A branch é deletada automaticamente.

---

## Sobre a Kairos Labs

A Kairos Labs é um estúdio de software independente focado em ferramentas com IA. O SWAI Assistant é um dos vários produtos em desenvolvimento ativo.

**Contato:** contact.kairoslabs@gmail.com  
**GitHub:** [github.com/CabPiz](https://github.com/CabPiz)
