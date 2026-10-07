# GUIDE_HOSTING_NUBE — Opciones de alojamiento en la nube para el cliente

## 1. Ficha

| Campo | Valor |
|:------|:------|
| Módulo | Alojamiento del ERP en la nube (alternativa al servidor en el local) |
| Código | GUIDE-HOSTING-NUBE |
| Versión | v0.1 |
| Estado | **Propuesta — evaluación de opciones** (no implementado; ver §10) |
| Fecha | 2026-09-30 |
| Audiencia | Comercial, soporte y clientes |
| Relacionado | [`REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md`](../requirements/REQ_DEPLOY_DISTRIBUCION_ONPREMISE.md) (estrategia implementada hoy), [`GUIDE_PAQUETES_CLIENTE.md`](GUIDE_PAQUETES_CLIENTE.md) (las licencias no cambian) |
| Fuentes | Investigación web sep-2026: consenso de comunidad (Reddit r/selfhosted, r/hetzner, Hacker News, dev.to) + precios de lista de cada proveedor (ver §11) |

> **Alcance de esta versión:** catálogo de opciones con costos y recomendaciones para
> conversar con el cliente. La instalación en la nube **todavía no es un procedimiento
> implementado ni documentado paso a paso**: hoy el producto se instala y soporta
> on-premise (ver §10 "Qué falta").

---

## 2. Qué significa "llevar el sistema a la nube" (para todos)

Hoy el ERP vive en una **computadora del propio local** (laptop o servidor, perfil
`server`): el sistema, los datos y las impresiones funcionan dentro del negocio, y si
esa máquina o el local fallan, el sistema se detiene.

Alojarlo **en la nube** significa alquilar esa misma "computadora" en un centro de
datos profesional: el ERP corre idéntico (mismos contenedores, mismos datos, misma
licencia), pero las cajas y terminales se conectan desde el navegador a través de
internet, desde cualquier lado.

**A favor:**

- El sistema deja de depender del corte de luz, del robo de la máquina o del disco
  del local: el centro de datos tiene energía y internet redundantes.
- Las sucursales y los teléfonos entran al mismo sistema sin configurar VPN por
  sucursal (hoy cada sucursal requiere su tailnet).
- El proveedor puede dar soporte sin depender de que alguien en el local encienda
  el servidor.
- Las terminales ya no necesitan confiar en la CA interna: en la nube el certificado
  de seguridad es público (el candado del navegador funciona solo).

**En contra:**

- Aparece una **mensualidad en dólares con tarjeta internacional** (hoy la compra
  es única y el costo de correrlo es la electricidad del local).
- El negocio **deja de facturar si se corta el internet del local** (con el sistema
  en el local, un corte de internet no detiene las cajas). Se recomienda un enlace
  de respaldo (4G/fibra secundaria) si se elige esta modalidad.
- Los datos salen físicamente del local: quedan en el país de la región elegida
  (Brasil si se sigue la recomendación de §5) y hay que mantener la disciplina de
  backups descargables.

## 3. Qué cambia y qué no cambia

| | On-premise (hoy) | En la nube (propuesta) |
|:--|:--|:--|
| Dónde corre | Computadora del local | Servidor alquilado (VPS) en centro de datos |
| Instalación | `install.sh` desde el bundle | El mismo bundle + instalador (procedimiento a definir, §10) |
| Certificado de seguridad | CA interna (instalar en cada terminal) | Certificado público (nada que instalar) |
| Sucursales | VPN Tailscale por cliente + entidad | Solo usuario y contraseña, sin VPN |
| Impresión en el local | Impresoras locales | Igual — las terminales del local siguen imprimiendo allí |
| SIFEN / facturación electrónica | Conectividad del local | Conectividad del centro de datos (mejor) |
| Licencia | Perpetua, viaja con la instalación | **Idéntica — no cambia nada** (ver GUIDE_PAQUETES_CLIENTE) |
| Backups | Bundle etiquetado en disco/pendrive | Snapshot del proveedor + backup descargable externo |
| Costo | Compra única | Compra única **+ mensualidad del servidor en USD** |

## 4. Qué necesita el sistema (dimensionamiento)

El stack completo son 3 servicios Docker (PostgreSQL + API Go + frontend Nginx). Para
un negocio de 1–3 sucursales con uso intensivo de cajas y consultas BI:

| Recurso | Mínimo | Recomendado |
|:--|:--|:--|
| CPU | 1 vCPU | **2 vCPU** |
| RAM | 2 GB | **4 GB** (más si se usa BI con datos grandes) |
| Disco | 20 GB SSD | **40–50 GB SSD** |
| Tráfico/mes | < 10 GB | < 50 GB (las pantallas del ERP son livianas) |

