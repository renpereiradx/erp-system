# ADR-F7.6 — El doble `/api` (contrato HTTP acoplado)

**Estado:** ✅ Superado por refactor F7.7 (2026-08-05) — ver `PLAN_F7.7_BASEURL_EXPLICITO.md`.
Originalmente "Aceptado (C: documentar + guardia)"; **reaperturado** cuando el proyecto pasó a
estar fuera de producción, permitiendo asumir el riesgo del refactor.
**Fecha:** 2026-08-05 (original) · 2026-08-05 (reapertura + F7.7)
**Supera:** hallazgo D-B4 de `ESTADO_INTEGRACION_BACKEND.md` §322, ítem F7.6 de
`ESTADO_INTEGRACION_FRONTEND.md`, decisión D-F7.6 de `PLAN_F7_CONTRATO_HTTP.md`.
**Decisión original:** NO se refactoriza el código. Se documenta el acoplamiento y se establece guardia.
**Decisión final:** reapertura → F7.7 ejecuta el refactor (URL directa al backend, sin proxy rewrite).

---

## Historial de la decisión

1. **2026-08-05 (mediodía):** Opción C (documentar + guardia) elegida — el proyecto estaba en
   producción y el riesgo cross-repo superaba al beneficio cognitivo.
2. **2026-08-05 (tarde):** El usuario comunica que el proyecto está **fuera de producción** → el
   riesgo cae drásticamente. Reapertura y ejecución de F7.7 (opción B evolucionada: URL directa al
   backend en vez de `baseURL=''`, porque cloud/demo ya lo probaban y el backend tiene CORS).

---

## Contexto

El frontend tiene **dos familias de endpoints** con convenciones distintas:

- **Familia v1** (9 servicios: `attributes`, `audit`, `brands`, `parties`, `permissions`, `roles`,
  `tags`, `users`, `variants`): pasan el path **completo** `/api/v1/<recurso>` al cliente HTTP.
- **Familia no-v1** (~12 familias: `payment`, `products`, `purchase`, `sale`, `stock`, etc.):
  pasan el path **directo** `/<recurso>` (ej. `/payment/process`).

El cliente HTTP (`BusinessManagementAPI.ts:122`) concatena `baseURL + endpoint` **sin normalizar**,
y `baseURL` vale `/api` en dev/prod (vía `VITE_API_URL`).

## Mecanismo del acoplamiento (verificado línea a línea)

| # | Pieza | Archivo:línea | Valor / comportamiento |
|:--|:------|:--------------|:-----------------------|
| 1 | `baseURL` | `src/config/api.config.ts:11` | `/api` (dev, prod) · URL directa (cloud, demo) |
| 2 | Concatenación | `src/services/BusinessManagementAPI.ts:122` | `` `${this.baseUrl}${endpoint}` `` — sin normalizar |
| 3a | Proxy dev | `vite.config.js:83` | `path.replace(/^\/api/, '')` — quita **un** `/api` |
| 3b | Proxy prod | `nginx.conf:24` | `rewrite ^/api/(.*)$ /$1 break;` — quita **un** `/api` |
| 4 | Endpoints v1 | 9 servicios en `src/services/*.ts` | pasan `/api/v1/<recurso>` completo |
| 5 | Endpoints no-v1 | ~12 familias en `src/services/*.ts` | pasan `/<recurso>` directo |

### Trazado de una petición (los dos casos)

```
Familia v1 — GET /api/v1/brands:
  Servicio pasa  "/api/v1/brands"
  Concatenación  /api + /api/v1/brands  =  /api/api/v1/brands
  Proxy (dev)    quita 1er /api         →  /api/v1/brands
  Proxy (prod)   rewrite ^/api/(.*)$    →  /api/v1/brands
  Backend        PathPrefix("/api/v1/brands")  →  MATCH ✅

Familia no-v1 — POST /payment/process:
  Servicio pasa  "/payment/process"
  Concatenación  /api + /payment/process =  /api/payment/process
  Proxy (dev)    quita /api             →  /payment/process
  Proxy (prod)   rewrite                →  /payment/process
  Backend        HandleFunc("/payment/process")  →  MATCH ✅
```

### Entornos sin proxy (cloud, demo)

