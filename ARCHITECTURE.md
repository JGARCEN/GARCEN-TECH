# Arquitectura de Vibecoding

Este documento es el mapa técnico del repo: cómo está organizado, qué decisiones
lo sostienen y qué cosas **no** debes romper cuando lo extiendas (tú o la IA que
programa contigo). Si vas a tocar algo estructural, lee esto primero.

## Mapa del monorepo

Es un monorepo con **yarn 1.x workspaces**. Dos apps Next.js + contenido y schema compartidos:

```
vibecoding/
├── web/                  ← App principal (landing + producto). Workspace "web".
│   ├── config.js         ← LA fuente de verdad: branding, copy, features, pricing
│   ├── middleware.js     ← Refresh de sesión + protección de rutas privadas
│   ├── app/
│   │   ├── (marketing)/  ← Landing pública + /docs (espejo de docs-content/)
│   │   ├── (app)/        ← Zona privada: /dashboard, /chat, /agent (requieren login)
│   │   ├── admin/        ← Panel de leads /admin (contraseña, fuera de route groups)
│   │   ├── login/        ← Login con Google (fuera de route groups)
│   │   ├── auth/callback/← Callback OAuth de Supabase
│   │   ├── api/          ← waitlist, ai/{chat,structured,agent}, webhooks/resend
│   │   └── docs-search/  ← Índice JSON para el buscador de docs
│   ├── components/       ← landing/, layout/, auth/, ai/, docs/
│   └── lib/              ← supabase/, resend/, openai/, tools/, agents/, audit.js, docs.js
├── docs/                 ← App de docs standalone (workspace "docs", puerto 3001,
│                            deployable aparte; ver docs/README.md)
├── docs-content/         ← Fuente MDX de la documentación (la leen web/ y docs/)
├── supabase/             ← migrations/ (001–007), config.toml, seed.sql
├── design-system/        ← MASTER.md: reglas visuales para cuando la IA diseña UI
└── package.json          ← Workspaces + scripts (yarn dev, yarn docs:dev, etc.)
```

## Decisiones de diseño (las que no se negocian)

### 1. `config.js` es la única fuente de copy y configuración

Todo el branding, el copy de la landing, los planes de pricing y los toggles de
features viven en `web/config.js`. Los componentes leen de ahí; **ningún texto de
producto se hardcodea en JSX**. Para personalizar el producto se edita ese archivo,
no los componentes.

Ojo: `config.js` se evalúa en **build**. Cambiarlo en producción requiere re-deploy.

### 2. Features encendidas/apagadas con `config.features`

El patrón para funcionalidad opcional es un toggle en `config.features` que la
UI o la lib consultan antes de renderizar/actuar:

- `waitlist` y `pricing` → secciones de la landing (`web/app/(marketing)/page.js`)
- `googleAuth` → botón de login (`web/app/login/page.js`, `Navbar.js`)
- `paypal` → botón PayPal.me en Pricing (`web/components/landing/PaymentButton.js`)
- `resend` → cliente de email (`web/lib/resend/client.js` devuelve `null` si está off)

Si agregas una feature nueva, dale su toggle y respeta el patrón: apagada, la app
debe seguir funcionando como si la feature no existiera.

### 3. La landing funciona sin variables de entorno

Filosofía central: el repo debe **buildear y servir la página pública sin
configurar nada**. Cada integración degrada con gracia:

- `web/middleware.js` → si no hay env de Supabase, deja pasar todo (guard al inicio de `lib/supabase/middleware.js`).
- `web/lib/resend/client.js` → sin `RESEND_API_KEY` (o con `features.resend` off) devuelve `null` y los envíos son no-op silenciosos.
- `web/lib/supabase/server.js` → `getUser()` devuelve `null` si Supabase no está configurado.

Si tocas middleware, auth o email, **conserva estos guards**. Romperlos rompe la
experiencia de la Semana 1 (publicar la landing sin cuentas creadas).

### 4. Emails con `after()`: nunca bloquean la respuesta

El envío de email es best-effort y ocurre **después** de responder al usuario,
con `after()` de `next/server`:

- `web/app/api/waitlist/route.js` → inserta en `waitlist` y agenda `sendWaitlistConfirm()` solo si el alta es nueva (un email duplicado se trata como éxito: código Postgres `23505`).
- `web/app/auth/callback/route.js` → agenda `sendWelcome()` solo en el primer login (compara `created_at` vs `last_sign_in_at` con ventana de 10 s).

Los helpers viven en `web/lib/resend/send.js` y las plantillas React Email en
`web/lib/resend/templates/`. Mantén este patrón para cualquier email nuevo.

### 5. Auth: Google vía Supabase SSR

Patrón oficial de `@supabase/ssr` con tres clientes en `web/lib/supabase/`:

- `client.js` → navegador. `server.js` → Server Components / Route Handlers / Server Actions (y exporta `getUser()`). `middleware.js` → refresh de sesión.
- `admin.js` → cliente con `SERVICE_ROLE_KEY` que **se salta el RLS**. Solo servidor; hoy solo lo usa `/admin` para leer `waitlist`.

Flujo: `/login` → `signInWithOAuth` (Google) → `/auth/callback` hace
`exchangeCodeForSession` → redirect a `/dashboard`. El middleware protege los
prefijos `/dashboard`, `/account` y `/chat` (lista `PROTECTED_PREFIXES` en
`lib/supabase/middleware.js`) y `web/app/(app)/layout.js` repite el guard con
`getUser()`. **No reordenes la lógica del middleware**: `getUser()` debe correr
entre crear la response y devolverla o las cookies se desincronizan (está
comentado en el archivo).