Cualquier VPS de entrada de 4 GB cumple. La limitación real no es el tamaño sino la
**latencia** (§5).

## 5. La latencia importa: elegir región São Paulo

La "velocidad" del sistema en las cajas depende de qué tan lejos está el servidor.
Mediciones reales desde Asunción (WonderNetwork, sep-2026):

| Ubicación del servidor | Latencia desde Paraguay | Sensación en el ERP |
|:--|:--|:--|
| **São Paulo, Brasil** | **~19–21 ms** | Instantáneo, como hoy en red local |
| Miami / EE.UU. este | ~148–170 ms | Retardo perceptible al teclear |
| Europa (Alemania/Reino Unido) | ~210–260 ms | Lerdo para un POS interactivo |

**Regla práctica: elegir proveedor con región en São Paulo.** Todos los grandes la
tienen (AWS `sa-east-1`, Azure Brasil Sur, Google Cloud São Paulo, Oracle Cloud
`sa-saopaulo-1`) y varios VPS económicos también (Vultr, Hostinger, Contabo).
El favorito de la comunidad (Hetzner, §6.3) **no** tiene región en Sudamérica — por
eso aquí solo sirve para backups, no para las cajas.

## 6. Catálogo de opciones — de gratis a pago

Precios de lista aproximados a sep-2026 en USD (o EUR donde aplica); pueden variar y
suelen requerir **tarjeta de crédito internacional**.

### 6.1 Gratis permanente: Oracle Cloud Always Free — la única "en serio"

| | Detalle |
|:--|:--|
| Qué da | Servidores ARM: **2 vCPU / 12 GB RAM** + 200 GB disco, por US$0/mes, sin vencimiento (para cuentas nuevas; hasta jun-2026 eran 4 vCPU/24 GB — Oracle recortó la capa gratuita) |
| Región | Elegir **São Paulo** al crear la cuenta (la elección de "región home" es definitiva) |
| Para qué alcanza | El ERP completo con margen: 12 GB RAM es más que el recomendado |
| Condiciones | Tarjeta internacional para verificar identidad (no se cobra); riesgo conocido de que Oracle **reclame servidores "dormidos"** (un ERP usado a diario no es problema); el ARM de São Paulo a veces queda "sin capacidad" y hay que reintentar la creación |
| Comunidad | Consenso: es la única opción gratis apta para un servicio real; con reparos por los recortes de 2026 y la experiencia de signup |

**Veredicto:** excelente para la evaluación de 30 días o un negocio chico con
presupuesto cero, aceptando que "gratis" depende de las reglas de Oracle.

### 6.2 Gratis de plataformas PaaS (Render, Railway, Koyeb, Fly.io) — solo pruebas

Plataformas donde se sube el contenedor y ellas administran todo. Sus capas
gratuitas (Render duerme tras ~15 min sin tráfico; Railway ya solo da crédito de
prueba; Koyeb tiene un free tier que no expira) **no sirven para un ERP de uso
diario**: los planes gratuitos se pausan o tienen topes, y la base de datos habría
que separarla en un PostgreSQL gestionado gratuito (Supabase/Neon: ~500 MB,
se pausan por inactividad).

**Veredicto:** descartar para producción. Mencionados porque aparecen primero en
cualquier búsqueda de "hosting gratis docker".

### 6.3 Pago económico — VPS en São Paulo (US$5–25/mes)

Un VPS es una computadora alquilada con acceso completo: corre el mismo bundle
Docker de siempre. Esta es la categoría que la comunidad usa para auto-alojarse.

| Proveedor | Plan ejemplo (São Paulo) | Precio/mes | Notas |
|:--|:--|:--|:--|
| **Hostinger VPS Brasil** | KVM 1: 1 vCPU / 4 GB / 50 GB NVMe | ~US$5–6.50 promo (**renueva ~US$12**) | Mejor relación specs/precio con región local; ojo con el precio de renovación |
| **Vultr São Paulo** | Cloud Compute 1 vCPU / 1 GB → 4 GB | ~US$6 → ~US$18–24 | Mucha variedad de planes locales; paneles sencillos |
| **AWS Lightsail São Paulo** | Entrada ~US$5 (≈0.5–1 GB); 2 vCPU/4 GB ~US$20 | US$5–20 | El "VPS fácil" de Amazon; snapshots a US$0.05/GB-mes; Brasil puede tener pequeño recargo |
| **Contabo São Paulo** | 4 vCPU / 8 GB | ~US$5–8 (+ setup a veces) | Specs enormes por poco precio, pero red compartida y soporte lentos (reputación mixta en comunidad) |
| Hetzner (Alemania/EE.UU. — sin Sudamérica) | CX22: 2 vCPU / 4 GB | **€3.79** (ARM €3.29) | **El favorito absoluto de la comunidad** por precio/calidad — pero ~210–260 ms desde Paraguay: solo para backup/BI nocturno, no cajas |
| RackNerd y similares (EE.UU.) | 1–2 GB | ~US$10–12 **al año** | Calidad hobby; no recomendado para un ERP de negocio |

