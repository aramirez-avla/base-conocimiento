# Guía maestra para agentes: stack, buenas prácticas, migración y despliegue en Horizon (AVLA)

> **Versión:** 1.0 · **Consolidada:** 2026-11-09 · **Autor de la consolidación:** equipo IA AVLA (aramirez@avla.com)
> **Vigencia de los hechos de plataforma:** verificados hasta 2026-11-06. Lo no verificado está marcado `SUPUESTO`.
> **Fuentes:** ver §17. Este documento **reemplaza como punto de entrada** a los documentos dispersos; no los borra.

---

## 0. Cómo usar este documento (léelo primero si eres un agente)

Este documento es la **fuente única** para que un agente de IA pueda:

1. **Entender el contexto** de cualquier repo de AVLA que se despliega (o se va a desplegar) en Horizon.
2. **Armar planes de migración** (Lovable / Supabase / Apps Script / n8n / FastAPI+Angular → Horizon) alineados a la práctica real.
3. **Escribir código** que cumpla los lineamientos y pase las puertas automáticas.

### 0.1 Orden de lectura según la tarea

| Tarea | Lee, en este orden |
|---|---|
| Escribir o modificar código en una app Horizon | §0.3 → §4 (contrato) → §5 (buenas prácticas) → §3 (perfil de stack) → §12 (prohibido / requiere aprobación) |
| Planificar una migración | §1 → §2 → §8 (metodología + tabla de reemplazos + plantilla) → §4 → §12 → §15 |
| Desplegar o diagnosticar un despliegue | §2 → §6 (paso a paso) → §11 (fricciones) → §13 (checklists) |
| Integrar IA (LLM) | §7.1 (gateway) → §5.4 (seguridad/LGPD) |
| Sincronizar cambios desde Lovable | §9 (upstream-sync) → §4 → §5 |

### 0.2 Precedencia cuando algo choca

1. **Hechos de plataforma verificados** (§2, §4): no se negocian; si el código los contradice, el código está mal.
2. **Este documento**, que ya resolvió las contradicciones entre fuentes (tabla en §14).
3. `docs/LINEAMIENTOS-AVLA.md` de `avla-tech/lab-qa-ia-lab` (lineamientos Horizon v2), cuando trate algo que aquí no esté.
4. `AGENTS.md` / `CLAUDE.md` del repo, para **decisiones propias de esa app** (no pueden relajar §4 ni §12).
5. El código existente del repo («imita el destino antes que inventar»).

Si encuentras un conflicto no resuelto aquí: **no lo resuelvas en silencio**. Aplica la opción más segura (fail-closed), y repórtalo en el PR / en tu respuesta.

### 0.3 Reglas de conducta del agente (siempre)

- **No inventes.** Ni comandos de build (léelos de `package.json` / `pyproject.toml` / CI), ni nombres de variables, ni reglas de negocio, ni datos. Lo que no sabes traducir va a «Requiere decisión humana».
- **Cambios mínimos y coherentes** con el estilo y el **idioma del repo** (si el repo usa `obtenerToken`, no introduzcas `fetchToken`).
- **Nunca** hagas commit, push, deploy, ni apliques migraciones contra una base real sin pedido explícito del humano.
- **Nunca** borres, saltes (`skip`, `xit`, `@Disabled`) ni debilites pruebas para que pase algo.
- **Reporta resultados reales** de build/test. No digas «pasó» si no lo corriste. Distingue CONFIRMADO vs SOSPECHA y cita `archivo:línea`.
- **No reduzcas alcance en silencio**: si algo no se puede, dilo.
- Con usuarios no técnicos: lenguaje simple, sin jerga, en su idioma. Si piden algo que viola los lineamientos: no lo implementes, explica en 1–2 frases y ofrece alternativa.
- Mantén al día la sección **«Decisiones de esta app»** del `AGENTS.md` del repo cuando tomes una decisión de diseño.

---

## 1. Contexto: qué es Horizon y qué se despliega ahí

### 1.1 Horizon

- **Horizon** es el *Internal Developer Portal* de AVLA para el **laboratorio de IA** (`lab-qa`): `https://lab-qa-idp.avla.cloud`.
- Construye una imagen Docker desde un repo GitHub de la organización **`avla-tech`** (repo **privado**) y la corre en EKS (clúster `lab-qa-ia-eks`, namespace `lab-qa-poc`).
- Cada app queda en su propio subdominio: `https://<nombre-pod>.lab-qa.avla.cloud`, servida **en la raíz** del host (sin subpaths).
- Cada app queda detrás de un **portero**: sidecar `oauth2-proxy` (puerto 4180) con login Google `@avla.com`, que reenvía a la app en el **puerto 8080**.
- Menú del portal: Inicio, Deploy App, Acceso a DB, Acceso a mi app (`/apps/acceso-app`), Eliminar app (`/undeploy`), Mis apps, Monitor, Ayuda.

### 1.2 Dos mundos normativos en AVLA (no los confundas)

| | **Horizon / apps del laboratorio IA** (este documento) | **Plataforma core / TI** (referencia) |
|---|---|---|
| Qué | Apps internas hechas con IA (Lovable, Claude Code), mini-apps, hub, gateway IA | Sistemas core de seguros (pólizas, endosos, etc.) |
| Stack | TanStack Start · Node/TS monorepo (NestJS/Fastify + React/Vite) · FastAPI + Angular | Java 25 / Spring Boot 4.1, Angular 22, Node 24 / NestJS 12, PostgreSQL 18, MongoDB 8 |
| VCS / CI | GitHub `avla-tech` + GitHub Actions + build de Horizon | Bitbucket `avla_ti` + Jenkins + Nexus + SonarQube (migrando a GitHub Actions según Manual) |
| Identidad | Portero oauth2-proxy (`X-Forwarded-Email`) | JWT RS256 / SSO corporativo |
| Normas | Este documento + `LINEAMIENTOS-AVLA.md` v2 | Manual de Ingeniería + lineamientos Confluence AYD + `avla-proyecto-modelo` |

Las **buenas prácticas generales** del mundo core (pruebas de comportamiento, dinero en decimal, LGPD, sin `any`, etc.) **también aplican** en Horizon (§5). Lo que **no** aplica en Horizon es su infraestructura (Jenkins, Nexus, Vault, `/api/v1` obligatorio, Liquibase en pipeline).

### 1.3 Inventario de apps (a 2026-11-06)

| App | Repo | Stack | Capacidad | Estado |
|---|---|---|---|---|
| Portal de Fianzas (Perú) | `lab-qa-ia-lab-01` | NestJS/Fastify + React + PG + pg-boss | B | Desplegado — **mejor caso documentado de migración** |
| Dashboard Buk / capacitación | `lab-qa-ia-lab-02` | FastAPI + PG | A | Desplegado (portero activo al 06-11) |
| Cotador Property (subscrição) | `lab-qa-ia-lab-03` (ver también `Avla-subscricao-web`, rama divergente) | FastAPI + PG + Alembic | B | Desplegado |
| GEMINI (plantilla) | `lab-qa-ia-lab-04` | nginx estático / Alembic de ejemplo | — | Plantilla |
| (Playwright) | `lab-qa-ia-lab-05` | Python + Playwright/Chromium | B | Documentado |
| Formateador (Lovable) | `lab-qa-ia-lab-06` | TanStack Start (Lovable) | A | Migrado 06-11 — **referencia del perfil TanStack** |
| DeployBoard | `lab-qa-ia-lab-08` | Node monorepo (api/web/shared) + PG | — | **Estándar más reciente** (06-11) + PoC upstream-sync |
| Minihub | `avla-miniapps-hub` | FastAPI + Angular | A | Desplegado |
| Gateway IA | `gateway-ia-avla` | FastAPI, SQLite en `/tmp` | — | Desplegado (`lab-qa-gateway-ia`) |
| Carpeta Tributaria | `carpeta-tributaria` | FastAPI + Angular | B | Desplegado |
| Spread Financeiro | `spread-financeiro` | FastAPI + Angular + LibreOffice + poppler | C | Rama `standalone` |
| Spread Seguro Crédito | `spread-seguro-credito` | FastAPI + Angular + poppler | B | Rama `feat-async` |
| Consultas Externas | `consultas-externas` | FastAPI + Angular | — | Dockerfile listo; bloqueado por red a APIs internas |

---

## 2. Arquitectura de la plataforma (hechos verificados)

### 2.1 Camino de red

```
Navegador ─► Cloudflare ─► CloudFront (por app) + WAF compartido `lab-qa-ia-common` ─► ALB ─► Pod
                                                                                         ├─ oauth2-proxy :4180 (portero)
                                                                                         └─ app :8080
```

Consecuencias duras:

| Límite | Valor | Qué hacer |
|---|---|---|
| Timeout de CloudFront | **~30 s** → `504` | Todo trabajo que pueda pasar de ~10 s: **`202` + polling** (§4.6). Diseña requests síncronas ≤ 10 s. |
| WAF (AWS CommonRuleSet + SQLi) | Bloquea cuerpos **> 8 KB**, **GET con body**, requests **sin `User-Agent`**, texto que parece SQLi (comillas) → `403` con HTML de CloudFront | Binarios **directo a S3 con POST/PUT prefirmado**; nada de base64 en JSON; nunca body en GET; clientes de backend siempre con `User-Agent`. |
| Réplicas | **Siempre 1** | No asumas escalado horizontal; el polling llega al mismo pod. |
| Disco del pod | **Efímero** (se pierde al reiniciar o cambiar una variable) | Nada importante en `/tmp`, SQLite ni memoria. |
| Red a APIs internas de AVLA | **Aislado** (tener internet no prueba lo contrario) | Probar desde el pod; usar el API-Gateway público; o pedir ruta a DevOps. |
| Llamadas backend→backend entre apps | Salen por URL pública y **chocan con el portero** (`302` a Google) | Solo dos excepciones soportadas: gateway IA (Bearer) y hub (cookie) — §7. |

### 2.2 Portero (oauth2-proxy) e identidad

- Login Google `@avla.com`. Hasta **5 personas** nominadas por app, con vencimiento en días. El dueño siempre entra.
- El portero **se despliega al otorgar el primer acceso**. Antes de eso la URL debe considerarse **abierta** y el header de identidad **falsificable** → conceder el primer acceso **inmediatamente** después del deploy.
- **Tipo de acceso** (página «Acceso a mi app»):
  - **«Solo login con Google»** ← **el que usan las apps**. La app recibe el correo en **`X-Forwarded-Email`**. Horizon no toca `Authorization`.
  - «Login con Google + token del endpoint» ← para **servicios consumidos por programas** (el gateway IA). El portero ocupa `Authorization: Bearer <token de Horizon>` y rechaza JWT propios.
