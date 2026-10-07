# AGENTS.md — ERP System (Monorepo)

Instrucciones para agentes AI de coding que trabajen en este workspace.
Última actualización: 2026-10-07

---

## Estructura del Proyecto

Este directorio envuelve dos repositorios Git independientes que forman el ERP System:

```
erp-system/
├── AGENTS.md                         ← Este archivo (guía global)
├── docs/                             ← Ingeniería de requerimientos (REQ por módulo; ver docs/README.md)
├── business_management/              ← Backend Go (API REST) — puerto :3001
│   ├── AGENTS.md                     ← Guía específica backend
│   ├── main.go
│   ├── internal/                     ← ★ Arquitectura limpia (en migración)
│   │   ├── catalog/                  ← Contexto acotado: Product + Variants (FASE 5 activa)
│   │   ├── currency/                 ← Contexto acotado migrado (FASE 1 ✅)
│   │   ├── paymentmethod/            ← Contexto acotado migrado (FASE 2 ✅)
│   │   ├── tax/                      ← Contexto acotado migrado (FASE 3 ✅)
│   │   ├── brand/                    ← Contexto acotado migrado (FASE 4 ✅)
│   │   └── …                         ← + sifen, documents, party, sale, purchase, inventory,
│   │                                    analytics, identity, manufacturing… (ver internal/)
│   ├── wiring/                       ← Composition root (inyección de dependencias)
│   ├── apperr/                       ← Errores de dominio tipados
│   ├── txm/                          ← Unit of Work / transacciones cross-repo
│   ├── handlers/                     ← HTTP handlers legacy (se migran por fase)
│   ├── models/                       ← Modelos anémicos legacy (se reemplazan)
│   ├── database/
│   │   ├── migrations/               ← Migraciones SQL versionadas
│   │   └── postgres/                 ← Adaptadores de persistencia
│   └── conductor/                    ← Planes y status de migración
└── erp-webapp/                       ← Frontend React SPA — puerto :5173
    ├── src/
    │   ├── features/                 ← ★ Feature-Sliced Design (en migración)
    │   │   ├── products/             ← Migrado a TSX + Zod ✅
    │   │   ├── purchases/            ← Migrado a Feature-Sliced ✅
    │   │   └── sales/                ← Migración en progreso (~)
    │   ├── domain/                   ← Lógica de negocio pura (cálculos, reglas)
    │   │   ├── purchase/pricing/     ← Migrado ✅
    │   │   └── sale/calculations/    ← Migrado ✅
    │   ├── store/                    ← Zustand stores globales
    │   ├── components/ui/            ← Radix UI + Fluent 2 Design System
    │   ├── services/                 ← Clientes HTTP hacia el backend
    │   └── lib/i18n.js               ← Sistema de internacionalización
    ├── design/                       ← design/tokens.json (fuente única de tokens, Tailwind 4)
    └── conductor/                    ← Planes y status de refactorización UI/UX
```

---

## Tech Stack

| Capa         | Tecnología                                         | Puerto  |
| :----------- | :------------------------------------------------- | :------ |
| **Backend**  | Go 1.24+, Gorilla Mux, PostgreSQL, JWT             | `:3001` |
| **Frontend** | React 19, Vite 6, TailwindCSS 4, Zustand, Radix UI | `:5173` |
| **DB**       | PostgreSQL (lib/pq, golang-migrate)                | `:5432` |

---

## Arquitectura Backend — Migración en Curso

El backend está en transición de un **monolito por carpetas** hacia **Clean Architecture pragmática**
(Ports & Adapters sin la ceremonia completa). La migración sigue el patrón **Strangler Fig**: un
contexto acotado a la vez, sin big-bang, con el sistema shippeable después de cada fase.

### Arquitectura Objetivo (por contexto acotado migrado)

```
internal/<contexto>/
├── domain.go          ← Entidades de dominio puras (sin tags db: ni json:)
├── port.go            ← Interfaces (puertos) que define el dominio
├── service.go         ← Use cases / lógica de negocio
├── service_test.go    ← Tests unitarios con mocks por puerto (no god-mock)
├── handler.go         ← Adaptador HTTP (solo parsea request/response)
└── repository.go      ← Adaptador de persistencia (implementa el puerto)
```

### Estado de la Migración

