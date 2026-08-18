# Vibecoding

Plantilla del **curso Vibe Code con Change and Code × Startup Chihuahua** (9 semanas). Hecha por Pedro Gutiérrez (Roni).

Construido para founders mayormente no técnicos con 4 horas semanales: arrancas con una plantilla lista y la extiendes semana a semana hasta llegar a Demo Day con un producto AI-native funcional.

## Qué instalar (una vez que tienes el repo)

Antes de correr nada, ten esto listo. Guía de instalación paso a paso: `/docs/setup/prepara-tu-compu`. Lista de cuentas con sus enlaces: `/docs/setup/cuentas`.

**En tu computadora:**

- **Node 20 o 22 LTS** ([nodejs.org](https://nodejs.org)) — usa una versión par. El repo trae `.nvmrc`: con nvm corre `nvm use`.
- **yarn 1.x** — `npm install -g yarn`
- **Git** y **[Cursor](https://cursor.com)** (el editor con IA).

**Cuentas (todas con tier gratuito):**

- [GitHub](https://github.com), [Supabase](https://supabase.com) y [Vercel](https://vercel.com) — créalas hoy.
- [OpenAI](https://platform.openai.com) (Sem 3+) y [Resend](https://resend.com) (Sem 1+) — opcionales al inicio.

## Quick start (30 min, día de Sem 1)

```bash
# 1. Forkea con "Use this template" en GitHub y clona tu repo
git clone https://github.com/<tu-usuario>/<tu-producto>.git
cd <tu-producto>

# 2. Instala dependencias (yarn workspaces resuelve todo)
yarn install

# 3. Copia variables de entorno y rellena
cp web/.env.example web/.env.local

# 4. Arranca
yarn dev
```

Abre `http://localhost:3000` — verás tu landing.
Abre `http://localhost:3000/docs` — verás la documentación completa, mapeada semana a semana.

> Primera vez: sigue la checklist en `/docs/setup/prepara-tu-compu` y luego el paso a paso en `/docs/setup/quick-start` (después de `yarn dev`).

## Estructura del repo

```
vibecoding/
├── web/             ← Next.js app (donde construyes tu producto)
├── docs/            ← App de docs (espejo de /docs para deploy aparte)
├── docs-content/    ← Fuente MDX de la documentación
├── supabase/        ← Migrations y schema
└── design-system/   ← Reglas visuales para cuando la IA diseña UI
```

El mapa técnico completo (decisiones, convenciones y qué no tocar) está en
[`ARCHITECTURE.md`](./ARCHITECTURE.md). Las reglas para agentes de IA que editan
el repo (Cursor, Claude Code) están en [`CLAUDE.md`](./CLAUDE.md).

## Stack

- **Next.js 15** (App Router, JavaScript) + **Tailwind 4** + **DaisyUI**
- **Supabase** (Postgres) + **Google Auth**
- **OpenAI** SDK + **LangGraph.js** (agentes — material extra opcional)
- **Resend** (email) + **Vercel Web Analytics**
- **Vercel** + **yarn 1.x** workspaces

Además trae un **panel `/admin`** para ver tus leads del waitlist (protegido con
`ADMIN_PASSWORD`) y cobro simple con **PayPal.me** (toggle `features.paypal`).

Detalles y razones de cada elección: `/docs/intro/stack`.

## Cómo usar las docs

Las docs están dentro del repo (`docs-content/`) y se sirven en `/docs` cuando corres `yarn dev`. Cuando deployas a Vercel, tus docs también se publican — son **tuyas**, edítalas.

- **Tutoriales** (`/docs/tutoriales/semana-N`): qué hacer cada semana del curso.
- **Features** (`/docs/features/*`): cómo funciona cada pieza del stack.
- **Recetas** (`/docs/recetas/*`): playbooks completos para casos comunes (agente Gmail, marketplace simple, etc).
- **Troubleshooting** (`/docs/troubleshooting`): errores comunes y su solución.

## Licencia

MIT — usa esto para tu producto, modifícalo, distribúyelo. Si construyes algo cool, etiquétanos.