- Cookie del portero en el **dominio padre `.lab-qa.avla.cloud`**: un login sirve para todas las apps; «Cerrar sesión» (`/oauth2/sign_out`) cierra todas; las apps del lab son *same-site* entre sí → protege CSRF (§5.4).
- El TLS termina antes del pod: el pod ve `X-Forwarded-Proto: http`. Usa `--proxy-headers` (uvicorn) / `trustProxy` (Fastify).
- Error `403` con página «Secured with OAuth2 Proxy» = la cuenta no está entre las personas con acceso o su acceso venció → el dueño la agrega en `/apps/acceso-app`. (Distinto del 403 HTML de CloudFront = WAF.)

### 2.3 Capacidad (recursos por pod)

| Cap. | Request | Límite | Cupo del lab (tope 10) | Usar para |
|---|---|---|---|---|
| **A** | 50m / 64 Mi | 300m / **256 Mi** | 1 | CRUD, dashboards, hub |
| **B** | 200m / 256 Mi | 500m / **512 Mi** | 4 | Procesa PDF/Excel moderado, generación documental |
| **C** | 400m / 512 Mi | 1000m / **1024 Mi** | 8 | LibreOffice, Chromium, PDFs grandes |

- **Mide el pico de memoria en local con el caso más pesado real** antes de elegir. Referencias: PDF de 48 págs. = 343 MiB (OOMKilled en A); Property ~576 MiB (en B con `*_MAX_JOBS=1`); React-PDF extremo ~190 MB.
- Limita concurrencia con una variable `<APP>_MAX_JOBS` (por defecto 1–2) y responde `503 {error:"saturado"}` al superarla.

### 2.4 PostgreSQL

- Horizon crea la base **`lab_qa_<app>`** con usuario **`usr_lab_qa_<app>`** en el RDS compartido **`lab-qa-ia-apps`** (PostgreSQL 16).
- **La base se crea DESPUÉS de levantar el pod** y **las credenciales NO se inyectan**: se cargan a mano en el portal cuando llega la ficha (Acceso a DB).
- Variables: `PGHOST`, `PGPORT=5432`, `PGDATABASE`, `PGUSER`, `PGPASSWORD`, **`PGSSLMODE=require`** (el RDS exige TLS; sin esto, Node `pg` reintenta para siempre con código `28000`), **`PG_POOL_MAX=3`** (RDS compartido por todo el lab).
- **Usa `PG*` separadas**, nunca una `DATABASE_URL` armada a mano (caracteres `@ : / ? # %` en la clave la rompen; ya hubo incidentes). Soportar `DATABASE_URL` como alternativa solo para local/compat.

### 2.5 Almacenamiento S3

- Formulario: «Objects» (usar si la app guarda archivos), «Frontend» (**no usar**: la SPA va en la imagen) o «No necesita bucket».
- Con «Objects», Horizon **sí inyecta** (verificado en lab-03): **`S3_BUCKET_OBJECTS`, `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`**. Lee esos nombres (no `BUCKET_NAME`).
- El bucket existe **recién después del primer deploy** (no se pueden pre-cargar semillas).
- El CORS del bucket solo admite el origen de la app en Horizon, y **se ha perdido en redeploys** (y la URL de la app puede cambiar): tras cada deploy verifica el preflight con `OPTIONS` + `Origin` + `Access-Control-Request-Method` (Node no hace preflight, así que un script Node «funciona» aunque el navegador falle).

### 2.6 Variables de entorno (formulario del portal)

- Formato: `CLAVE=valor;CLAVE=valor` en un solo campo.
- Cada valor: **1–256 caracteres imprimibles**, **sin espacios**, **sin `;`** (corta la variable sin aviso; si es inevitable, `%3B`), **nunca vacía** (`HUB_URL=` es rechazada).
- **No caben** llaves PEM (288–408 chars) ni JSON (se trunca). Divide en varias variables o empaqueta como archivo en la imagen.
- **Cambiar una variable reinicia el pod pero NO reconstruye la imagen**. Para tomar un commit nuevo hay que **redesplegar**.
- Secretos: solo en el portal. Nunca en repo, imagen, logs, chats ni mensajes al usuario.
- **Regla de oro:** la imagen **arranca y funciona sin ninguna variable** cargada en Horizon. Todo valor necesario tiene defecto en el `Dockerfile` o en el módulo de config; lo opcional degrada con motivo en `/readyz`.

---

## 3. Stack aprobado por perfil (Horizon)

Elige **un** perfil. Si el repo ya tiene uno, **imítalo**. Un stack fuera de estos perfiles requiere ADR + aprobación de Arquitectura.

### 3.1 Perfil `node-tanstack-lovable` (apps nacidas en Lovable)

Referencia viva: `lab-qa-ia-lab-06`. Plantilla: «Plantilla base Avla 1/2 y 2/2» de `avla-tech/lab-qa-ia-lab` (no la reescribas: construye encima).

| Capa | Elección |
|---|---|
| Front | React (SPA) con TanStack Start/Router |
| Backend | Rutas `/api/*` de TanStack Start (Node 22 + TypeScript estricto) en el mismo proyecto |
| Validación | **zod** en cada endpoint y en `config.ts` |
| Datos | `s.store` con dos módulos: **básico** `APP_DATA=disk` / `APP_FILES=disk` (estado transitorio con TTL `STATE_TTL_MIN`, un pod) o **permanente** `APP_DATA=postgres` (Drizzle ORM + `pg`) / `APP_FILES=s3` (`@aws-sdk/client-s3` v3, prefirmado ≤ 15 min) — **el permanente requiere aprobación** |
| Pruebas | Vitest (`src/test/`), `node --test tests/*.mjs` para lógica pura; PGlite solo en vista previa (`import.meta.env.DEV`) |
| Librerías frecuentes | `unpdf` (PDF), `exceljs` (Excel; **no** `xlsx`, tiene CVE) |
| Build | `node:22-alpine`, **`npm install -g npm@11.21.0`** (npm 10 falla con `reading 'edgesOut'`), **`ENV NITRO_PRESET=node-server`** antes del build (si no, genera para Cloudflare), runtime copia solo `.output/`, `CMD ["node",".output/server/index.mjs"]` |

Estructura obligatoria:

```
src/server/config.ts      # ÚNICO lector de process.env. zod, todo con defecto. Nunca loguea valores.
src/server/http.ts        # json, apiError, log (JSON por línea), safe (sin stack traces)
src/server/auth.ts        # currentUser, requireUser, requireAdmin, puedeVer (dueño)
src/server/procesos.ts    # iniciarProceso, ErrorUsuario (contrato ExecutionProcess)
src/server/api.ts         # entrada del backend; reexporta lo anterior y el store
src/server/store/disk.ts  # (o postgres.ts con LAS MISMAS funciones; se cambia el export en api.ts)
src/routes/healthz.ts · src/routes/readyz.ts · src/routes/api/me.ts
src/routes/api/getExecutionProcess.$idProcess.ts
src/lib/limites.ts        # límites compartidos cliente/servidor (una sola vez)
src/test/plantilla.test.ts  # debe seguir en verde
```

- Cada ruta `/api/*` hace `const s = await import("@/server/api")` **dentro del handler** y envuelve todo en `s.safe(...)`.
- **Prohibido:** `index.html` en la raíz (Nitro lo usa como renderer e intercepta todo), editar `src/routeTree.gen.ts`, portar `.lovable/` o config Cloudflare/Wrangler, quitar `@lovable.dev/vite-tanstack-config`.
- No reescribas la historia de una rama conectada a Lovable. No uses «Publish» de Lovable para entregar.

### 3.2 Perfil `node-monorepo` (npm workspaces) — **estándar más reciente**

Referencias: `lab-qa-ia-lab-08` (más nuevo) y `lab-qa-ia-lab-01` (más completo; ADRs).

| Capa | Elección |
|---|---|
| Lenguaje | TypeScript estricto en todo el repo (ADR-001; *pendiente de validación formal por Arquitectura*) |
| API | NestJS 11 sobre Fastify 5 (override `fastify ^5.12.7` por CVE) o Fastify directo; config con zod |
| Web | React 19 + Vite 7 + React Router 7; servida por la API desde `WEB_DIST_DIR` |
| Compartido | `packages/shared` (tipos, esquemas zod, reglas); se compila primero |
| Datos | PostgreSQL + Drizzle ORM (`drizzle-kit generate`, SQL revisado y versionado; **nunca `drizzle-kit push`** contra un ambiente) |
| Colas | **pg-boss** sobre el mismo PostgreSQL (no Redis) |
| PDF | `@react-pdf/renderer` en proceso hijo (heap 256 MB, timeout 60 s, SIGKILL); plan B Playwright tras el puerto `PdfRenderer` (ADR-002) |
| Logs | pino (JSON); `pino-pretty` solo en dev |
| Pruebas | Vitest; integración con **testcontainers** (PostgreSQL, S3 con `adobe/s3mock`) |
| Arquitectura | Monolito modular hexagonal: `domain` (sin imports externos) → `application` (casos de uso + puertos) → `infrastructure` (adapters) → controladores sin reglas de negocio; `dependency-cruiser` lo verifica |

```
packages/shared/            # se compila primero
apps/api/src/infrastructure/config/config.ts   # ÚNICO lector de process.env
apps/api/migrations/  (o apps/api/drizzle/)     # migraciones numeradas; nunca editar una aplicada
apps/web/
deploy/horizon.env.example  # manifiesto de variables (§6.6)
Dockerfile                  # en la raíz
```

- Un workspace nuevo exige su `COPY` en **ambas** etapas del Dockerfile. Las dependencias van en el `package.json` del workspace (no en la raíz, salvo tooling).
- `VITE_*` con secretos → pasan a la API.

### 3.3 Perfil `python-angular` (FastAPI + Angular) — *borrador vivo*

Referencias: `carpeta-tributaria`, `avla-miniapps-hub`, `lab-qa-ia-lab-02`, `lab-qa-ia-lab-03`.

| Capa | Elección |
|---|---|
| Backend | Python 3.12, FastAPI, Pydantic; uvicorn **1 worker** con `--proxy-headers` |
| Front | Angular (standalone), compilado en una etapa del Dockerfile y servido por FastAPI desde `<APP>_SPA_DIR` |
| Datos | PostgreSQL + SQLAlchemy + **Alembic** con candado (`pg_advisory_lock`) |
| Pruebas | pytest (+ cobertura), ruff (format + check), mypy si está configurado; front: `ng lint`, karma, Playwright + axe para e2e |
| Estructura | `api/` (rutas) · `app/` (orquestación) · `core/` (dominio determinístico) · `infra/`; reglas de negocio como funciones puras (+ `regras.yaml` y casos golden en `tests/golden/` cuando aplique) |