### 6.4 Pago "cloud grande" (US$10–40/mes)

DigitalOcean, Linode/Akamai, Vultr High-Frequency (~US$12 por 2 GB) o instancias
chicas de AWS/Azure/Google en São Paulo (~US$15–25 un equivalente a 2 vCPU/4 GB).
Se paga más por lo mismo que un VPS, a cambio de: snapshots/backups gestionados
muy cómodos, firewall y monitoreo integrados, soporte con SLA, y facturación
empresarial. Razonable cuando hay varias sucursales o se quiere el respaldo de un
proveedor grande.

### 6.5 Complementos

- **Backup fuera del servidor (casi obligatorio en nube):** snapshot diario del
  proveedor + copia semanal descargable (el propio bundle ya sabe hacerla). En un
  almacenamiento barato (Hetzner Storage ~€4, Backblaze B2 ~US$1/100 GB) cuesta
  centavos. Regla de la comunidad: *un backup que vive solo en el mismo proveedor
  no es un backup*.
- **PostgreSQL gestionado gratuito (Supabase/Neon):** no recomendado para el ERP
  (pausas y topes); mejor la base incluida en el propio servidor.
- **Coolify / Dokploy:** paneles libres que convierten un VPS en algo tipo-Heroku
  (deploy, SSL, BD con clics). Nota interna: opcional para el proveedor — el
  bundle ya trae su instalador y docker-compose.

## 7. Qué sugiere la comunidad (síntesis de la investigación)

De Reddit (r/selfhosted, r/hetzner), Hacker News y comparativas 2026:

1. **Para auto-alojar un stack Docker, un VPS es el estándar** — no las plataformas
   PaaS: más barato, sin límites, y el mismo `docker compose` que corre local.
2. **Hetzner es el consenso #1 en precio/calidad** (2 vCPU/4 GB por ~€3.79,
   "imposible de superar en valor de hardware") — con la salvedad regional: no
   tiene Sudamérica.
3. **Oracle Always Free es la única capa gratuita "para producción"** (2 vCPU/12 GB
   tras el recorte de junio-2026), con reparos conocidos: reclamo de instancias
   dormidas, altas de cuenta con problemas y capacidad ARM fluctuante.
4. **Las capas gratuitas de PaaS (Render/Railway/Fly) se consideran para demos**,
   no para servicios serios: se duermen o caducan.
5. **Coolify/Dokploy sobre VPS es la moda** para quien quiere experiencia Heroku
   sin pagarla.

Fuentes concretas en §11.

## 8. Recomendación por escenario

| Escenario del cliente | Opción recomendada | Costo orientativo |
|:--|:--|:--|
| **Evaluación 30 días / negocio chico, presupuesto cero** | Oracle Always Free São Paulo (si hay tarjeta internacional y se consigue capacidad ARM) | US$0 |
| Evaluación sin tarjeta internacional | Lightsail o Vultr São Paulo 1 mes | ~US$5–6 |
| **Negocio real, 1–2 sucursales** | VPS São Paulo 2 vCPU / 4 GB (Hostinger KVM 1, Vultr o Lightsail) + snapshot diario + backup externo | **US$10–25/mes** |
| Multi-sucursal, BI con datos grandes | Cloud grande en `sa-east-1` o VPS 8 GB, con backups gestionados | US$25–50/mes |
| **Hoy, sin cambiar la operación** (recomendado a corto plazo) | Mantener on-premise (estrategia implementada) + **backup nocturno a la nube** barata | ~US$0–5/mes |

La última fila es la recomendación actual del proveedor: por ~US$0–5/mes el cliente
gana resiliencia contra incendio/robo/disco sin cambiar nada de su operación diaria,
y la decisión de "todo en la nube" queda para cuando el cliente la pida por
necesidad (sucursales lejos, personal remoto, cese de dependencia del local).

## 9. Preguntas frecuentes