| Fase    | Contexto                                                  | Estado         | Tareas |
| :------ | :-------------------------------------------------------- | :------------- | :----- |
| FASE 0  | Scaffolding (`apperr`, `txm`, `wiring`, `depguard`)       | ✅ Completa    | 6/6    |
| FASE 1  | Currency                                                  | ✅ Completa    | 9/9    |
| FASE 2  | PaymentMethod                                             | ✅ Completa    | 8/8    |
| FASE 3  | Tax (Classification + Rate)                               | ✅ Completa    | 11/11  |
| FASE 4  | Brand                                                     | ✅ Completa    | 8/8    |
| FASE 5  | Catalog (Product + Variants + Tags + Search)              | 🔄 En progreso | 2/7    |
| FASE 6  | Inventory + Stock                                         | ⏳ Pendiente   | 0/8    |
| FASE 7  | Pricing (+ Redis cache adapter)                           | ⏳ Pendiente   | 0/8    |
| FASE 8  | Sale (envuelve SQL, no reescribe)                         | ⏳ Pendiente   | 0/8    |
| FASE 9  | Purchase (partir god file 2.513 líneas)                   | ⏳ Pendiente   | 0/8    |
| FASE 10 | Analytics/BI (11 repos read-only)                         | ⏳ Pendiente   | 0/8    |
| FASE 11 | User/Auth/Roles                                           | ⏳ Pendiente   | 0/8    |
| FASE 12 | Eliminar service-locator global (`GetRepository`/`GetDB`) | ⏳ Pendiente   | 0/4    |
| FASE 13 | Partir `addRoutes` (≤ 100 líneas)                         | ⏳ Pendiente   | 0/4    |

**Métricas de progreso:**

- God-interface `Repository`: **eliminada** (FASE 12, 2026-07-31 — junto a `services/`; los contextos de `internal/` definen puertos propios en `port.go`)
- Tests: 14 paquetes OK, 0 FAILs

### Reglas Críticas para Agentes en el Backend

> **NUNCA** revivir el god-interface: `repository/` (y su `GetRepository`/`GetDB`) fue
> eliminado en FASE 12 (2026-07-31). Cada contexto define sus propios puertos en
> `internal/<contexto>/port.go` y recibe sus dependencias por constructor vía `wiring/`.

> **NUNCA** llamar a `repository.GetRepository()` o `repository.GetDB()` desde código nuevo.
> Usar inyección por constructor a través de `wiring/`.

> **NO** poner lógica de negocio en handlers. Los handlers solo parsean request/response y
> delegan al use case en `internal/<contexto>/service.go`.

> Los contextos en `internal/` tienen `depguard` activo: no pueden importar los paquetes
> legacy (`handlers/`, `services/`, `repository/`, `database/postgres/`).

---

## Arquitectura Frontend — Migración en Curso

El frontend está en transición hacia **Feature-Sliced Design** con separación estricta entre
lógica de dominio, estado y presentación.

### Estructura Objetivo (por feature migrada)

```
src/features/<feature>/
├── components/        ← Componentes visuales puros (solo UI)
├── hooks/             ← Hooks con lógica de UI y orquestación
└── types/             ← Tipos TypeScript del feature

src/domain/<feature>/  ← Lógica de negocio pura (cálculos, validaciones)
                          Sin dependencias de React, sin side effects
```

### Estado de la Migración Frontend

| Módulo                      | Estado         | Notas                                                 |
| :-------------------------- | :------------- | :---------------------------------------------------- |
| Products                    | ✅ Completo    | TSX, Zod, glass-mica/acrylic, hooks puros             |
| Purchases                   | ✅ Completo    | Feature-Sliced, `usePurchasesLogic`, domain pricing   |
| Sales                       | ✅ Completo    | `useSalesLogic` purgado (2026-10-05, FE e52c737); feature 100% TS, dominio en `domain/sale/` |
| Clientes, Proveedores, etc. | ⏳ Pendiente   | Sin feature-sliced aún                                |

### Reglas Críticas para Agentes en el Frontend

> **NUNCA** strings hardcoded de UI. Usar siempre `src/lib/i18n.js`.

> **NUNCA** lógica de cálculos o reglas de negocio dentro de componentes React.
> Moverlos a `src/domain/<feature>/`.