- **Dos modos de bundle sin `--build-arg`:** la imagen compila los dos bundles (p. ej. `/app/spa-solo` y `/app/spa-portal`) y una variable `<APP>_SPA_DIR` elige en runtime. **Por defecto, el bundle local/solo** (el de producción exige token del hub y en el lab no hay emisor). No uses `ARG SPA_CONFIGURATION` (obliga a reconstruir).
- La API también se publica bajo el prefijo que llama el bundle (`/<app>-api`).
- No toques los Dockerfiles de producción existentes (`backend/Dockerfile`, `frontend/Dockerfile`, Jenkins + nginx).

### 3.4 Desvíos

- Cualquier desvío de stack → **ADR** en `docs/adr/NNNN-titulo.md` (MADR simplificado, inmutable; se reemplaza con «supersede»). Ejemplo existente: spread en Python con ADR 0002.
- Versiones: usa majors publicadas hace meses (no la última del día). Upgrade de un major por PR.

---

## 4. Contrato de la app en Horizon (OBLIGATORIO)

Una app que no cumple esto **no despliega** o **falla en runtime**. Las puertas automáticas (§10) verifican gran parte.

### 4.1 Imagen

1. **Un solo `Dockerfile`, en la raíz.** Horizon no lee otros ni pasa `--build-arg` (ningún `ARG` sin valor por defecto).
2. Multi-stage: build → **runtime como última etapa** (las etapas de test van antes).
3. Si hay frontend, se compila en una etapa y **lo sirve el backend** (una imagen por app).
4. `ENV PORT=8080` + `EXPOSE 8080`, y el proceso escucha `${PORT}`. (Con 8000 → `502` en ~1 ms.)
5. Usuario no-root **UID 10001, grupo 0**: `chown 10001:0` y `chmod -R g=u` en directorios escribibles; `USER 10001:0`.
6. `HEALTHCHECK` **sin `curl`** (Node: `node -e "fetch(...)"`; Python: `python -c "import urllib.request..."`) contra `/healthz`.
7. Runtime solo con dependencias de producción; sin npm/npx/corepack/yarn/curl (Trivy). `tini` como PID 1 si hay subprocesos (Python, Chromium, LibreOffice).
8. Fija la imagen base (digest recomendado; mínimo tag de versión).
9. `.dockerignore` con `node_modules`, `.output`, `dist`, `.git`, `.env*`, datos.

### 4.2 Salud y arranque en dos tiempos

- **`GET /healthz`** → `200` **siempre** que el proceso viva. **Nunca** toca base, disco ni terceros.
- **`GET /readyz`** → `200` si está lista; si no, `503` con `{ "estado": "...", "motivo": "..." }` nombrando qué falta. Nunca expone secretos.
- **Arranque sin base:** la app levanta HTTP de inmediato; el resto de la API responde `503 servicio_iniciando` mientras tanto. En segundo plano: espera `PG*` → conecta con backoff (1 s → 30 s) → verificación previa (versión, privilegio `CREATE`, extensiones) → **toma advisory lock** → migra → seeds idempotentes → `lista`. **Sin reiniciar el pod.** Un fail-fast por falta de base termina en CrashLoop.
- Vocabulario estándar de estados/motivos de `/readyz`:
  `esperando_base_datos` · `conectando` (+ motivo `clave_o_usuario_incorrecto` | `base_inexistente` | `host_desconocido` | `host_inalcanzable` | `ssl` | `otro`) · `migrando` · `permisos_insuficientes` · `base_no_compatible` · `error_migracion` · `bucket_no_configurado` · `bucket_no_disponible` · `fuente_no_configurada` · `lista`.
- Recomendado (plan de mitigación): log `FALTA_VARIABLE nombre=<X> para=<qué>` por cada variable faltante al arrancar, y un endpoint de diagnóstico tipo `/saude` con `{app, version, base{ok,motivo}, gateway{ok}, hub{ok}, bucket{ok}, memoria{limite_mb,pico_mb}, faltan_variables[]}`.

### 4.3 Configuración

- **Un único módulo** lee el entorno (`config.ts` / `config.py`); validado con zod / Pydantic; todo con defecto seguro u opcional. `process.env` / `os.environ` / `import.meta.env` **fuera de ese módulo está prohibido** (lo detecta `chequeo-horizon.sh`).
- Nunca registra valores, solo nombres.
- IDs, URLs, rangos, TTL fijos en el código → variable con el valor actual como defecto + `.env.example` / manifiesto + README.
- Cumple §2.6 (≤ 256, sin espacios, sin `;`, sin vacías, sin JSON/PEM).

### 4.4 Identidad y autorización

- La app **no implementa login** (ni contraseñas, ni OAuth, ni JWT propio, ni cookies de sesión). Lee el correo de **`X-Forwarded-Email`** (configurable por `PORTERO_HEADER`, default ese; dominios permitidos `PORTERO_DOMINIOS=avla.com`).
- **Todo endpoint de datos exige usuario** (`requireUser`) salvo `/healthz` y `/readyz`. Sin header válido → `401` con mensaje de qué revisar. **Nunca un usuario genérico fuera de modo desarrollo.**
- Rol: `admin` si el correo está en **`ADMIN_EMAILS`** (coma, sin espacios); si no, `usuario`. Roles más finos → tabla en la base, con el primer acceso creando al usuario con rol mínimo.
- Rol y propiedad se verifican **en el servidor** en cada endpoint. Recurso de otro usuario → **`404`** (no 403). Modificar datos ajenos → `requireAdmin`.
- `/api/me` devuelve correo y rol (y `dev: true` en modo dev).
- **Modo desarrollo / bypass:** `AVLA_AUTH_DISABLED=1` (Python) o `AUTH_MODE=dev` (Node) **solo se acepta con `AVLA_ENV` ∈ {`dev`,`local`,`test`}**; con cualquier otro valor el proceso **aborta al arrancar**. Una mini-app suelta puede traerlo de fábrica en su Dockerfile; **un portal/hub nunca**.
- Sin botón propio de cerrar sesión; si lo piden, enlazar a `/oauth2/sign_out` (cierra todas las apps del lab).
- Endpoint de diagnóstico de identidad (`/diagnostico/identidad`) solo con flag `DIAGNOSTICO_IDENTIDAD=1`, mostrando únicamente headers de identidad (sin `Authorization`/`Cookie`, enmascarando JWT). **Apagar tras el primer deploy.**
- Un portero vencido responde `302`: el front usa `fetch(..., { redirect: 'manual' })` y lo trata como `401` (recargar para re-login).

### 4.5 Datos y migraciones

- Fuente de verdad: **PostgreSQL** (datos) y **S3** (archivos). Estado transitorio en disco solo con TTL y aceptando su pérdida.
- Migraciones:
  - **Idempotentes, con candado** (`pg_advisory_lock` / Alembic con lock), **aplicadas por la app** al detectar la base, no por el entrypoint ni por el agente.
  - Numeradas, versionadas, **nunca se edita una ya aplicada**. Preferir **aditivas**; con datos reales, *expand/contract*.
  - Declarar si son reversibles; en Python probar `upgrade head` → `downgrade base` y `alembic check` (modelo vs migración) como en `lab-qa-ia-lab-04/tests/bd/test_migraciones.py`.
  - Seeds que dependen de variables (p. ej. admin inicial) **no van en el SQL**: seeds idempotentes (`ON CONFLICT DO NOTHING`) dentro del lock en cada arranque.
  - Una migración fallida → estado `error_migracion`, transacción revertida, sin workers.
  - La creación de colas (pg-boss) va **dentro** del lock (deadlock con varias instancias).
  - Deadlocks y errores de serialización = transitorios (reintentar).
- Errores de conexión **clasificados** en log y `/readyz`; nunca reenviar el mensaje crudo del driver.
- Columnas comunes recomendadas: `id uuid`, `created_at`, `updated_at`, `created_by`, `version` (agregados, para concurrencia optimista → `409`). Prefijo de tabla por módulo (`cli_`, `op_`, `doc_`, `iam_`…).
- Dinero: **nunca `float`/`number`**. `NUMERIC` en base; `Decimal` / `decimal.js`; string decimal en el transporte.

### 4.6 Procesos largos: contrato ExecutionProcess (inmutable)

Todo procesamiento de archivos o trabajo que pueda pasar de ~10 s es **asíncrono**, aunque hoy parezca rápido.

```ts
export const ExecutionProcessStates = { CREATED: 1, PROCESSING: 2, FINISHED_SUCCESS: 3, FINISHED_ERROR: 4, SCHEDULED: 5 } as const;
export const TERMINALES = [3, 4];
```

- `POST /api/<recurso>/...` → `202 { idProcess }` (validación síncrona → `400`).
- `GET /api/getExecutionProcess/{idProcess}` → `{ id, status, progress_data, output_data }` (los dos últimos son **strings con JSON**). `output_data` lleva solo referencias (ids), nunca el binario. Error: `{"errorDetail":"<frase para la persona>"}`.
- El registro se crea **antes de encolar** (el primer polling nunca da 404). Todo desenlace es terminal (un `.catch` siempre escribe `4`). Guardia de «sin noticias» (> ~5 min sin avance ⇒ `4` «El proceso se interrumpió, vuelva a intentarlo»); si puede durar > 5 min, reporta avance al menos cada minuto. TTL 24 h con purga. Proceso ajeno / inexistente / malformado → `404`.
- Front: polling **cada 2 s** con progreso y botón para dejar de esperar; se detiene en 3 o 4.
- Trabajo pesado en **proceso hijo** con heap acotado + timeout + SIGKILL + semáforo (`servicio_saturado`).

### 4.7 HTTP, errores y logs

- Rutas `/api/*` (sin versión en Horizon; `/api/v1` aceptable si el repo ya lo usa).
- Errores: `{ "error": "<codigo_estable>", "message": "<texto para la persona>" }`. **Nunca stack traces** ni errores crudos de terceros (detalle solo al log).
- Códigos: `400` validación · `401` sin identidad · `403` sin rol · `404` no existe o es ajeno · `409` conflicto/versión · `413/415` archivo · `202` proceso · `503 saturado | servicio_iniciando | fuente_no_configurada`.
- Paginación en servidor: `page`, `pageSize ≤ 100`, respuesta con `total`.
- Logs: una línea JSON por evento (`log()`), **sin** credenciales, tokens, cookies, contenido de archivos ni datos personales. En procesos: tipo de error + `idProcess`.
- CSRF (cookie compartida same-site): validar `Sec-Fetch-Site: same-origin` (con `Origin` de respaldo) en métodos que modifican.
- CSP estricta (sin `style="..."` inline → clases).

### 4.8 Archivos

- Subida **directa navegador → S3** con POST prefirmado (tamaño exacto y tipo como condiciones), prefijo `tmp/` con lifecycle de 1 día, validación posterior en un job, borrado del rechazado. Con `APP_FILES=disk`: un archivo por `PUT`, ≤ `LIMITES.maxBytesArchivo`.
- Nombre interno = UUID (el original va en los datos). Validar tipo **por contenido**, tamaño y cantidad.
- Datos tributarios/personales: borrar apenas dejan de ser necesarios.
- Documentos oficiales: **rellenar las plantillas oficiales de negocio** (`templates/`), nunca generar xlsx/docx «desde cero» ni copiar lo que hacía el legado si el formato oficial difiere.

