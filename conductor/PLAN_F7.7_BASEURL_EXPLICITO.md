# PLAN F7.7 — Contrato HTTP explícito (sin doble `/api`)

**Repo:** `erp-webapp/` (FE puro · 0 cambios backend)
**Fecha:** 2026-08-05 · **Estado:** ✅ Ejecutado (gate verde · smoke pendiente de backend levantado)
**Contexto:** el proyecto está **fuera de producción** → se asume riesgo de refactor.
**Supera:** decisión "C — documentar + guardia" del `ADR-F7.6-DOBLE_API.md` (reapertura).
**Principio rector:** cada servicio pasa el path exacto que el backend registra. Sin prefijos
ocultos, sin rewrites que se cancelan, sin magia.

---

## Objetivo

Eliminar el acoplamiento del "doble `/api`" (D-B4 / F7.6) de raíz. Hoy:

```
v1:    FE /api/v1/brands → +baseURL /api → /api/api/v1/brands → rewrite quita 1 → /api/v1/brands
no-v1: FE /payment       → +baseURL /api → /api/payment        → rewrite quita 1 → /payment
```

El proxy rewrite está duplicado entre Vite (dev) y Nginx (prod), y los entornos sin proxy
(`.env.cloud`, `.env.demo`) **ya prueban** que el modo "URL directa al backend" funciona.

## Diseño: URL directa al backend (sin proxy)

Alinear dev/prod al modelo que cloud/demo ya usan. En vez de baseURL `/api` + proxy rewrite,
`baseURL` es siempre la URL real del backend. El backend ya tiene CORS configurado para
`http://localhost:5173` (`server/broker.go:163`).

```
ANTES (acoplado, 3 piezas dependientes):
  dev/prod:  baseURL=/api + endpoints + proxy rewrite (Vite + Nginx)
  cloud/demo: baseURL=URL backend + endpoints (sin proxy) ← ESTO YA FUNCIONA

DESPUÉS (uniforme, URL directa en todos lados):
  dev:    baseURL=http://localhost:5050 + endpoints → sin proxy, sin rewrite
  prod:   baseURL=URL backend + endpoints → Nginx solo sirve estáticos, no proxyea API
  cloud:  baseURL=URL Cloud Run + endpoints (sin cambio)
  demo:   baseURL=http://localhost:5050 + endpoints (sin cambio)
```

Trazado de una petición después del refactor:
```
v1:    /api/v1/brands → http://localhost:5050/api/v1/brands → backend match ✅
no-v1: /payment       → http://localhost:5050/payment       → backend match ✅
```

## Alcance medido (7 puntos planificados → 11 ejecutados)

El plan original medía 7 puntos. Durante la ejecución surgieron **4 consumers ad-hoc extra** con el
patrón `import.meta.env.VITE_API_URL || '/api'` que no usaban el cliente HTTP central — se detectaron
al romper 1 test de MSW (`useProductStore.cache.test.js`) y por barrido grep. Alcance final: **11 puntos**.

| # | Archivo | Cambio | Origen |
|:--|:--------|:-------|:-------|
| 1 | `.env.development:10` | `VITE_API_URL=/api` → `http://localhost:5050` | planificado |
| 2 | `.env.api:5` | `VITE_API_URL=/api` → `http://localhost:5050` | planificado |
| 3 | `.env.production:11` | `VITE_API_URL=/api` → `http://localhost:5050` (sobrescribir en deploy) | planificado |
| 4 | `src/utils/devAuth.js:31` | `\|\| '/api'` → `\|\| 'http://localhost:5050'` | planificado |
| 5 | `src/components/ui/ApiStatusIndicator.jsx:30` | mismo cambio | planificado |
| 6 | `vite.config.js:77-85` | eliminar bloque `proxy` | planificado |
| 7 | `nginx.conf:20-48` | eliminar `location /api/` + rewrite | planificado |
| 8 | `src/config/api.config.js:9` | fallback `\|\| '/api'` → `\|\| 'http://localhost:5050'` | surgido (consistencia) |
| 9 | `src/features/products/__tests__/useProductStore.cache.test.js:6` | mismo fallback | **surgido (rompió test MSW)** |
| 10 | `src/services/paymentApiDebug.ts:20` | mismo fallback | surgido (grep) |
| 11 | `src/components/payment/PaymentApiDiagnostic.jsx:20,114` | mismo fallback (2 sitios) | surgido (grep) |
| — | `.env.example:79` | comentario actualizado (ya no aplica `/api` para proxy) | surgido (docs) |