- **¿Se pierde la licencia comprada si nos vamos a la nube?** No: la licencia es
  perpetua y viaja con la instalación, igual que en la migración laptop → servidor.
- **¿Los datos siguen siendo nuestros?** Sí. Están en el servidor alquilado y se
  pueden exportar/restaurar en cualquier momento con el backup del bundle. Elegir
  la región São Paulo implica que físicamente viven en Brasil.
- **¿Qué pasa si se corta el internet del local?** Con el sistema en la nube, las
  cajas no facturan hasta que vuelva el enlace (recomendamos respaldo 4G). Es el
  único punto donde on-premise es más robusto.
- **¿Y si no pagamos la mensualidad?** El proveedor de nube suspende el servidor;
  los datos suelen retenerse un tiempo y se recuperan al ponerse al día. Nunca
  depender de una sola copia dentro del mismo proveedor (§6.5).
- **¿Se puede imprimir en el local?** Sí: las terminales del local imprimen igual
  (la impresora sigue al lado de la caja). Los teléfonos siguen sin imprimir.
- **¿Podemos volver del cloud al local después?** Sí: mismo formato de backup y
  restore que usa la migración entre máquinas.
- **¿Quién administra el servidor?** El proveedor del ERP, con el mismo instalador
  y acceso remoto (Tailscale sigue siendo útil para soporte).

## 10. Qué falta para ofrecer esta modalidad (pendiente)

Este documento es un catálogo de opciones, no un procedimiento. Si un cliente elige
la nube, antes se debe implementar y documentar (como REQ_DEPLOY + guía operativa):

1. Guía de instalación del bundle en VPS Linux con **HTTPS público** (Let's
   Encrypt) en lugar de la CA interna.
2. **Backups automáticos** (snapshot + copia externa descargable) y práctica de
   restore.
3. Monitoreo mínimo (caída del servidor, disco lleno) y actualizaciones remotas.
4. Decisión comercial: quién contrata y paga el servidor, precio al cliente y SLA
   de soporte.

## 11. Fuentes (investigación sep-2026)

- [r/selfhosted — Cheap VPS for a small self-hosted setup](https://www.reddit.com/r/selfhosted/comments/1uk3dua/cheap_vps_for_a_small_selfhosted_setup/)
- [r/selfhosted — Best VPS for small projects](https://www.reddit.com/r/selfhosted/comments/1uxit6o/best_vps_for_small_projects/)
- [r/hetzner — Best value shared vCPU VPS at ~10€/month](https://www.reddit.com/r/hetzner/comments/1e896u4/best_valueformoney_shared_vcpu_vps_at_10month/)
- [dev.to — Self-Hosting on a Budget: Best VPS Providers Tested 2026 (Hetzner vs DO vs Linode vs OVH)](https://dev.to/enfernandes/self-hosting-on-a-budget-best-cheap-vps-providers-tested-hetzner-vs-digitalocean-vs-linode-vs-ovh-2f8b)
- [Oracle Cloud Free Tier — documentación oficial](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm) y cobertura del recorte de ARM A1 a 2 OCPU/12 GB (jun-2026): [qatrial.com](https://qatrial.com), [draft-publish.com](https://draft-publish.com)
- PaaS free tiers: [freetiers.com — Koyeb Free Tier](https://freetiers.com), [stackformakers.com — Fly vs Railway vs Render](https://stackformakers.com)
- Coolify/Dokploy: [dokploy.com — 9 Best Heroku Alternatives 2026](https://dokploy.com/blog/heroku-alternatives), [bitdoze.com — Coolify vs Dokploy vs Kamal 2](https://www.bitdoze.com/coolify-vs-dokploy-vs-kamal-2/)
- Precios VPS São Paulo: [vpsarena.app — Brazil/São Paulo plans](https://vpsarena.app), [dohohub.com — Hostinger vs Vultr live prices](https://dohohub.com), [whichdevtool.com — Lightsail pricing 2026](https://whichdevtool.com)
- Latencias desde Asunción: [wondernetwork.com — Asunción→São Paulo](https://wondernetwork.com/pings/Asuncion/Sao-Paulo), [→Miami](https://wondernetwork.com/pings/Asuncion/Miami), [→todas las ciudades](https://wondernetwork.com/pings/Asuncion)

## 12. Historia de cambios

| Versión | Fecha | Cambio | Autor |
|:--|:--|:--|:--|
| v0.1 | 2026-09-30 | Primera versión: catálogo de opciones de nube (gratis → pago) con investigación de comunidad, latencias desde Paraguay y recomendación por escenario. Estado: propuesta, sin procedimiento implementado | renpereiradx + agente |