---

## 5. Buenas prácticas de código

### 5.1 Código limpio

- TypeScript `strict`. **Sin `any`**, `# type: ignore`, `eslint-disable`, `noqa` nuevos sin justificación en el mismo comentario.
- Funciones de una responsabilidad; clases ≤ ~400 líneas; sin código muerto ni imports sin uso.
- Reutiliza los helpers del repo (`safe`, `apiError`, `log`, `getConfig`, `requireUser`…) en vez de crear variantes.
- Comentarios explican el **por qué**, no el qué. **Sin `TODO` sin issue.**
- Nunca silenciar excepciones (`catch {}`, `except: pass`).
- Términos de negocio idénticos en DTO, entidad y tabla (`apolice`, `tomador`, `garantia`, `endoso`); no traducir a medias.
- Nombres: TS `camelCase`/`PascalCase`, archivos NestJS `kebab-case.<rol>.ts`; Python `snake_case`; sufijos por rol (`*_service.py`, `*_router.py`, `*UseCase`, `*Repository`…).
- Patrones solo donde aportan: puertos/adapters para dependencias externas (`CorreoSaliente`, `PdfRenderer`, `IdentityProvider`), Strategy para modos.

### 5.2 Reglas de negocio

- **«El LLM extrae y narra; el código decide.»** Cada regla es una función pura con su test. Cambió la regla → cambió el código/`regras.yaml` + el test.
- Campo ausente → «a informar» / «No informado»; **nunca un valor adivinado**.
- No recalcular en código lo que la planilla oficial ya calcula (spread).
- Fuente caída ≠ resultado vacío: marcar `parcial`/`error`.
- Snapshot congelado al enviar (cambios posteriores no alteran lo enviado).
- IDs con generador seguro (nunca `Math.random`); folios desde secuencia de PG.

### 5.3 Idioma

- Documentos de Horizon, AGENTS y reglas: **español**. Repos de la familia spread/subscrição: **portugués**. Respeta el idioma del repo/archivo; **no mezcles** en un mismo repo.
- UI: texto en el idioma del usuario; si la app es bilingüe, pt-BR **y** es completos (sin claves faltantes; ngx-translate en Angular). Números, fechas y moneda por locale. Zona horaria explícita (`America/Santiago`, `America/Sao_Paulo`, `America/Lima` según país).

### 5.4 Seguridad y datos sensibles (LGPD / SUSEP)

- Fail-closed por defecto. Validación en el borde (zod/Pydantic). Queries parametrizadas siempre.
- Secretos nunca en código/repo/imagen/logs. Si encuentras uno commiteado: avisa, no lo reutilices, márcalo para rotación.
- Credenciales Google: **Workload Identity Federation** (tokens de corta duración; ADC en local). **Nunca llaves JSON** de service account.
- PII: minimizar, no loguear, retención definida, borrar al terminar.
- **La IA asiste, no decide** lo que mueve dinero o niega un derecho (LGPD art. 20, Lei 15.040): decisión por regla determinística o humano; ante error del modelo, degradar, nunca asumir «aprobado».
- Librerías/fuentes por CDN: aceptadas pero reportadas como deuda.
- Accesibilidad WCAG 2.1 AA. UX: nunca JSON ni jerga en pantalla; estados vacío / cargando / error / sin permiso.

### 5.5 Pruebas

- Pruebas de **comportamiento** (no `should create` / `toBeTruthy` vacíos). Cubrir para cada recurso: negocio, rol, dueño (`404` ajeno), validación, y el flujo asíncrono (`202 + id`, `400` síncrono, `1→2→3`, error → `4` con `errorDetail`, guardia «sin noticias»).
- Base real en integración (testcontainers / service PostgreSQL en CI), no H2/SQLite como sustituto.
- Prueba de humo del contenedor (§10) y doble arranque contra PG (migraciones no se repiten).
- Golden tests para documentos y cálculos; ratchet («trinquete») si hay fallos conocidos heredados.
- Todo cambio de lógica portado se porta **con su prueba** (o se agrega).

---

## 6. Despliegue en Horizon, paso a paso

Orden global: **gateway IA (si aplica) → app → hub (si aplica)**.

### 6.1 Antes del formulario

- Repo privado en `avla-tech`. Rama **sin `/`** (`develop`, `standalone`, `feat-async`; nunca `feat/x`).
- Prueba local (obligatoria):
  1. `docker build .` desde la raíz, **sin argumentos**.
  2. `docker run -p 8080:8080 <img>` **sin variables** → `/healthz` 200, `/readyz` 503 con `motivo`.
  3. Levantar PostgreSQL **después** (`docker compose up -d --build app` y luego `docker compose up -d postgres`; la app **sin `depends_on`**) → conecta, migra y pasa a `lista` **sin reiniciarse**.
  4. Reiniciar → las migraciones no se repiten.
  5. Medir el pico de memoria con el caso más pesado → elegir capacidad.
- Tener a mano: correos de las personas con acceso (≤ 5); si usa IA, modelos ya dados de alta en el gateway y token (§7.1).

### 6.2 Formulario «Deploy App»

| Campo | Valor |
|---|---|
| Repositorio | `github.com/avla-tech/<repo>` (sin `@rama`) |
| Rama | En su propio campo, sin `/` |
| Tipo de acceso | **Solo login con Google** |
| Capacidad | A / B / C según el pico medido (§2.3) |
| Almacenamiento S3 | **Objects** o **No necesita bucket** (nunca «Frontend») |
| Variables | Solo lo que la imagen no trae (§6.6). Primer deploy **sin credenciales de base** |

### 6.3 Inmediatamente después

1. **Conceder el primer acceso** en `lab-qa-idp.avla.cloud/apps/acceso-app` (activa el portero).
2. Verificar `/healthz` = 200 y `/readyz` = `503 esperando_base_datos` (el pod no se reinicia).
3. Cuando Horizon entregue la ficha de la base: cargar `PGHOST;PGPORT;PGDATABASE;PGUSER;PGPASSWORD;PGSSLMODE=require;PG_POOL_MAX=3` (reinicia el pod) → `/readyz` = 200.
4. Si marcaste Objects: verificar que la app lee `S3_BUCKET_OBJECTS`… y el **CORS** con preflight.
5. Diagnóstico de identidad → llega `X-Forwarded-Email` → **apagar** `DIAGNOSTICO_IDENTIDAD`.
6. `/api/me` muestra el correo y el rol correcto. Crear/confirmar el **primer admin**.
7. Cargar datos semilla (§6.5) y verificar conteos.
8. Caso de punta a punta con datos ficticios marcados como prueba.
9. Si usa hub: el evento aparece en `usage_events`.
10. **Redesplegar con un cambio trivial**: datos intactos, migraciones y seeds sin duplicar.
11. Un correo no autorizado no entra. El Monitor muestra 1 pod.
12. Registrar en el historial de `DEPLOY-HORIZON.md` (fecha, ambiente, quién, resultado, fricciones).

### 6.4 Integraciones opcionales

- Gateway IA → §7.1. Hub → §7.2. Google (Sheets/Gmail) → §7.3.

### 6.5 Datos iniciales

- La app arranca **sin datos** y lo dice («carga de datos pendiente»); `/readyz` no bloquea por eso.
- Carga = un `.sql` **transaccional y repetible** generado en local por un script del repo, ejecutado con `psql` (idealmente vía Docker) contra la base de Horizon **por un humano**. Verificar con `SELECT count(*)`.
- No se corren scripts ad hoc en la base de Horizon: correcciones de datos van como migración o endpoint de admin.

### 6.6 Manifiesto de variables (`deploy/horizon.env.example`)

Toda app tiene un manifiesto con cada variable anotada por **origen** y **obligatoriedad**, y dos líneas listas para pegar. Plantilla:

```dotenv
# =========================================================
# Manifiesto de variables para Horizon — <app>
# Formato del portal: CLAVE=valor;CLAVE=valor  (≤256 chars, sin espacios, sin ';', ninguna vacía)
# La imagen arranca SIN ninguna de estas variables (regla de oro).
# =========================================================

# --- Viene en la imagen (NO definir en Horizon) ---
# PORT=8080  WEB_DIST_DIR=public  MIGRATIONS_DIR=migrations  <APP>_SPA_DIR=/app/spa-solo

# --- Cargar a mano: base (@origen=ficha Acceso a DB, @obligatoria=si usa postgres) ---
PGHOST=<host-rds>
PGPORT=5432
PGDATABASE=lab_qa_<app>
PGUSER=usr_lab_qa_<app>
PGPASSWORD=<secreto>
PGSSLMODE=require
PG_POOL_MAX=3

# --- Inyecta Horizon si S3=Objects (@origen=horizon) ---
# S3_BUCKET_OBJECTS  AWS_REGION  AWS_ACCESS_KEY_ID  AWS_SECRET_ACCESS_KEY

# --- Identidad (@obligatoria=no) ---
ADMIN_EMAILS=<correo1>,<correo2>
# DIAGNOSTICO_IDENTIDAD=1   # solo primer deploy, luego quitar/0

# --- Gateway IA (@obligatoria=si usa IA) ---
GATEWAY_URL=https://gateway-ia.lab-qa.avla.cloud
GATEWAY_TOKEN=<secreto: token Horizon de un responsable>
GATEWAY_TIMEOUT=120
GATEWAY_MODO=async

# --- Hub (@obligatoria=no) ---
HUB_URL=https://<hub>.lab-qa.avla.cloud
HUB_APP_SLUG=<slug-en-apps.yaml>
HUB_REPASSAR_COOKIES=_oauth2_proxy

# --- Opcionales ---
LOG_LEVEL=info
<APP>_MAX_JOBS=1

# Línea primer deploy:
# ADMIN_EMAILS=...;LOG_LEVEL=info
# Línea al recibir la base (se agrega):
# PGHOST=...;PGPORT=5432;PGDATABASE=...;PGUSER=...;PGPASSWORD=...;PGSSLMODE=require;PG_POOL_MAX=3
```

### 6.7 Documento de despliegue por repo (`docs/DESPLIEGUE-HORIZON.md`)

Secciones: 1) **Ficha** (repo, rama, acceso, capacidad, S3, BD) · 2) **Variables** (primer deploy y al recibir la base; referencia al manifiesto, sin duplicar tablas) · 3) **Dependencias** (gateway, hub, seed) · 4) **Verificación** paso a paso · 5) **Qué cambió respecto del origen** (Antes | Ahora: integraciones, datos, auth, config, descartes — es el «diccionario» para agentes) · 6) **Verificado en local / NO probado aún** · 7) **Diagnóstico** (Señal | Significado | Acción) · 8) **Particularidades** · 9) **Historial de despliegues** (Fecha | Ambiente | Quién | Resultado | Fricciones).