> Respetar el Design System **Fluent 2**: usar efectos `glass-mica`/`glass-acrylic` en modales,
> elevación con `shadow-fluent-*`, tipografía modular, micro-animaciones `animate-in fade-in`.

> **Guía operativa de UI:** antes de generar cualquier interfaz, leer `erp-webapp/DESIGN.md`.
> Contiene reglas binarias, mapeo token → clase Tailwind real (según `design/tokens.json`),
> recetas copy-paste de componentes (`src/components/ui/`), anti-patrones y checklist.

> **Performance React (skill `vercel-react-best-practices`):** usar siempre esta skill del
> workspace al escribir o refactorizar React en `erp-webapp/`, siempre que sea conveniente:
> features nuevas, hooks con estado, listas, data fetching y refactors → auditar contra sus
> reglas (`rerender-*`, `rendering-*`, `bundle-*`, `client-*`, `js-*`) y corregir o documentar
> cada hallazgo; en hotfixes triviales aplicarla con criterio. Este proyecto es una SPA Vite
> sin SSR: las reglas `server-*` no aplican. Escribir el reporte en `conductor/` del repo
> SOLO cuando el usuario lo pida explícitamente (ejemplo de reporte previo:
> `AUDIT_REACT_PERFORMANCE_MAINLAYOUT_FRONTEND.md`).

> Package manager: **pnpm** exclusivamente. Nunca `npm` ni `yarn`.

> **Contrato HTTP (post-F7.7):** `VITE_API_URL` debe ser la URL directa del backend
> (ej. `http://localhost:5050`). El proxy rewrite se eliminó en Vite y Nginx; el backend tiene CORS
> para `localhost:5173`. Si se reactiva un proxy inverso, cambiar `VITE_API_URL=/api` y restaurar el
> rewrite en `vite.config.js` + `nginx.conf` **en el mismo diff**.

---

## Perfiles de Agente (Hermes)

Este workspace se gestiona con Hermes Agent. 4 perfiles por nivel de complejidad:

| Perfil      | Cuándo usarlo                           | Modelo        | Reasoning | Harness                    |
| :---------- | :-------------------------------------- | :------------ | :-------- | :------------------------- |
| `quick`     | Hotfixes, typos, cambios triviales      | Qwen 3.7 Plus | high      | Hermes                     |
| `engineer`  | Features, endpoints, componentes, tests | GLM 5.2       | max       | Hermes → delega a OpenCode |
| `architect` | Diseño, refactors grandes, planning     | Kimi K3       | medium    | Hermes → delega a `agy`    |
| `reviewer`  | Code review, auditorías, verificación   | GLM 5.2       | high      | Hermes                     |

### Selección de Harness Worker por Área

| Área                   | Harness preferido | Modelo                | Motivo                                                     |
| :--------------------- | :---------------- | :-------------------- | :--------------------------------------------------------- |
| **Backend Go**         | OpenCode          | Kimi K3 / Deepseek V4 | `opencode run` one-shot, sin TUI, ideal para orquestación  |
| **Frontend React**     | AGY CLI           | Gemini 3.1 Pro        | Integración superior con TailwindCSS 4, Radix UI, Fluent 2 |
| **Review / Auditoría** | Hermes reviewer   | Kimi K3 high          | Integrado con Kanban, reportes estructurados               |
| **Planning / ADR**     | AGY CLI           | Gemini 3.1 Pro        | Razonamiento arquitectónico profundo                       |

### Comandos CLI

```bash
quick chat                            # Hotfixes rápidos
engineer chat                         # Desarrollo estándar
architect chat                        # Diseño y planning
reviewer chat                         # Code review

# Override de modelo puntual:
engineer chat -m deepseek/deepseek-v4-pro # Para tareas Go intensivas
architect chat -m glm-5.2-thinking    # Thinking explícito

hermes kanban list --board erp        # Ver tareas del board ERP
hermes kanban create --board erp      # Crear tarea nueva
hermes project show erp-system        # Estado del workspace
~/.hermes/scripts/init-feature.sh <backend|frontend> <feature_name> # Inicializar tríada conductor
```

---

## Idioma

- **Comunicación con el usuario**: Siempre en español
- **Código**: En inglés (variables, funciones, comentarios)
- **Commits**: Conventional commits en inglés (`feat:`, `fix:`, `refactor:`)
- **Documentación interna**: Español

