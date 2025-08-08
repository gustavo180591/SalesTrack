# SalesTrack — plan.md
> Paso a paso para montar el sistema de orquestación de bots (SvelteKit 2, Svelte 5, Tailwind CSS 4, TypeScript, Prisma, PostgreSQL, Docker) con logs/métricas en tiempo real, scheduler mínimo, webhooks HMAC y despliegue ágil.  
> **Principios:** seguro por defecto, DX simple, sin violar ToS de plataformas (p.ej. Meta/Instagram).

---

## 0. Requisitos previos
- Node.js ≥ 20, PNPM/NPM.
- Docker Desktop o Podman + Docker Compose.
- Git.
- psql (opcional para inspección local).
- Editor con ESLint/Prettier (VS Code recomendado).

**Checklist**
- [x] `node -v` y `docker -v` correctos
- [x] Puerto 5173 libre (dev), 5432 (DB)

---

## 1. Crear el monorepo base
1) Inicializar proyecto
```bash
mkdir salestrack && cd salestrack
git init
npm init -y
```
2) Estructura mínima
```
apps/web            # SvelteKit (frontend + endpoints SSR)
prisma              # Esquema, migraciones, seeds
infra               # Docker y orquestación
scripts             # helpers (webhooks, e2e, etc.)
```
3) Agregar SvelteKit + TS + Tailwind 4
```bash
npm create svelte@latest apps/web
# Elegir: Skeleton + TypeScript + ESLint + Prettier + Playwright
cd apps/web && npm i && cd ../..
# Tailwind 4 (compat) según guía oficial
```

**Criterio de aceptación**
- [x] `npm run dev` arranca SvelteKit vacío.

---

## 2. Configurar Docker y Postgres
1) `infra/docker-compose.yml` con **db** (PostgreSQL 16) y **web**.
2) Variables básicas en `.env.example` (copiar a `.env` en dev).  
3) Volumen persistente para datos.

```bash
npm run dev:docker   # Levanta DB y app
```

**Criterio de aceptación**
- [ ] DB saludable (`pg_isready`)
- [ ] App accesible en `http://localhost:5173`

---

## 3. Modelado de datos (Prisma)
1) Crear `prisma/schema.prisma` con modelos:
   - `User`, `Bot`, `Task`, `TaskEvent`, `Log`, `Schedule`, `AuditLog` + enums.
2) Índices: `Task.status`, `Task.scheduledFor`, `Log.createdAt`, `Bot.status`.
3) Instalar Prisma y generar cliente:
```bash
npm i -D prisma
npm i @prisma/client
npx prisma init --datasource-provider postgresql
npx prisma migrate dev -n init
```
4) Seed inicial (`prisma/seeds/seed.ts`): admin, operator, bot demo, tareas/logs.

**Criterio de aceptación**
- [ ] Migraciones aplicadas
- [ ] `npx prisma db seed` crea usuarios/bot de demo

---

## 4. Autenticación y RBAC
1) Implementar login con email/password (argon2/scrypt).  
2) Sesión vía cookie HttpOnly `sid` (SameSite=Lax, Secure en prod).  
3) Roles: `ADMIN`, `OPERATOR`.
4) Guards en rutas + `AuditLog` por acción sensible.

**Criterio de aceptación**
- [ ] Usuario `admin@salestrack.local` puede acceder a Settings
- [ ] `OPERATOR` ve dashboard y corre tareas, sin acceso a Settings

---

## 5. Validación y manejo de errores
1) Zod en `src/lib/validation/*` (por ejemplo `TaskCreateSchema`).  
2) Validador global en endpoints (400 si falla).  
3) Manejador centralizado de errores (log estructurado + mensaje seguro).

**Criterio de aceptación**
- [ ] Crear tarea con payload inválido responde 400 con detalles
- [ ] Errores se registran sin filtrar datos sensibles al cliente

---

## 6. API interna (bots, tareas, salud)
### Rutas
- `GET /api/health` → `{status:'ok'}`
- `GET /api/version` → `{version, commit?}`
- `GET|POST /api/bots` (listado/alta-actualización + `secretHash`)
- `GET|POST /api/tasks` (listar/crear)
- `GET|POST /api/tasks/[id]` (`GET`: detalle; `POST`: `{action:'cancel'}`)

**Criterio de aceptación**
- [ ] Crear una tarea retorna objeto `Task`
- [ ] Cancelar cambia `status` a `CANCELED` (si procede)

---

## 7. Webhooks firmados (bots externos)
1) Endpoint `POST /api/webhooks/bot-events` (BODY JSON).
2) Cabeceras: `x-bot-id`, `x-timestamp`, `x-signature` (HMAC-SHA256).  
   Firma: `HMAC(secret, ts + "\\n" + body)`.
3) Validar ventana temporal (`HMAC_WINDOW_SEC`, ej. 300) y **IP allowlist** opcional.
4) Persistir `TaskEvent` y `Log`; publicar en canal realtime.

**SDK mínimo**  
- Node TS: `createClient({ botId, secret, baseUrl }).send(evt)`  
- Python: `create_client(bot_id, secret, base_url)['send'](evt)`

**Criterio de aceptación**
- [ ] Webhook con firma válida = 2xx; inválida = 401
- [ ] Evento `TASK_LOG` aparece en la UI en < 2s

---

## 8. Realtime (SSE por defecto, WS opcional)
1) Bus interno `publish/subscribe`.  
2) Endpoint `GET /api/realtime/logs?botId=..&taskId=..` (SSE).  
3) (Opcional) `GET /api/ws` para WebSocket.

**Criterio de aceptación**
- [ ] Abrir pestaña Logs muestra streaming en vivo
- [ ] Reconexión automática (`retry`) funciona

---