`VITE_API_URL` apunta directo al backend (`http://localhost:5050` o la URL de Cloud Run) **sin
rewrite**. Verificado que ambos casos funcionan:

- v1: `URL + /api/v1/brands` → el backend sirve `/api/v1/...` ✅
- no-v1: `URL + /payment/process` → el backend sirve `/payment/...` ✅

## Por qué se reapertura (cambio de decisión)

Originalmente (opción C) se decidió NO refactorizar porque el proyecto estaba en producción y el
riesgo cross-repo superaba al beneficio. Cuando el usuario comunicó que el proyecto está **fuera de
producción** (2026-08-05), ese cálculo cambió:

- El riesgo de romper prod desaparece.
- El beneficio de eliminar la deuda cognitiva del doble `/api` pasa a primer plano.
- Cloud/demo ya prueban que el modo "URL directa al backend" funciona, así que el refactor no es un
  salto al vacío sino una alineación a un modo ya probado.

La opción elegida en F7.7 es "URL directa al backend" (evolución de la opción B original): en vez de
`baseURL=''`, se alinean dev/prod a lo que cloud/demo ya hacen — `baseURL` es siempre la URL real del
backend. El proxy rewrite se elimina en dev (Vite) y prod (Nginx) porque deja de ser necesario.

## Riesgo residual y guardia

El acoplamiento no es runtime, es **de mantenibilidad**. Rompe si alguien edita **una** pieza en
aislamiento sin tocar las demás. Escenarios prohibidos:

| Escenario | Qué rompe |
|:----------|:----------|
| Quitar `rewrite` del proxy Vite (`vite.config.js:83`) | Todos los no-v1 → 404 |
| Quitar `rewrite` del proxy Nginx (`nginx.conf:24`) | Todos los no-v1 → 404 (solo prod) |
| Quitar `/api/v1/` de los servicios v1 FE | Los 9 servicios v1 → 404 (backend espera `/api/v1/`) |
| Cambiar `baseURL` a `''` sin tocar proxy | Los no-v1 → 404 (pierden el `/api` que el rewrite quita) |
| Cambiar `VITE_API_URL` a URL completa pero olvidar que los no-v1 no llevan `/api` | Ya probado en cloud/demo: funciona |

### Guardia (también reflejada en `AGENTS.md`)

> **PROHIBIDO editar en aislamiento** cualquiera de estas piezas: `baseURL` (`api.config.ts`),
> el rewrite del proxy Vite (`vite.config.js:83`), el rewrite del proxy Nginx (`nginx.conf:24`),
> o los paths `/api/v1/...` de los 9 servicios v1. Cualquier cambio requiere tocar las demás en
> el mismo diff y verificar los tres entornos (dev con proxy, prod con Nginx, cloud/demo sin proxy).

## Cuándo reapertura este ADR

- Si se unifican las convenciones de ruteo del backend (todo bajo `/api/v1/` o todo sin prefijo) →
  momento ideal para alinear FE de paso.
- Si se migra el dev server o el proxy de prod y el rewrite se pierde o cambia.
- Si se añade un nuevo servicio cuyo path no encaja en ninguna de las dos familias y rompe la
  heurística.

## Alternativas consideradas y descartadas

- **A. Quitar `/api/v1/` de los servicios v1 FE (sin tocar proxy)**: descartada — genera 404
  porque el proxy quita un `/api` y el backend registra `/api/v1/...`, no `/v1/...`.
- **B. `baseURL=''` + quitar rewrite**: coherente (contrato explícito) pero requiere coordinar
  FE + Vite + Nginx + envs en un big-bang cross-repo. **Evolucionada en F7.7** a "URL directa al
  backend" (mismo efecto, mejor diseño: cloud/demo lo prueban, no hay que inventar baseURL vacío).
- **C. Documentar + guardia (este ADR en su forma original)**: ✅ elegida inicialmente (2026-08-05
  mediodía). **Superada** por F7.7 el mismo día al reaperturarse (proyecto fuera de producción).
- **D. Investigar cloud/demo primero**: ejecutada como paso previo; confirmó que URL directa
  funciona y abrió el camino a la opción B evolucionada que se ejecuta en F7.7.