**Sin cambios:** `.env.cloud`, `.env.demo` (ya son el modelo), `BusinessManagementAPI.ts` (la
concatenación `baseUrl+endpoint` es correcta cuando `baseUrl` es la URL real), los 41 servicios (sus
paths ya son los correctos: v1 pasan `/api/v1/...`, no-v1 pasan `/<recurso>`).

## Decisiones

| # | Decisión | Razón |
|:--|:---------|:------|
| D-F7.7a | URL directa al backend (no baseURL='') | baseURL='' rompería el proxy Vite (no intercepta paths sin `/api`); URL directa es lo que cloud/demo ya usan y el backend tiene CORS para localhost:5173 |
| D-F7.7b | Eliminar el proxy, no dejarlo inerte | Un proxy inerte es configurable dead code; si no se usa, se borra. Reduce superficie |
| D-F7.7c | Nginx prod: solo servir SPA, no proxyear API | Alinea prod con cloud/demo (URL directa). Si en el futuro se quiere proxy inverso en prod, es un diff explícito |
| D-F7.7d | `getEndpoint()` de api.config.js se deja intacto | Está muerto (0 consumidores verificado). Su cleanup es cosmético y no afecta el contrato |
| D-F7.7e | `.env.production` usa `http://localhost:5050` (no placeholder) | Un placeholder rompería el build de Vite (necesita URL real). El default seguro se sobrescribe al desplegar |
| D-F7.7f | Alinear TODOS los consumers ad-hoc `|| '/api'` (no solo los planeados) | Surgieron 4 extra al romper un test MSW; el barrido grep los encontró. Consistencia crítica: si alguno queda con `/api`, se desincroniza del MSW y del cliente HTTP |

## Gate de verificación

| Ítem | Comando | Resultado |
|:-----|:--------|:----------|
| Typecheck | `pnpm exec tsc --noEmit` | ✅ 0 errores |
| Build | `pnpm build` | ✅ exit 0 (solo warning pre-existente de chunk size) |
| Tests | `pnpm test` | ✅ 60 fallos = baseline (verificado con stash: 60 sin cambios, 60 con cambios → 0 nuevos) |
| Smoke v1 | `GET http://localhost:5050/api/v1/brands` | ⏳ Pendiente — backend no levantado |
| Smoke no-v1 | `GET http://localhost:5050/products/list/1/10` | ⏳ Pendiente — backend no levantado |

**Resultado del gate de código:** ✅ verde. El smoke manual queda pendiente hasta levantar el
backend en `localhost:5050`; los paths ya están alineados con lo que el backend registra (verificado
en `routes.go` y los `PathPrefix` de cada contexto).

**Bug detectado y correguido durante la verificación:** el test MSW
`useProductStore.cache.test.js` rompió (1 fallo nuevo, 60→61) porque su fallback local `|| '/api'`
se desincronizó del nuevo default del cliente HTTP. Corregido alineando su fallback (punto #9 del
alcance). Esto validó la decisión D-F7.7f y demostró que el barrido grep de consumers ad-hoc era
necesario.

## Riesgos y mitigación

| Riesgo | Mitigación |
|:-------|:-----------|
| CORS en dev si el backend no permite localhost:5173 | Verificado: `server/broker.go:163` lo permite por defecto |
| Olvidar levantar `FRONTEND_URL`/`CORS_ALLOWED_ORIGINS` en prod | `.env.example` ya lo documenta; placeholder en `.env.production` |
| Nginx sin proxy en prod → SPA no encuentra API | Ese es el punto: baseURL es la URL del backend, el browser pega directo |
| `devAuth.js`/`ApiStatusIndicator.jsx` se rompen si olvido alinearlos | Están en el alcance (puntos 4 y 5) |
