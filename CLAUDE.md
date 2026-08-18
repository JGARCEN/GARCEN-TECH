# CLAUDE.md — Reglas para agentes de IA en este repo

Este repo es la plantilla del curso **Vibe Code** (Change and Code). Quien te
pide cambios es, casi siempre, un founder **no técnico** construyendo su producto
encima de esta plantilla. Tu trabajo es extenderla **sin romper su arquitectura**.
El mapa completo está en `ARCHITECTURE.md` — léelo antes de cambios estructurales.

## Comandos (siempre yarn, NUNCA npm)

```bash
yarn install       # instalar dependencias (yarn 1.x workspaces)
yarn dev           # app principal → http://localhost:3000
yarn build         # build de producción de web
yarn lint          # lint de web
yarn docs:dev      # app de docs → http://localhost:3001
yarn docs:build    # build de la app de docs
```

- **Nunca `npm install` ni `npm run`**: mezcla lockfiles y rompe los workspaces.
- Dependencias nuevas: `yarn workspace web add <paquete>` (o `yarn workspace docs add`).
- Node 20 o 22 LTS (`.nvmrc` = 22). Verifica un cambio con `yarn lint && yarn build` antes de darlo por terminado.

## Reglas de oro

1. **Copy y configuración van en `web/config.js`, no en JSX.** Textos de landing, branding, pricing y toggles viven ahí. Si el usuario pide cambiar textos o colores, edita `config.js` (y `config.brand.primary` para el color), no los componentes.
2. **No rompas el modo "sin env vars".** La landing debe buildear y renderizar sin ninguna variable de entorno. Conserva los guards: middleware deja pasar todo sin Supabase, Resend hace no-op sin API key, `getUser()` devuelve `null`. Nunca hagas que la página pública dependa de una integración configurada.
3. **Features opcionales se apagan con `config.features`.** Feature nueva = toggle nuevo + código que lo respeta. Apagada, la app funciona como si no existiera.
4. **Migraciones: solo hacia adelante.** Schema nuevo = archivo nuevo en `supabase/migrations/` (siguiente número: `008_...`). Nunca edites una migración existente. Toda tabla nueva lleva su **RLS en la misma migración**, y las server actions filtran por `user_id` además del RLS.
5. **Docs duplicadas = cambios duplicados.** `web/components/docs/*`, `lib/docs.js` y `lib/searchFilter.js` existen copiados en `web/` y `docs/`: edita ambos. Los redirects de `web/next.config.mjs` y `docs/next.config.mjs` deben quedar **idénticos**; si mueves/renombras un MDX de `docs-content/`, agrega el redirect en los dos.
6. **`docs-content/` no se mueve de la raíz.** Ambas apps lo resuelven con `path.join(process.cwd(), "..", "docs-content")`.
7. **Emails con `after()`.** Cualquier envío nuevo sigue el patrón de `api/waitlist` y `auth/callback`: responder primero, enviar después, best-effort vía helpers de `web/lib/resend/send.js`.
8. **`web/lib/supabase/admin.js` (service role) es solo-servidor.** Se salta el RLS: nunca lo importes desde componentes cliente ni expongas la key con `NEXT_PUBLIC_`.
9. **JavaScript, no TypeScript.** Nada de `.ts`/`.tsx` ni anotaciones de tipos. Todo el copy, comentarios y commits en **español**.
10. **En MDX, escapa las llaves en prosa**: `\{` — una `{` suelta rompe el build.

## Dónde va cada cosa

| Quieres agregar… | Va en… |
|---|---|
| Texto/copy/branding/planes | `web/config.js` |
| Sección de landing | `web/components/landing/` + render en `web/app/(marketing)/page.js` |
| Página privada (requiere login) | `web/app/(app)/` (+ prefijo en `PROTECTED_PREFIXES` de `web/lib/supabase/middleware.js` si es un prefijo nuevo) |
| Endpoint de API | `web/app/api/<nombre>/route.js` |
| Tabla / cambio de schema | `supabase/migrations/00X_*.sql` (con RLS) |
| Tool para la IA (function calling) | `web/lib/tools/examples/` + registro en `web/lib/tools/index.js` |
| Email transaccional | Plantilla en `web/lib/resend/templates/` + helper en `web/lib/resend/send.js` |
| Agente LangGraph (material extra) | `web/lib/agents/` (que nada del core dependa de esto) |
| Página de documentación | `docs-content/<sección>/*.mdx` (frontmatter: `title`, `description`, `order`) |
| Estilo/reglas visuales | `design-system/vibecoding/MASTER.md` manda; tema en `web/app/globals.css` |

## Qué NO hacer

- No uses npm, pnpm ni bun.
- No hardcodees textos de producto en componentes.
- No edites migraciones ya existentes ni crees tablas sin RLS.
- No toques el orden interno de `web/lib/supabase/middleware.js` (crear response → `getUser()` → devolver; está comentado ahí).
- No agregues features del stack que este template recortó a propósito: **no** Stripe (el cobro es PayPal.me), **no** PostHog (analytics es `@vercel/analytics`), **no** RAG/pgvector, **no** MCP, **no** hardware. Si el usuario los pide, impleméntalos como algo suyo, no como "parte del template".
- No conviertas `/admin` en un sistema de roles: es contraseña simple (`ADMIN_PASSWORD`) a propósito.
- No muevas `docs-content/` ni cambies las rutas relativas que lo resuelven.

## Al terminar cualquier cambio

```bash
yarn lint && yarn build
```

y si tocaste docs (contenido, componentes o redirects): `yarn docs:build`.