### 6. Base de datos: migraciones ordenadas + RLS siempre

`supabase/migrations/` es la única definición del schema. Se aplican en orden:

| Migración | Qué crea |
|---|---|
| `001_auth_profiles.sql` | `profiles` + trigger `handle_new_user` (auto-crea el profile al registrarse) |
| `002_waitlist.sql` | `waitlist` (email único) |
| `003_core_items.sql` | `core_items`, el CRUD genérico del MVP |
| `004_ai_conversations.sql` | `ai_conversations` + `ai_messages` (historial de chat) |
| `005_tool_calls.sql` | `tool_calls` (auditoría de tools, RLS inline) |
| `006_analytics_events.sql` | `events` (tracking propio opcional) |
| `007_rls_policies.sql` | Policies RLS de las tablas base |

Reglas:

- Schema nuevo = **migración nueva** (`008_...`, `009_...`). Nunca edites una migración ya aplicada.
- Toda tabla nueva lleva su **RLS inline en la misma migración** (patrón de la 005).
- Defensa en profundidad: las server actions filtran por `user_id` **además** del RLS (ver `web/app/(app)/dashboard/actions.js`). No quites uno confiando en el otro.

### 7. Las docs viven en dos apps que comparten fuente

`docs-content/` (raíz del repo) es la única fuente MDX. La sirven **dos** apps:

- `web/` en `/docs` (para que el alumno tenga docs junto a su producto)
- `docs/` como sitio standalone (puerto 3001, deployable aparte)

Consecuencias que hay que respetar:

- Ambas resuelven la carpeta con `path.join(process.cwd(), "..", "docs-content")` (en `web/lib/docs.js` y `docs/lib/docs.js`). `docs-content/` **no se mueve** de la raíz.
- Los componentes de docs (`components/docs/*`), `lib/docs.js` y `lib/searchFilter.js` son **copias duplicadas** en `web/` y `docs/`. Si editas uno, replica el cambio en el otro.
- Los **redirects de docs** están duplicados en `web/next.config.mjs` y `docs/next.config.mjs` y deben mantenerse **idénticos**. Si mueves o renombras una página MDX, agrega el redirect en los dos archivos.
- MDX gotcha: una `{` sin escapar en prosa rompe el build (acorn la lee como expresión JS). Escríbela como `\{`.

### 8. IA: OpenAI + registry de tools + agentes opcionales

- `web/lib/openai/` → `client.js`, `chat.js` (streaming), `structured.js` (outputs con Zod). Modelos y parámetros salen de `config.ai`.
- `web/lib/tools/index.js` → **registry central** de herramientas para function calling, compartido por chat y agentes. Una tool nueva = un archivo en `lib/tools/examples/` con su `execute()` + registro en el índice.
- `web/lib/audit.js` → `logToolCall()` registra cada invocación en `tool_calls`. Best-effort: nunca lanza.
- `web/lib/agents/` → agentes LangGraph (`graph.js` + `examples/recoverDecideAct.js`), expuestos en `/api/ai/agent` y `/agent`. Son **material extra opcional** del curso: no hagas que el resto del stack dependa de ellos.

### 9. Panel `/admin` de leads

Ruta fuera de los route groups (como `/login`): no usa layout de marketing ni de
app. Auth simple por contraseña (`ADMIN_PASSWORD` en `.env.local`) con cookie
httpOnly de 8 horas (`web/app/admin/actions.js`). Lee `waitlist` con el cliente
admin (service role) porque la migración 007 no permite leerla con el anon key.
Es deliberadamente simple para un proyecto de curso; no lo conviertas en un
sistema de roles sin necesidad.

### 10. Analytics: Vercel Web Analytics

`<Analytics />` de `@vercel/analytics` está montado en `web/app/layout.js`. Cero
configuración: se activa solo al deployar en Vercel. Para eventos propios existe
la tabla `events` (migración 006) como opción manual.

## Convenciones del repo

- **JavaScript, no TypeScript.** (El paquete `typescript` en devDependencies solo existe porque `eslint-config-next` lo exige.)
- **Español** en copy, comentarios, docs y mensajes de commit.
- **yarn 1.x, nunca npm.** No mezcles lockfiles.
- **Node 20 o 22 LTS** (hay `.nvmrc` con `22`; con nvm: `nvm use`).
- **Tailwind 4 + DaisyUI**: el color primario se inyecta desde `config.brand.primary` como `--color-primary`. Las reglas visuales completas están en `design-system/vibecoding/MASTER.md`.
- Imports con alias `@/` dentro de cada workspace.
- En Vercel, la app principal se deploya con **Root Directory = `web`** (y las docs, opcionalmente, con Root Directory = `docs`).

## Comandos

Desde la raíz del monorepo:

```bash
yarn install       # instala todo (workspaces)
yarn dev           # app principal en :3000
yarn build         # build de web
yarn lint          # lint de web
yarn docs:dev      # app de docs en :3001
yarn docs:build    # build de docs
yarn docs:lint     # lint de docs
```

## Checklist antes de dar un cambio por bueno

1. `yarn lint && yarn build` pasan (y `yarn docs:build` si tocaste docs).
2. Sin env vars configuradas, la landing sigue renderizando.
3. Si moviste una página MDX: redirect agregado en **ambos** `next.config.mjs`.
4. Si editaste un componente de docs: cambio replicado en `web/` y `docs/`.
5. Si creaste una tabla: migración nueva con RLS inline.
6. Copy nuevo en `config.js`, no en JSX.
