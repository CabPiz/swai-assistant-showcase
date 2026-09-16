# SWAI Assistant — Vitrina Técnica

🌐 [English](./README.md) · [Português](./README.pt.md) · **Español**

> ⚠️ **Este repositorio es una vitrina técnica pública.** El código fuente completo se mantiene en un repositorio privado por razones propietarias. Este repo documenta la arquitectura, las decisiones de ingeniería y el stack utilizado en el proyecto.

[![Licencia](https://img.shields.io/badge/licencia-Propietaria-red.svg)]()
[![Stack](https://img.shields.io/badge/stack-Next.js%2015%20%7C%20TypeScript%20%7C%20Tauri-blue)]()
[![Monorepo](https://img.shields.io/badge/monorepo-Turborepo-EF4444)]()
[![Calidad](https://img.shields.io/badge/calidad-SonarCloud-4E9BCD)]()

Un asistente de IA en tiempo real para Summoners War, entregado como **dashboard web + overlay de escritorio nativo**. Desarrollado de forma independiente como producto de [Kairos Labs](https://github.com/CabPiz).

---

## Visión General de la Arquitectura

```
swai-assistant/              ← Monorepo Turborepo (pnpm workspaces)
├── apps/
│   ├── web/                 ← Dashboard Next.js 15 (Vercel)
│   ├── proxy/               ← Cloud proxy Hono (Fly.io)
│   └── desktop/             ← App de escritorio Tauri v2 (proxy local + overlay RTA)
└── packages/
    ├── types/               ← Types TypeScript compartidos + schemas Zod
    ├── core/                ← Lógica de dominio (motor de análisis de runas)
    └── ui/                  ← Biblioteca de componentes React compartidos
```

### Flujo de Datos

```
Cliente del Juego
    │
    ▼
[App Escritorio Tauri]  ←── proxy local (listener pasivo — nunca modifica el tráfico)
    │                         │
    │                         ▼
    │              [Cloud Proxy Hono — Fly.io]
    │                         │
    │                         ▼
    └──────────────► [App Web Next.js — Vercel]
                              │
                              ▼
                    [Vercel AI SDK + enrutamiento de LLM]
                    ├── Tier Free   → Gemini Flash
                    └── Pro/Guild   → Claude Haiku / Sonnet
```

El proxy es **estrictamente pasivo**: lee el tráfico para extraer el estado del juego para análisis de IA. Nunca inyecta, modifica ni retransmite datos alterados. El jugador siempre actúa manualmente dentro del cliente del juego.

---

## Stack

| Capa | Tecnología | Por qué |
|---|---|---|
| Monorepo | **Turborepo + pnpm workspaces** | Paquetes compartidos, builds en paralelo, caché de tareas |
| Web | **Next.js 15 + App Router** | RSC, server actions, streaming de respuestas de IA |
| Estilos | **Tailwind CSS + shadcn/ui** | Sistema de diseño consistente con primitivos accesibles |
| Lenguaje | **TypeScript** (strict mode, project references) | Type safety de extremo a extremo en todos los apps y paquetes |
| Validación | **Zod** | Validación de schema en tiempo de ejecución en cada frontera de API |
| Cloud Proxy | **Hono** en Fly.io | Ligero, listo para edge, TypeScript-first |
| Escritorio | **Tauri v2** | App nativa con Rust + WebView; binario liviano |
| Base de Datos | **Supabase** (PostgreSQL + Auth + Realtime) | Postgres administrado, RLS, suscripciones en tiempo real |
| AI SDK | **Vercel AI SDK** | Interfaz de streaming unificada entre proveedores |
| Pruebas | **Jest + Testing Library + Playwright** | Cobertura unitaria, de integración y E2E |
| Calidad | **SonarCloud** | Gates de cobertura, ratings de seguridad, code smells |
| CI/CD | **GitHub Actions** | Lint → type-check → test → build → quality gate |
| Git hooks | **Husky + lint-staged** | Garantía de estándares en el pre-commit |

---

## Decisiones de Ingeniería

### ¿Por qué Turborepo?
El proyecto tiene tres targets de despliegue (SaaS web, cloud proxy, escritorio nativo) compartiendo types, componentes de UI y lógica de dominio. Turborepo proporciona caché de build y orquestación de tareas con consciencia de dependencias entre ellos, sin un framework pesado.

### ¿Por qué Tauri en lugar de Electron?
- Binario ~10× más pequeño (runtime Rust vs. Node + Chromium empaquetados)
- Modelo de seguridad nativo del SO; sin browser empaquetado que actualizar
- El backend en Rust maneja el listener del proxy local de forma eficiente

### ¿Por qué Hono para el cloud proxy?
Hono corre en cualquier runtime (Node, Deno, Bun, edge workers) y tiene overhead prácticamente nulo. La capa de proxy es stateless — solo valida, enriquece y reenvía los payloads de estado del juego. El middleware chain de Hono hace que este pipeline sea limpio y testeable.

### ¿Por qué un `packages/types` separado?
Todos los schemas Zod e interfaces TypeScript viven en un único paquete importado por web, proxy y desktop por igual. Un cambio de schema falla el type-check en todos los consumidores a la vez, detectando incompatibilidades antes del runtime.

### Enrutamiento de modelo por tier
Llamar a Claude Sonnet para cada usuario free no es económicamente viable. La capa de enrutamiento selecciona el modelo según la suscripción del usuario:
- **Free** → Gemini Flash (costo-efectivo, rápido)
- **Pro / Guild** → Claude Haiku → Claude Sonnet (según complejidad de la tarea)

Esto mantiene el margen bruto positivo mientras ofrece calidad premium de IA a los usuarios de pago.

---

## Calidad y Cumplimiento

### Calidad de Código (SonarCloud)
Cada pull request pasa por un Quality Gate:
- Umbral de cobertura obligatorio
- Cero nuevos bugs o vulnerabilidades permitidos para merge
- Security hotspots revisados antes del merge

### Cumplimiento Legal (EU AI Act — Artículo 6)
Este producto está clasificado como **Riesgo Mínimo** bajo el EU AI Act (herramienta de entretenimiento, sin decisiones que afecten derechos fundamentales).

Obligaciones implementadas:
- **Disclosure de IA**: todo output generado por IA está marcado con el componente `<AIDisclosureBadge />`
- **Observabilidad**: toda llamada de LLM es trazada con proveedor, modelo, conteo de tokens y costo estimado vía `tracedLLMCall()`
- **Human-in-the-loop**: el asistente sugiere; el jugador siempre actúa manualmente
- **Minimización de datos**: el estado del juego se procesa en memoria y no se almacena más allá del alcance de la sesión

---

## Pipeline de CI

```
push / PR
  │
  ├── lint (ESLint)
  ├── type-check (tsc --noEmit)
  ├── test (Jest — reporte de cobertura enviado a SonarCloud)
  ├── build (Turborepo — web + proxy)
  └── quality gate (SonarCloud — bloquea merge en caso de fallo)
```

El merge se realiza vía squash tras CI verde. La branch se elimina automáticamente.

---

## Sobre Kairos Labs

Kairos Labs es un estudio de software independiente enfocado en herramientas con IA. SWAI Assistant es uno de varios productos en desarrollo activo.

**Contacto:** contact.kairoslabs@gmail.com  
**GitHub:** [github.com/CabPiz](https://github.com/CabPiz)