No describas el funcionamiento general de Horizon en cada repo: enlaza a esta guía.

---

## 7. Integraciones soportadas

### 7.1 Gateway IA (`gateway-ia-avla`)

- URL `https://gateway-ia.lab-qa.avla.cloud` (`lab-qa-gateway-ia`). Proxy hacia Anthropic. Reemplaza a `api-chat-models`. **Prohibido** el AI gateway de Lovable y llamar directo a proveedores desde las apps.
- **Alta de modelos** (antes de desplegar la app): agregar en `models.yaml` del gateway `slug`, `allowed_apps` (vacío = solo admin, fail-closed), `provider: anthropic`, `model_id`, `cd_country`, `params` (`max_tokens`, `prompt_caching`; **sin `temperature`** en modelos que la rechazan), `system_prompt`. Redesplegar el gateway y verificar `GET /v1/models/<slug>` → 200. El seed **crea pero no actualiza** modelos existentes (cambios de `max_tokens`: verificar a mano). Un `POST /v1/models` desaparece al reiniciar.
- **Token:** el dueño de `gateway-ia` da acceso al correo `@avla.com` del **responsable** de la app consumidora en `/apps/acceso-app`; esa persona copia su token → `GATEWAY_TOKEN`. Es **personal**: si se revoca/vence, la app se cae. No usar el del dueño.
- Variables del consumidor: `GATEWAY_URL` (sin `/` final ni `/v1`), `GATEWAY_TOKEN` (secreto), `GATEWAY_TIMEOUT=120`, **`GATEWAY_MODO=async`** (por el corte de ~30 s), opcional `GATEWAY_POLL_INTERVAL=2`, `GATEWAY_POLL_MAX_INTERVAL=5`, `GATEWAY_DEADLINE=300`.
- Endpoints: `GET /v1/models` (prueba el token, gratis) · `POST /v1/executions` (síncrono) · **`POST /v1/executions:async`** → `202 {executionId}` · `GET /v1/executions/{id}` (polling).
- Body: `{ modelName, message, variables{}, documents[{mime_type, data_base64, name}], externalRef }` (no usar `fileUrls`). Documentos: PDF/PNG/JPEG/GIF/WEBP, ≤ 20, ≤ 32 MiB (ojo: ingress 25m y base64 +33 %).
- Headers: `Authorization: Bearer <token>`, `Content-Type: application/json`, **`Idempotency-Key`** (recomendado), `X-Correlation-Id`, `User-Agent`.
- **Idempotencia:** clave **determinística derivada del hecho de negocio** (`spread-<CNPJ>-2025-extracao`), nunca UUID/timestamp; ≤ 200 chars; alcance por app. `409 em_progresso` (en curso → seguir polling por `executionId`), `409 execution_ja_existe` (ya hecho), ERROR libera la clave, TTL 24 h.
- **Salida:** el `200` síncrono entrega la salida cruda **una sola vez** → **persístela al recibirla**. El `GET` devuelve salida **enmascarada** (`[PII]`, rompe números de ≥ 10 dígitos) salvo que el gateway tenga `RESULT_ENCRYPTION_KEY` (requiere aprobación de DPO/Seguridad) → `outputRaw: true`. El consumidor **falla cerrado** si recibe SUCCESS sin `outputRaw: true` cuando lo necesita.
- `usage.costUsd` es **string** → `Decimal`, nunca float; `priced:false` = desconocido, no cero.
- Errores: `302` HTML de Google / `403` HTML de CloudFront = **no llegó al gateway** (token o WAF); `504` = CloudFront; `404 modelo_no_encontrado`; `429 rate_limit_excedido | cuota_agotada`; `503 provider_no_disponible`.
- Limitación del lab: todas las apps entran como `dev@avla.com` y comparten cuota → identifícate con `externalRef`.
- LGPD: minimiza (manda solo lo necesario vía `variables`), prompt que prohíba veredictos, maneja `429/503` degradando. Antes de datos reales: confirmar base legal y DPA con el DPO.

### 7.2 Minihub (`avla-miniapps-hub`)

- Registrar la app en `backend/apps.yaml` del hub (`id`/`slug` `^[a-z0-9][a-z0-9-]*$`, `name`, `description`, `category`, `launch_url: ${HUB_APP_<X>_URL}`, `allowed_groups: ["${HUB_GROUP_<X>}"]`, `enabled`, `metrics.stages` opcional que debe coincidir con las etapas que emite la app). Variable faltante = el hub **no arranca** (para todas las apps). **El agente no edita `apps.yaml` de otro repo: lo reporta.**
- Antes de redesplegar el hub: cargar `HUB_APP_<X>_URL` y `HUB_GROUP_<X>`, agregar el grupo a **`AVLA_DEV_GROUPS`** (no `AVLA_ALLOWED_GROUPS`), apps no desplegadas con `https://sin-desplegar.invalid` y su grupo fuera de la lista, **respaldar la base del hub**.
- En la app: `HUB_URL`, `HUB_APP_SLUG` (idéntico al slug) y `HUB_REPASSAR_COOKIES=_oauth2_proxy` (reenvía la cookie del portero). Telemetría `POST {HUB_URL}/api/events` con `{"app","type","durationMs"}` (campos extra → 422). Un slug no registrado → 422/404 **silencioso**: loguea y cuenta los rechazos.

### 7.3 Google (Sheets, Gmail) y otras externas

- Requieren **aprobación de Arquitectura/DevOps**. Vía técnica aceptada: **Workload Identity Federation** (`GCP_WIF_AUDIENCE`, `GCP_SERVICE_ACCOUNT`, `GCP_WIF_ORIGEN=aws|archivo`), ADC en local, y **modo degradado** (p. ej. «cargar Excel» manual, «correo para copiar»).
- Prohibido: conectores de Lovable (`connector-gateway.lovable.dev`, `LOVABLE_API_KEY`), llaves JSON, hojas públicas.
- APIs internas de AVLA: inalcanzables desde el lab salvo prueba en contrario desde el pod; la app es «proveedor externo» y solo consume URL + Swagger entregados por AVLA, con la URL en config.

---

## 8. Metodología de migración a Horizon

### 8.1 Principios

1. **Inventario primero**: roles, reglas de negocio (con cita `archivo:línea` del legado), integraciones, datos, bugs conocidos que la migración corrige.
2. **Se reimplementa, no se transplanta.** Se reutilizan **contratos** (ExecutionProcess, portero) y **casos de prueba**, no código de otro stack.
3. Legado como referencia viva en `legacy/` hasta el cutover; luego se elimina.
4. **Paridad** funcional (campos), visual (CSS/markup portados tal cual cuando aplica) y documental (golden tests contra **formatos oficiales de negocio**).
5. **Despliegue primero:** el esqueleto llega a Horizon en la fase 0 para despejar incógnitas de plataforma antes de construir negocio.
6. **Recortar a MVP desplegable** cuando el plan objetivo excede lo que Horizon permite; registrar regresiones aceptadas explícitamente.
7. **No inventar funcionalidades** que el origen no tiene (ni estados, ni campos).
8. **Cada decisión con su porqué** (tabla fechada, ADRs, tabla «Qué cambió respecto del origen»).
9. **Dependencias no habilitadas no bloquean:** puerto/adapter con implementación degradada y motivo en `/readyz` y en la UI.
10. Auditar los errores típicos del legado: contraseñas mock, guards solo en cliente, `ok:true` con errores silenciados, `Math.random` como ID, XSS por `innerHTML`, desfase UTC de un día, solo el primer elemento de una lista, variables con espacios/`=` inicial, dev apuntando a prod.

### 8.2 Tabla de reemplazos origen → Horizon

| Origen | Destino en Horizon |
|---|---|
| Supabase client en el navegador | Endpoint `/api/*` del backend + consumo desde la web. **El navegador nunca habla directo con la base.** |
| `supabase/migrations/` | Migración equivalente en el mecanismo del destino (Drizzle / Alembic), idempotente, señalada en el PR, **nunca aplicada por el agente** |
| Supabase Edge Functions | Rutas `/api/*`; si > ~10–30 s, `202` + polling |
| Lovable Cloud (auth, storage, jobs, emails, realtime, secrets) | **Prohibido** → stack AVLA |
| Login propio / Google OAuth / Supabase Auth / JWT | Portero + `X-Forwarded-Email`; `requireUser`/`requireAdmin` |
| Roles hardcodeados o en variables | Tabla de usuarios/roles + `ADMIN_EMAILS` como bootstrap |
| Supabase Storage / Drive | S3 «Objects» con prefirmado |
| Multipart / base64 al backend | Directo navegador → S3 (WAF 8 KB, memoria) |
| Google Docs / DocumentApp → PDF | `@react-pdf/renderer` en proceso hijo (o python-docx/openpyxl sobre plantillas oficiales) |
| Gmail / MailApp | Puerto `CorreoSaliente` degradado («correo para copiar») hasta aprobar WIF/SES |
| Conectores Lovable (Sheets…) | API directa con WIF + respaldo «cargar Excel» |
| AI gateway de Lovable / llamadas directas a LLM | Gateway IA de AVLA con Bearer + `GATEWAY_MODO=async` |
| Cola / Redis / cron | pg-boss sobre PostgreSQL (cron de negocio requiere aprobación; polling y purga TTL permitidos) |
| Realtime / websockets | Refresco periódico (p. ej. cada 10 s) |
| `process.env` / `import.meta.env` dispersos, `VITE_*` secretos | Módulo único de config en el backend |
| IDs, URLs, TTL fijos | Variable con el valor actual como defecto |
| Endpoint abierto | `requireUser` / `requireAdmin` |
| SQLite / estado en `/tmp` | PostgreSQL |
| `localStorage` como base | Borrador en servidor con autosave + `version` (409) |
| Apps Script `doPost` anónimo | API con identidad del portero |
| Webhook n8n abierto / flujo n8n | Endpoint autenticado; tabla «nodo por nodo → dónde vive ahora» |
| Dockerfile nginx de ejemplo + `index.html` | Dockerfile de la plantilla (§16.1); sin `index.html` en la raíz |
| MinIO en local | `adobe/s3mock` |
| Errores de terceros al navegador | Mensaje para la persona; detalle al log |

### 8.3 Fases estándar (adaptar)