## 9. Scheduler mínimo + colas
1) Guardar reglas `cronExpr` o `rrule` en `Schedule`.  
2) Loop `tick()`:
   - Dispara `taskTemplate` cuando vence.
   - Toma `PENDING` por prioridad/antigüedad.
   - Respeta `Bot.concurrency` (locks simples por bot).
3) Backoff exponencial con jitter hasta `maxAttempts`.
4) Timeout por tarea → marca `FAILED` si excede.

**Criterio de aceptación**
- [ ] `0 * * * *` crea una `Task` cada hora
- [ ] `concurrency=1` nunca corre 2 tareas del mismo bot a la vez

---

## 10. UI (SvelteKit + Tailwind 4)
### Páginas
- **Dashboard**: KPIs (tareas por estado, éxito 7d, tiempo promedio), tabla de errores recientes, estado de bots.
- **Bots**: lista, `status`, `lastSeenAt`, `concurrency`, acciones `start/stop` (placeholders).
- **Tareas**: formulario (bot + acción + payload + prioridad + timeout + schedule opcional). Listado con filtros.
- **Logs**: stream en vivo + búsqueda (texto, nivel) + filtros (fecha, bot, task).
- **Métricas**: líneas/barras (tareas por estado, éxito 7d, tiempos) y Top bots 24h.
- **Settings**: claves, ventana HMAC, allowlist IP, rate limits, modo dark.

### Componentes reutilizables
- `Card`, `Table` (paginación server), `Modal`, `Toast`, `Badge`, `CodeBlock`, `Form/Field`, `Chart` (SVG o Recharts opc.).

**Criterio de aceptación**
- [ ] Form valida con Zod en cliente/servidor
- [ ] Logs con resaltado por nivel y copia de mensaje (`CodeBlock`)

---

## 11. Seguridad
- Cookies seguras + CSRF (forms) y verificación `Origin/Referer` en JSON sensibles.
- Rate limit: login/webhooks (in-memory dev; Redis prod opc.).
- Sanitización de logs/render.
- `AuditLog` por acción con IP y actor.

**Criterio de aceptación**
- [ ] Fuerza bruta de login bloqueada tras umbral
- [ ] Webhook desde IP no permitida → 403 (si allowlist activo)

---

## 12. Métricas y consultas base
- Tareas por estado, tasa de éxito (7d), tiempo promedio por tipo, errores por tipo (7d), top bots activos (24h).  
- Exponer endpoints o cargar directamente en página Métricas.

**Criterio de aceptación**
- [ ] KPIs/Charts cargan < 1s con 10k tareas
- [ ] Consultas usan índices definidos

---

## 13. Tests
### Unit (Vitest)
- HMAC `sign/verify` + ventana temporal.
- Backoff exponencial + jitter.
- Scheduler: cron dispara correctamente.

### E2E (Playwright)
- Login → crear tarea → webhook helper envía `TASK_STARTED` + `TASK_LOG` → UI lo muestra.  
- Cancelar tarea → estado cambia y se loguea `AuditLog`.

**Criterio de aceptación**
- [ ] `npm test` verde
- [ ] `npm run e2e` realiza flujo completo sin flakes

---

## 14. DX
- Scripts NPM: `dev`, `build`, `preview`, `lint`, `format`, `test`, `e2e`, `seed`, `dev:docker`.
- Troubleshooting en README: SSL DB, clock skew, CORS, body size, puertos.
- Husky (opcional) para `pre-commit` (lint/format).

**Criterio de aceptación**
- [ ] `npm run dev:docker` deja todo listo en 1 comando
- [ ] `npm run seed` crea datos demo consistentes

---

## 15. Deploy
### Alternativas
- **Vercel**: SSR + SSE ok; WS usar Edge/Serverless WebSockets o migrar a Fly/Railway.
- **Fly.io / Railway**: contenedor con Node 20; `DATABASE_URL` gestionada.
### Pasos
1) Build con `sveltekit build` (adapter apropiado).
2) `prisma migrate deploy` contra DB de prod.
3) Configurar variables de entorno (AUTH_SECRET, HMAC_SECRET, PUBLIC_BASE_URL, RATE_LIMITS, TZ).
4) Endpoints `/health` y `/version` para probes.
5) Observabilidad: logs a stdout (agregar export opcional).

**Criterio de aceptación**
- [ ] Deploy 1-click o mínimamente documentado
- [ ] `/health` responde 200 en prod

---

## 16. Legal/ToS (no negociable)
- No scraping agresivo ni automatizaciones fuera de APIs oficiales.
- Respetar límites de rate y permisos de cada plataforma.
- Acciones `like_post` / `follow_user` solo como **placeholders** en el sistema de tareas.

---

## 17. Roadmap (v2+)
- Multi-tenant (`tenantId`) + políticas de aislamiento.
- Redis para colas y rate limit distribuido.
- Webhooks salientes (Slack/Discord al fallar).
- Export de logs a S3/BigQuery; retención escalonada.
- Rotación automática de secretos por bot + JTI para anti-replay.

---

## 18. Lista de verificación final
- [ ] Proyecto arranca tras clonar (README claro)
- [ ] UI sobria (Tailwind 4), dark mode
- [ ] Formularios con Zod (client+server) y estados (loading/ok/error)
- [ ] Streaming de logs (SSE) + filtros/búsqueda
- [ ] Endpoints bots/tasks/webhooks/health/version
- [ ] Seguridad (login, RBAC, rate-limit, CSRF, HMAC, allowlist opc.)
- [ ] Scheduler mínimo con cron y backoff
- [ ] Prisma models + índices + seeds
- [ ] Tests: Vitest + Playwright (E2E)
- [ ] Docker y scripts DX
- [ ] Guías de deploy (Vercel/Fly/Railway)