---

## Reglas Generales

1. **Separación de repos**: Backend y frontend son repositorios Git independientes. No mezcles lógica.
2. **API Contract**: Cualquier cambio de endpoint en el backend requiere actualizar el frontend.
3. **No rompas builds**: Verificar `make build` (backend) y `pnpm build` (frontend) antes de cada commit.
4. **Tests obligatorios**: Toda lógica nueva lleva tests. Backend: `go test ./...`. Frontend: `pnpm test`.
5. **Planes en conductor/**: Antes de refactors, documentar el plan en `conductor/` del repo afectado.
6. **Migraciones SQL**: Solo mediante `make migrate-create`. Nunca modificar migraciones existentes.
7. **Fases de arquitectura**: Respetar el orden de las fases. No saltar fases ni mezclar contextos.
8. **Puertos de desarrollo — liberar ANTES de levantar** (`make run`, `pnpm dev`): los procesos dev viven detached y nadie los mata al cerrar la sesión o la terminal; `make run` no detiene la instancia previa, solo intenta bind → `address already in use` (5050 backend, 3001 docs-server, 5173 Vite). Verificar y liberar por PID — fuente de verdad `ss -tlnp`:

   ```bash
   ss -tlnp | grep -E ':3001|:5050|:5173'   # quién retiene cada puerto
   kill <PID>                                # liberar
   ```

   - Backend :5050 corre detached (`nohup ./bin/business_management &`) por workflow: los agentes lo reinician tras cambios BE y queda vivo como instancia compartida. Matarlo solo cuando el owner vaya a correr `make run` él mismo.
   - `go run` deja binarios temporales llamados `main` (~/.cache/go-build) imposibles de pkill por nombre: matar SIEMPRE por PID de `ss`.
   - El recipe `run` solo trapea el docs-server: un `make run` interrumpido (terminal cerrada) deja el docs huérfano reteniendo :3001 (systemd --user lo adopta). Si `make run` falla con bind, revisar AMBOS puertos.
   - Vite :5173 es de vida larga por diseño (reuso entre sesiones). Una segunda instancia salta a :5174 y el backend rechaza CORS (solo permite 5173/3000): matar la previa, no convivir.
   - `pkill -f <patrón>` puede matchear la propia shell que lo ejecuta si el patrón viaja en su línea de comando (zsh -c …): anclar (`^/ruta/bin`) o matar por PID.
9. **Versionado**: al cerrar un hito verificado, taggear `vX.Y.Z` en AMBOS repos y sincronizar `package.json`. Convención completa en la sección **Versionado y Releases**.

---

## Versionado y Releases

- **SemVer** `vMAJOR.MINOR.PATCH`, mismo número en backend y frontend por release. MAJOR: ruptura de contract API, migración que exige intervención de datos o cambio de flujo de instalación. MINOR: features nuevas visibles (módulos, endpoints, pantallas). PATCH: fixes sin features.
- Los tags se crean sobre `dev` (línea viva) al cerrar el hito; `git tag` NO viaja con `git push` simple → `git push origin dev --tags`.
- **Identificación en runtime**: `make build` inyecta `git describe --tags` al binario vía ldflags (`internal/platform/version`) y `GET /version` (sin auth) lo reporta junto a `commit` y `build_date` — así se identifica qué versión corre una instalación de cliente. Worktree con cambios sin commit → reporta `vX.Y.Z-dirty` (build no reproducible; commitear antes de emitir un bundle).
- **Frontend**: sincronizar `"version"` de `package.json` con el tag en cada release (mismo número, sin prefijo `v`). Ese sync alimenta la UI: el badge de versión FE se inyecta al build (`__APP_VERSION__`) y no requiere pasos extra de release.
- **Versión en la UI** (PLAN_VERSION_UI_FRONTEND_2026-10-07): SOLO `SystemVersionBadge` (`src/components/`) renderiza versiones — sidebar, login, settings, branch selection. La comparación FE vs BE detecta instalaciones parcialmente actualizadas. Prohibido un literal `"vX.Y.Z"` en `src/` (guard `version-literal` en `lint:design`; excepciones: `src/domain/version/`, tests, pragma `allow-version` para refs de specs externas como SIFEN v1.2 / MT §11.1.1). Los payloads de auditoría `system_version` derivan de `FE_APP_VERSION` (`src/domain/version/feVersion.ts`), nunca un número propio.
- Hitos anteriores a v1.0.0 (2026-10-04) no tienen tag: identificarlos por SHA de commit.

---

## Convenciones Backend (Go)

- Imports: stdlib → third-party → internal
- Errores: siempre `if err != nil { return }`, nunca silenciar con `_`
- Context: propagar `context.Context` en toda la cadena de llamadas
- Handlers: solo parsean request/response — toda lógica va en el use case
- Tests: `testify` con table-driven tests; mocks por puerto (no god-mock)
- Linter: `golangci-lint run` debe pasar en verde (config: `depguard`, `gofmt`, `govet`)

## Convenciones Frontend (React)

- State: Zustand stores en `src/store/` para estado global; `useState`/`useReducer` para estado local
- Componentes: Radix UI primitives + Fluent 2 custom styling
- Validación: Zod en formularios, con feedback visual inmediato
- Autofill: todo `<form>` lleva `autoComplete="off"` y los campos de contraseña `autoComplete="new-password"` (login: `username`/`current-password`) — evita que el navegador guarde datos del ERP en su autofill (terminales compartidas del cliente; ver `docs/guides/GUIDE_TERMINALES_LOCAL.md` §4)
- i18n: `src/lib/i18n.js` siempre — nunca strings hardcoded de UI
- Branding UI: ninguna cadena visible dice "ERP" ni promete "ERP completo" — el producto se muestra como "sistema de gestión comercial" (sidebar, login, settings, branch selection, `<title>`, títulos de pestaña, footers de reportes impresos). Excepción: nombres de edición del licenciamiento (ERP Core, Core + BI Premium). Posicionamiento completo: `docs/sales/GUIA_VENTAS.md` §2.
- Temas: Fluent 2 con tokens en `design/tokens.json` (Tailwind 4 `@theme`; modo oscuro vía clase `.dark`)
- TypeScript: todos los archivos nuevos en `.tsx`/`.ts`, sin `.jsx`/`.js`
- Performance: seguir la skill `vercel-react-best-practices` del workspace (ver reglas críticas de frontend)

### Tests (diseño único para todos los módulos)

Fuente completa: `erp-webapp/conductor/PLAN_TEST_DESIGN_FRONTEND.md`. Reglas binarias:

1. **Ubicación canónica por tipo de sujeto**: `src/domain/**` → test colocado junto al módulo (`foo.test.ts`); features → `src/features/<f>/__tests__/`; páginas → `src/pages/__tests__/<Page>.page.test.*`; stores → `src/store/__tests__/<store>.store.test.*`; servicios → `src/services/__tests__/<service>.service.test.*`; contexts globales → `src/contexts/__tests__/`; componentes compartidos → colocados junto al componente. Tests nuevos siempre `.ts/.tsx` (`.js/.jsx` solo si el módulo bajo test aún es JS).
2. **Mock en la frontera del módulo consumido**, nunca internals. Barrel de feature con `importOriginal` para stubs puntuales (ej. modales pesados); servicios mockeados en `@/services/api` (para tests de servicio) o el módulo del service (para tests de store); stores Zustand con mock selector-compatible `(sel) => sel(state)`.
3. **i18n en tests**: `vi.mock('@/lib/i18n')` con `t: (key, fallback, vars) => …` respetando la firma real (fallback en español). Si un assert necesita texto real, la key debe existir en `locales/es` y `locales/en`; si falta, es un bug de i18n: agregarla en el mismo diff.
4. **NO mockear `lucide-react`** (jsdom renderiza SVG). Si algún día hace falta, solo parcial con `importOriginal` — prohibida la lista fija de exports.
5. **Asserts contra el contrato actual**: queries por rol / aria-label / data-testid (regex de testid ancladas); nunca clases cosméticas (BEM), contadores de render ni `getByText(a) || getByText(b)` (getByText lanza: usar `queryBy` para negativos).
6. **Componente retirado → DELETE del test** (documentado en `conductor/`), nunca "skip" ni placeholder siempre-verde.
7. **Gate de salida**: `npx vitest --run` 0 fallos, `npx tsc --noEmit` 0 errores, `pnpm build` y `pnpm lint:design` en verde. Recordar: `vite build` NO typechequea.