| Fase | Contenido | Criterio de salida |
|---|---|---|
| **M0 · Esqueleto desplegado** | Repo, Dockerfile, config, arranque sin base, identidad (portero/dev), `/me`, diagnóstico, ExecutionProcess, CI con puertas | Desplegado en Horizon con la secuencia §6.3 en verde |
| **M1 · Catálogos + entidades base** | CRUD principal, roles, migraciones | Paridad de campos; tests 401/403/404 |
| **M2 · Flujo principal + carga asíncrona** | Wizard/proceso núcleo, subida a S3 | Caso real de punta a punta |
| **M3 · Generación documental / integraciones** | Plantillas oficiales, IA vía gateway | Golden tests; ninguna request > 10 s; memoria medida |
| **M4 · Bandejas / dashboards** | Vistas operativas | UAT con 2–3 usuarios internos |
| **M5 · Endurecimiento + piloto** | Checklist de seguridad, ADRs, piloto en paralelo 1–2 semanas | Aprobación de negocio |
| (Opcional) **Migración histórica + cutover** | CLI `--dry-run|--commit` idempotente, capa anti-corrupción, reporte de conciliación; congelar origen → pasada final → validar → apagar | Conteos conciliados |

### 8.4 Plantilla de plan de migración (el agente la produce)

```markdown
# Plan de migración: <app> (<origen> → Horizon)
## 0. Contexto
- Por qué migrar (problemas del origen con cita archivo:línea) · Objetivo · Decisiones ya tomadas con el usuario
## 1. Inventario del origen
- Roles actuales → nuevos · Reglas de negocio a portar (cita) · Integraciones · Datos (volumen, dónde viven) · Bugs conocidos que se corrigen
## 2. Hechos de Horizon y su consecuencia
| Hecho de Horizon | Consecuencia en este plan |   ← usar §2 y §4 de la guía
## 3. Alcance
- Dentro (A1..An) · Fuera · Sustitutos temporales (modo degradado) · Regresiones aceptadas (explícitas)
## 4. Arquitectura destino
- Perfil de stack (§3) y por qué · Módulos · Capas · Procesos asíncronos · Arranque sin base
## 5. Tabla de reemplazos origen → destino   ← partir de §8.2
## 6. Identidad y autorización (portero, roles, dueño)
## 7. Modelo de datos y migraciones (tablas, prefijos, numeración, reversibilidad, seeds)
## 8. API (recursos, códigos, contratos ExecutionProcess)
## 9. Frontend (paridad visual, estados, i18n)
## 10. Despliegue (formulario, manifiesto de variables, capacidad con pico medido, secuencia §6.3)
## 11. Fases con entregables y criterio de salida (§8.3)
## 12. Verificación (pruebas, golden, humo, doble arranque, E2E)
## 13. Riesgos y puntos abiertos (Tema | Acción | Cuándo | Dueño)
## 14. Requiere decisión humana / aprobación (§12.2)
```

Documentos que deja una migración en el repo: `AGENTS.md` (resumen + «Decisiones de esta app» con su porqué), `CLAUDE.md` (`@AGENTS.md`), `docs/DESPLIEGUE-HORIZON.md` (§6.7, con la tabla «Qué cambió respecto del origen»), `deploy/horizon.env.example`, `docs/adr/*`, README, y — si seguirá recibiendo cambios del origen — `.upstream-sync.yml` (§9). Para trabajo largo, un documento de traspaso tipo `CONTEXTO_MIGRACION.md` (historia de decisiones fechada, estado por fase, trampas técnicas, pendientes con DevOps/negocio).

---

## 9. Flujo upstream-sync (Lovable → repo migrado) — PoC en `lab-qa-ia-lab-08`

Para repos ya migrados que siguen recibiendo funcionalidades del repo Lovable de origen.

- **Precondiciones:** `.upstream-sync.yml` (`source_repo` en una org de AVLA, `source_branch`, `baseline_sha`, `ignore_paths: [.lovable/, src/routeTree.gen.ts]`, `stack`, `config_module`, `env_example`); `docs/DESPLIEGUE-HORIZON.md` con la tabla «Qué cambió respecto del origen»; `AGENTS.md` que lo declara; app desplegada; checks `validar` y `sin-bloqueantes` requeridos; `CODEOWNERS`.
- **Workflow** (`workflow_dispatch`): prepara diff/commits desde `baseline_sha` → rama `sync/upstream-YYYYMMDD-<sha7>` → **agente que porta** (`claude -p`, sin commit/push) → **puertas** (`validar.sh`: formato, lint, tipos, tests, `chequeo-horizon.sh`, gitleaks, humo Docker) → **agente revisor** con contexto limpio (solo lectura; última línea `BLOQUEANTES: <n>`) → hasta 2 rondas de corrección → PR a `develop` (borrador + `necesita-humano` si no queda limpio). `baseline_sha` avanza dentro del PR.
- **Reglas para el agente que porta:** *porta, no refactorices*; no toques `.github/` ni `.upstream-sync.yml`; Dockerfile solo si el origen lo exige; aplica la tabla §8.2; migraciones en el mecanismo del destino y **nunca las ejecutes**; porta las pruebas; actualiza doc de despliegue, manifiesto y README. Informe con: Resumen · Portado (commit | cambio | archivos) · Omitido a propósito · Requiere decisión humana · Migraciones (archivo, qué hace, reversible) · Variables nuevas (nombre, defecto, ¿se carga en Horizon?) · Deuda detectada.
- **Revisor:** `[bloqueante]` solo para violaciones de Horizon, seguridad, fidelidad o pruebas debilitadas; deuda previa → `[sugerencia]`.

---

## 10. CI y puertas de calidad

### 10.1 Puertas mínimas (bloqueantes; ninguna con `continue-on-error`)

1. Formato sin diferencias (prettier / `ruff format --check`).
2. Lint (`eslint --max-warnings=0` / `ruff check` / `ng lint`).
3. Tipos (`tsc --noEmit` / `npm run typecheck` / mypy).
4. Tests (con PostgreSQL real si hay base), sin debilitar.
5. `docker build .` del Dockerfile raíz, sin argumentos.
6. **Humo del contenedor:** arranca **sin variables**; `/healthz` 200 en ≤ 60 s; `/readyz` 200 o 503 con motivo; con PG: arranque doble, migraciones no repetidas.
7. **Chequeo Horizon** (`chequeo-horizon.sh`): Dockerfile en la raíz, `EXPOSE 8080`, `PORT=8080`, `USER 10001`, HEALTHCHECK sin curl, sin `ARG` sin defecto, manifiesto de variables presente, sin lecturas de entorno fuera del módulo de config, sin `connector-gateway.lovable.dev` / `LOVABLE_API_KEY`; TanStack: sin `index.html` en raíz y con `NITRO_PRESET=node-server`.
8. **Secretos:** gitleaks sobre el historial/rango.
9. **Imagen:** hadolint (`failure-threshold: warning`) + Trivy `CRITICAL,HIGH` (`scanners: vuln,secret`, excepciones en `.trivyignore.yaml` con justificación y vencimiento). Fijar actions de terceros por SHA cuando haya riesgo de cadena de suministro.
10. Dependencias: `npm audit --omit=dev --audit-level=high` / `pip-audit`.

Comandos de validación por perfil:

```bash
# node-tanstack-lovable
npm install --no-audit --no-fund && npm run lint && npx tsc --noEmit && npm test && docker build -t app .
# node-monorepo
npm ci && npm run lint && npm run typecheck && npm test && docker build -t app .   # (lab-08: npm run verify)
# python-angular
ruff format --check . && ruff check . && mypy . && pytest -q && npx ng build && docker build -t app .
```

### 10.2 Git y PR

- Trabajo en ramas → PR a `develop`. No reescribir historia publicada (force push, rebase/amend de commits subidos).
- PR pequeño y revisable; deuda fuera de alcance → sección «Deuda detectada» + issue.
- Revisión en 3 capas: puertas automáticas → revisor con contexto limpio → aprobación humana `CODEOWNERS` con las puertas como *required checks*.
- `.gitattributes` para CRLF (equipo en Windows; usar `curl.exe` en PowerShell).

---

## 11. Fricciones conocidas (síntoma → causa → solución)

| Síntoma | Causa | Solución |
|---|---|---|
| `502` en ~1 ms | App escucha en 8000 | `PORT=8080`, `EXPOSE 8080` |
| Portal rechaza repo/rama | `repo@rama` o rama con `/` | Campos separados; rama sin `/` |
| Horizon no construye front+back | Solo lee el Dockerfile raíz, sin build args | Dockerfile único; backend sirve la SPA; dos bundles + variable |
| «Build de la imagen certificada» falla (`edgesOut`) | npm 10 con árbol de Lovable | `npm install -g npm@11.21.0` |
| Servidor generado para Cloudflare / rutas interceptadas | Falta `NITRO_PRESET=node-server` / `index.html` en raíz | Agregar preset; borrar `index.html` |
| «Abra pelo portal Avla» / `/sin-acceso` | Bundle production exige token del hub | Bundle local/solo por defecto |
| `banco_postgres_nao_configurado` / `esperando_base_datos` | Horizon no inyecta `PG*`; base creada después | Cargar `PG*` a mano |
| Reintentos infinitos, código `28000` | RDS exige SSL | `PGSSLMODE=require` |
| `migração falhou (OperationalError)` sin detalle | `DATABASE_URL` mal armada / clave mal copiada | `PG*` separadas + motivo clasificado |
| Variable truncada | `;` en el valor o JSON | Sin `;` (o `%3B`), sin JSON |
| Llave JWT rechazada | > 256 caracteres | Sin JWT propio en el lab |
| `OOMKilled` | Límite de capacidad | Medir pico, subir capacidad, `*_MAX_JOBS=1`, proceso hijo acotado |
| `504` | CloudFront ~30 s | `202` + polling; `GATEWAY_MODO=async` |
| `403` con HTML de CloudFront | WAF (body > 8 KB, GET con body, sin User-Agent, comillas) | Prefirmado a S3; sin body en GET; User-Agent; pedir excepción a DevOps |
| `403` «Secured with OAuth2 Proxy» | Cuenta sin acceso o acceso vencido | Dueño otorga acceso en `/apps/acceso-app` |
| `302` a Google desde otra app | Llamada backend→backend atraviesa el portero | Bearer (gateway) o cookie `_oauth2_proxy` (hub) |
| `404 modelo_no_encontrado` | Modelo sin alta en gateway | `models.yaml` + `GET /v1/models` |
| Modelo rechaza la llamada | `temperature` / seed no actualiza | Quitar `temperature`; verificar a mano |
| `usage_events` vacío | App no registrada en `apps.yaml` (422 silencioso) | Registrar; loguear rechazos |
| Hub no arranca | Faltan `HUB_APP_<X>_URL` / `HUB_GROUP_<X>` | Cargar antes del redeploy |
| Catálogo del hub vacío | Grupos en `AVLA_ALLOWED_GROUPS` | Usar `AVLA_DEV_GROUPS` |
| Datos perdidos tras cambiar variable | SQLite / `/tmp` | PostgreSQL |
| Subida desde navegador falla tras redeploy | CORS del bucket perdido / URL de la app cambió | Re-verificar preflight; actualizar CORS |
| «Cerrar sesión» cierra todo | Cookie en `.lab-qa.avla.cloud` | Conocido; CSRF same-origin |
| No llega a API interna de AVLA | Clúster aislado | Probar desde el pod; API-Gateway público; DevOps |
| pg-boss deadlock al arrancar | `createQueue` fuera del lock | Crear colas dentro del advisory lock |
| Errores de Drizzle sin `code` | `DrizzleQueryError` envuelve `pg` | Leer `error.cause.code` |
| CSP bloquea estilos | `style="..."` inline | Clases CSS |
| `pino-pretty` tumba la imagen | Dependencia de dev en runtime | Cargarlo solo si está instalado/dev |
| Lovable reescribe lógica crítica | Prompts de cambio en Lovable | Regla «No modifiques `src/lib/<núcleo>*`» + pruebas de paridad |

**Límites sin solución hoy:** 1 réplica · 5 personas por portero · cupo 10 del lab · token del gateway personal · sin ruta interna entre apps · sin jobs puntuales · sin ConfigMap/archivo montable.

**Pedidos pendientes a plataforma:** inyectar `PG*` al marcar base; editor por variable; timeout de CloudFront para el gateway; ruta interna entre apps; credencial de servicio por app; montar archivos de configuración; jobs puntuales.

---

## 12. Prohibido y requiere aprobación

### 12.1 Prohibido (no implementar aunque lo pidan)

- Lovable Cloud / Supabase en cualquier forma (BD, auth, storage, functions, jobs, emails, realtime, secrets); AI gateway y conectores de Lovable.
- Login propio (contraseñas, OAuth, JWT, cookies de sesión).
- Credenciales en código/repo/imagen; llaves JSON de service account; PEM o JSON en variables.
- Endpoints sin usuario (salvo `/healthz`, `/readyz`); usuario genérico fuera de dev.
- Bypass de auth fuera de `AVLA_ENV` ∈ {dev, local, test}; bypass de fábrica en la imagen de un portal/hub.
- Llamar a otra app de Horizon por su URL pública desde el backend (salvo gateway/hub como en §7).
- Conectarse a redes o bases internas de AVLA desde el lab sin acuerdo con DevOps.
- Dockerfile fuera de la raíz, `--build-arg` obligatorios, puerto ≠ 8080, `curl` en healthcheck, `/healthz` dependiente de la base.
- Requests síncronas largas (> ~10 s; nunca > 30 s); GET con body; cuerpos > 8 KB hacia el backend; binarios en base64 dentro de JSON.
- Datos importantes en SQLite / `/tmp` / memoria; `DATABASE_URL` armada a mano; leer `BUCKET_NAME`.
- `drizzle-kit push` contra un ambiente; editar migraciones aplicadas; que el agente aplique migraciones o scripts contra una base real.
- `index.html` en la raíz (TanStack); editar `routeTree.gen.ts`.
- Bucket «Frontend»; tipo de acceso «Google + token» para apps con usuarios; repartir la URL antes de conceder el primer acceso.
- `temperature` en modelos que la rechazan; `float` para dinero o `costUsd`.
- Borrar/saltar/debilitar pruebas; `any` sin justificación; stack traces al usuario.

### 12.2 Requiere aprobación de Arquitectura/DevOps antes de construir

Tareas programadas de negocio (cron) · correos o notificaciones · tiempo real (websockets) · **IA dentro de la app** · pagos · integraciones no provistas por AVLA (Google, Buk, APIs externas) · excepciones a los límites · módulos permanentes `postgres`/`s3` en el perfil TanStack · `RESULT_ENCRYPTION_KEY` con datos reales (DPO/Seguridad) · cualquier desvío de stack (ADR).

Respuesta estándar al usuario:

> «Esta función no está cubierta por los lineamientos de Avla y necesita aprobación de Arquitectura/DevOps antes de construirla. Consulte con ellos y vuelva con la indicación.»

Permitido sin aprobación: polling, purga por TTL, modo degradado, refresco periódico.

---

## 13. Checklists

### 13.1 Antes de entregar código (agente)

- [ ] Cumple §4 (imagen, salud, config, identidad, datos, async, HTTP, archivos).
- [ ] Ninguna lectura de entorno fuera del módulo de config; variables nuevas en manifiesto + README con defecto.
- [ ] Endpoints nuevos con `requireUser`/`requireAdmin`, validación zod/Pydantic, `404` para ajenos.
- [ ] Pruebas de negocio, rol, dueño, validación y flujo async agregadas y en verde (resultado real).
- [ ] Puertas §10.1 corridas localmente (o declarado cuáles no se pudieron correr y por qué).
- [ ] Migraciones nuevas: numeradas, idempotentes, aditivas, declaradas en el PR, **no aplicadas**.
- [ ] `AGENTS.md` «Decisiones de esta app» y `DESPLIEGUE-HORIZON.md` actualizados si cambió algo de despliegue.
- [ ] Nada de §12.1; lo de §12.2 escalado, no construido.

### 13.2 Pre-despliegue

- [ ] Repo privado en `avla-tech`; rama sin `/`.
- [ ] `docker build .` sin args; corre sin variables (`/healthz` 200, `/readyz` 503 con motivo).
- [ ] Base levantada después → migra y queda lista sin reinicio; segundo arranque sin re-migrar.
- [ ] Pico de memoria medido → capacidad elegida.
- [ ] Manifiesto `deploy/horizon.env.example` con origen/obligatoriedad; valores ≤ 256, sin espacios, sin `;`, sin vacías.
- [ ] Si usa IA: modelos en el gateway (`GET /v1/models/<slug>` 200) y token de un responsable.
- [ ] Si usa hub: entrada en `apps.yaml` + variables `HUB_*` listas + respaldo de la base del hub.
- [ ] Correos de acceso (≤ 5) a mano.

### 13.3 Post-despliegue

- [ ] Primer acceso concedido de inmediato.
- [ ] `PG*` + `PGSSLMODE=require` + `PG_POOL_MAX=3` cargadas; `/readyz` 200.
- [ ] `X-Forwarded-Email` llega; diagnóstico apagado; `/api/me` correcto; primer admin creado.
- [ ] CORS del bucket verificado con preflight (si aplica).
- [ ] Datos semilla cargados y conteos verificados.
- [ ] E2E con datos de prueba; evento en `usage_events` (si hub).
- [ ] Redeploy trivial: datos intactos, sin re-migrar.
- [ ] Correo no autorizado no entra; Monitor muestra 1 pod.
- [ ] Historial de despliegue actualizado.

---

## 14. Contradicciones entre fuentes y cómo quedan resueltas

| # | Tema | Versiones encontradas | **Decisión en esta guía** |
|---|---|---|---|
| 1 | Auth en el lab | LABORATORIO (24-09): JWT ES384 propio · PDF (29-10): sin JWT, portero | **Portero + `X-Forwarded-Email`**; sin JWT propio en el lab |
| 2 | Dónde vale el bypass | «solo dev» · «dev/local/test» | **`AVLA_ENV` ∈ {dev, local, test}**; en otro valor, el boot aborta |
| 3 | Capacidad A/B/C | «1/4/8 pods» · recursos | **Siempre 1 réplica**; A/B/C = recursos; 1/4/8 = cupo de 10 |
| 4 | Corte de 30 s / 8 KB | DEPLOY hub: «no aplican» · PDF y lab-01: sí aplican | **Aplican** (CloudFront ~30 s, WAF 8 KB) |
| 5 | ¿Horizon inyecta `PG*`? | READMEs lab-02/03: sí · PDF, lab-01, lab-08: no | **No**; se cargan a mano (soportar ambos casos en código) |
| 6 | Variables S3 | `BUCKET_NAME/BUCKET_REGION/AWS_ACCESS_KEY` (v1, lab-01) · `S3_BUCKET_OBJECTS/AWS_REGION/AWS_ACCESS_KEY_ID` (PDF, horizon-general) | **`S3_BUCKET_OBJECTS`, `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`** (verificado en lab-03) |
| 7 | `DATABASE_URL` | lab-01 la acepta · horizon-general la prohíbe | **`PG*` separadas en Horizon**; `DATABASE_URL` solo local/compat |
| 8 | Pool PG | lab-01 `max:5` · `PG_POOL_MAX=3` | **3** |
| 9 | Usuario del contenedor | `USER node` (lab-01) · 10001 propio (spreads) · `10001:0` + `g=u` (guía) | **`10001:0` con `chmod g=u`** |
| 10 | Puerto | 8000 (v1, AGENTS viejos) · 8080 | **8080** |
| 11 | Quién migra | pipeline (v1, core) · entrypoint (lab-02) · app con lock (lab-01/08, ADR-004) | **La app, en segundo plano, con advisory lock, al detectar la base** |
| 12 | Login | Google OAuth propio (PLAN_MIGRACION, v1) · portero (ADR-003, v2) | **Portero** |
| 13 | Variable de admins | `ADMIN_BOOTSTRAP_EMAILS` (lab-01) · `ADMIN_EMAILS` | **`ADMIN_EMAILS`** (aceptar el otro como alias en repos existentes) |
| 14 | Persistencia en disco | v2: disk con TTL permitido · horizon-general: nada importante en disco | **Disco solo para estado transitorio con TTL**; lo que debe sobrevivir, en PG/S3 |
| 15 | `depends_on` en compose | lab-02 lo usa · lab-01/08 lo evitan | **Sin `depends_on`** (simula Horizon) |
| 16 | Destino del HEALTHCHECK | `/readyz` (spreads) · `/healthz` | **`/healthz`** |
| 17 | Modo de bundle | `ARG SPA_CONFIGURATION` (spreads) · dos bundles + variable (carpeta, hub) | **Dos bundles + `<APP>_SPA_DIR` en runtime** |
| 18 | Base del gateway | Postgres (Fricciones) · SQLite en `/tmp` (PDF) | **SQLite en `/tmp` hoy** (efímero; migrar a RDS pendiente) |
| 19 | Bypass vs reglas de oro core («nunca un flag que apague auth») | — | En Horizon el bypass existe **solo** porque el portero es la puerta y solo en dev/local/test; **en core sigue prohibido** |
| 20 | Versionado de API | core `/api/v1` · Horizon `/api/*` | **`/api/*`** (respetar `/api/v1` si el repo ya lo usa) |
| 21 | Tests de BD | H2 (Confluence) · Testcontainers (Manual) | **Base real** (testcontainers / service PG) |
| 22 | Reversibilidad de migraciones | no exigida (lab-08) · probada (lab-04) | **Declarar siempre; probar `downgrade` en Python; preferir aditivas** |
| 23 | Ubicación doc de despliegue | `DEPLOY-HORIZON.md` raíz (lab-08) · `docs/DESPLIEGUE-HORIZON.md` (lab-06, reglas) | **`docs/DESPLIEGUE-HORIZON.md`** con la tabla «Qué cambió respecto del origen» |
| 24 | Stack base para apps nuevas | ADR-001 Node/TS (pendiente de Arquitectura) · FastAPI+Angular | Ambos válidos; **imitar el repo**; nuevos desde Lovable → TanStack; nuevos «a mano» → node-monorepo |

---

## 15. Supuestos y pendientes (no verificados)

- `SUPUESTO`: Ruta interna pod a pod en `lab-qa-poc` (DNS de servicio) — a confirmar con DevOps.
- `SUPUESTO`: Si el portal deja la URL abierta antes del primer acceso o pide login a cualquier `@avla.com` (fuentes ambiguas) — conceder acceso de inmediato en todo caso.
- `SUPUESTO`: Corte exacto de CloudFront (~30 s observado, no confirmado oficialmente).
- ADR-001 (stack Node/TS) pendiente de validación formal por Arquitectura.
- Gateway IA: migrar de SQLite a RDS; credencial de servicio por app; DPA con el proveedor no verificado.
- `consultas-externas`: slug `consultas-externas` no existe en `apps.yaml` actual.
- Perfil `python-angular`: herramientas de lint/tipos del equipo por confirmar.
- `lab-qa-ia-lab-03` y `Avla-subscricao-web` son ramas divergentes del mismo producto (Postgres vs features recientes): reconciliar antes de nuevos despliegues.
- Design system: no hay uno formal para Horizon. Referencias de marca disponibles: azul `#0031E2`, tipografía Poppins (plantilla oficial de subscrição). Declararlo como premisa a validar en cada UI nueva.

---

## 16. Plantillas de referencia

> Derivadas de `lab-qa-ia-lab-08` (Node) y `carpeta-tributaria` / `lab-qa-ia-lab-02` (Python). **Adáptalas al repo; si el repo ya tiene Dockerfile funcional, no lo reemplaces.**

### 16.1 Dockerfile — Node monorepo

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine AS build
RUN npm install -g npm@11.21.0
WORKDIR /src
COPY package.json package-lock.json ./
COPY packages/shared/package.json packages/shared/
COPY apps/api/package.json apps/api/
COPY apps/web/package.json apps/web/
RUN npm ci
COPY . .
RUN npm run build && npm prune --omit=dev

FROM node:22-alpine AS runtime
ENV NODE_ENV=production PORT=8080 AUTH_MODE=portero PGSSLMODE=require PG_POOL_MAX=3 \
    WEB_DIST_DIR=public MIGRATIONS_DIR=migrations
WORKDIR /app
RUN adduser -D -H -u 10001 -G root app \
 && rm -rf /usr/local/lib/node_modules/npm /usr/local/bin/npm /usr/local/bin/npx /usr/local/bin/corepack /opt/yarn*
COPY --from=build --chown=10001:0 /src/node_modules ./node_modules
COPY --from=build --chown=10001:0 /src/packages/shared/dist ./node_modules/@<scope>/shared/dist
COPY --from=build --chown=10001:0 /src/apps/api/dist ./dist
COPY --from=build --chown=10001:0 /src/apps/api/migrations ./migrations
COPY --from=build --chown=10001:0 /src/apps/web/dist ./public
RUN chmod -R g=u /app
USER 10001:0
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:'+process.env.PORT+'/healthz').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
CMD ["node", "dist/main.js"]
```

### 16.2 Dockerfile — TanStack Start (Lovable)

```dockerfile
FROM node:22-alpine AS build
RUN npm install -g npm@11.21.0
WORKDIR /src
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
ENV NITRO_PRESET=node-server
RUN npm run build

FROM node:22-alpine AS runtime
ENV NODE_ENV=production PORT=8080 APP_DATA=disk APP_FILES=disk STATE_DIR=/var/tmp/app STATE_TTL_MIN=30
WORKDIR /app
RUN adduser -D -H -u 10001 -G root app && mkdir -p /var/tmp/app && chown -R 10001:0 /var/tmp/app && chmod -R g=u /var/tmp/app
COPY --from=build --chown=10001:0 /src/.output ./.output
USER 10001:0
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:'+process.env.PORT+'/healthz').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
CMD ["node", ".output/server/index.mjs"]
```

### 16.3 Dockerfile — FastAPI + Angular

```dockerfile
FROM node:22-alpine AS spa
WORKDIR /web
COPY frontend/package.json frontend/package-lock.json ./
RUN npm ci
COPY frontend/ ./
RUN npx ng build --configuration local --output-path /out/spa-solo \
 && npx ng build --configuration production --output-path /out/spa-portal

FROM python:3.12-slim AS builder
ENV PIP_NO_CACHE_DIR=1 PIP_DISABLE_PIP_VERSION_CHECK=1
RUN python -m venv /opt/venv
COPY backend/requirements.txt .
RUN /opt/venv/bin/pip install -r requirements.txt

FROM python:3.12-slim AS runtime
# hadolint ignore=DL3008
RUN apt-get update && apt-get upgrade -y && apt-get install -y --no-install-recommends tini \
 && rm -rf /var/lib/apt/lists/* && useradd -u 10001 -g 0 -M app \
 && mkdir -p /var/tmp/<app> && chown -R 10001:0 /var/tmp/<app> && chmod -R g=u /var/tmp/<app>
ENV PATH=/opt/venv/bin:$PATH PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1 PORT=8080 \
    <APP>_SPA_DIR=/app/spa-solo <APP>_STATE_DIR=/var/tmp/<app>
WORKDIR /app
COPY --from=builder /opt/venv /opt/venv
COPY --chown=10001:0 backend/ ./
COPY --from=spa --chown=10001:0 /out/ /app/
USER 10001:0
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD python -c "import os,urllib.request,sys; sys.exit(0 if urllib.request.urlopen(f'http://127.0.0.1:{os.environ[\"PORT\"]}/healthz',timeout=4).status==200 else 1)"
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["sh", "-c", "exec uvicorn app.main:app --host 0.0.0.0 --port \"${PORT}\" --workers 1 --proxy-headers --forwarded-allow-ips '*'"]
```

### 16.4 docker-compose local

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: lab_qa_<app>
      POSTGRES_USER: usr_lab_qa_<app>
      POSTGRES_PASSWORD: ${PGPASSWORD:?defina PGPASSWORD en .env}
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U usr_lab_qa_<app>"], interval: 5s, retries: 10 }
  s3:
    image: adobe/s3mock
    environment: { initialBuckets: <app>-local }
    ports: ["9090:9090"]
  app:
    build: .
    ports: ["8080:8080"]
    # SIN depends_on: igual que en Horizon, la app arranca antes que la base
    environment:
      AVLA_ENV: local
      AUTH_MODE: dev            # o AVLA_AUTH_DISABLED=1 en Python
      PGHOST: postgres
      PGDATABASE: lab_qa_<app>
      PGUSER: usr_lab_qa_<app>
      PGPASSWORD: ${PGPASSWORD}
      PGSSLMODE: disable        # solo local
volumes: { pgdata: {} }
```

### 16.5 `AGENTS.md` mínimo de una app Horizon

```markdown
# AGENTS.md — <app>
Antes de cualquier cambio lee `GUIA-AGENTES-HORIZON.md` (guía maestra) y `docs/DESPLIEGUE-HORIZON.md`.
Si algo de este archivo contradice la guía, manda la guía.

## Resumen obligatorio
- Qué hace · Usuarios · Perfil de stack: <node-tanstack-lovable | node-monorepo | python-angular>
- Comandos: build / lint / typecheck / test (copiados de package.json / CI)
- Recibe cambios de un origen Lovable: <sí/no> (ver .upstream-sync.yml)

## Decisiones de esta app (mantener al día)
| Tema | Decisión | Por qué |
|---|---|---|
| Módulos | | |
| Roles | | |
| Entidades | | |
| Procesos asíncronos | | |
| Variables propias | | |
| Integraciones / modo degradado | | |
```

---

## 17. Mapa de fuentes (para profundizar)

| Tema | Archivo |
|---|---|
| Guía oficial de despliegue (29-10-2026) | `Desktop\proyectos\transversal\horizon\Guía de despliegue en Horizon.pdf` |
| Plan de mitigación (estructura de docs, manifiesto, `/saude`) | `Documents\GitHub\PLAN-MITIGACION-HORIZON.md` |
| Fricciones detalladas (Property) | `lab-qa-ia-lab-03\FRICCIONES_DESPLIEGUE_HORIZON.md` |
| Lineamientos Horizon v2 (norma vigente) | `lab-qa-ia-lab\docs\LINEAMIENTOS-AVLA.md` |
| Lineamientos v1 (obsoleto en auth/puerto) | `sii-data-extractor\docs\LINEAMIENTOS-AVLA.md` |
| Reglas Horizon más nuevas + perfiles + revisión | `lab-qa-ia-lab-08\.github\upstream-sync\reglas\*.md` |
| Flujo upstream-sync (scripts, prompts) | `lab-qa-ia-lab-08\.github\upstream-sync\` |
| Estándar Node más reciente | `lab-qa-ia-lab-08\Dockerfile`, `deploy\horizon.env.example`, `DEPLOY-HORIZON.md` |
| Caso de migración completo (planes, ADRs, trampas) | `lab-qa-ia-lab-01\docs\PLAN_MIGRACION.md`, `PLAN_MVP_HORIZON.md`, `CONTEXTO_MIGRACION.md`, `docs\adr\` |
| Migración desde Lovable/TanStack | `lab-qa-ia-lab-06\docs\DESPLIEGUE-HORIZON.md` |
| Migración desde n8n | `consultas-externas\docs\MIGRACION-N8N.es.md` |
| Pruebas de migraciones Alembic | `lab-qa-ia-lab-04\tests\bd\test_migraciones.py` |
| Consumo del gateway IA | `spread-financeiro\docs\consumir-gateway-horizon.es.md`, `gateway-ia-avla\docs\idempotencia.es.md`, `lgpd-base-legal.es.md` |
| Registro en el hub | `avla-miniapps-hub\backend\apps.yaml`, `backend\README.md`, `DEPLOY-HORIZON.md` |
| CI más estricto (familia hub) | `avla-miniapps-hub\.github\workflows\ci.yml` |
| Manifiestos K8s de producción (referencia core) | `consultas-externas\deploy\k8s\*.yaml` |
| Lineamientos core / Manual de Ingeniería | `Desktop\proyectos\modelos\avla-proyecto-modelo\` (`CLAUDE.md`, `guia-avla\`, `docs\manual\*.es.md`) |
| Despliegue en LAN (no Horizon) | `Documents\GitHub\ESTRATEGIA-DESPLIEGUE-LAN.md` |

Documentos **obsoletos o superados** (no usarlos como norma): `avla-miniapps-hub\docs\LABORATORIO-HORIZON*.md` (§ de JWT del lab), `avla-miniapps-hub\docs\COMO-AGREGAR-APP*.md` (handoff ES384), README §Horizon de lab-02/lab-03 (inyección de `PG*`/`BUCKET_NAME`), `avla-hub\apps\spread\guia-avla\lineamientos-avla.md` (stack viejo).
