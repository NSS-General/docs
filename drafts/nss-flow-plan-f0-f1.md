# NSS Flow — Plan F0 + F1

**Plataforma CRM multifuente y motor de seguimiento comercial de NSS Legal & Tax.**
Plan para aprobación antes de escribir código. Origen: prompt de la reunión
Estrategia Steps del 04-08-2026. Preparado el 08-08-2026.

> **Actualización 08-08-2026 (tarde):** Esteban aprobó partir por la base con su
> workflow (identidad única) sin esperar las decisiones de Clemente. F0 quedó
> construido, verificado y pusheado en `NSS-General/nss-flow` (ver PLAN.md y
> REGISTRO.md de ese repo, que es la copia viva de este plan). La base Postgres
> corre en Supabase con la migración inicial y el seed aplicados. D5 quedó
> resuelta: `nss-skills` ya está habilitado para la App.

---

## Lo que hay que decidir primero

| # | Decisión | Dueño |
|---|---|---|
| D1 | Acceso Gmail: ¿OAuth individual por socio, o delegación de dominio con service account (como ya opera Ark)? Define la lista de credenciales y el aislamiento por identidad | Clemente |
| D2 | Dónde corre NSS Flow (VPS propio / Cloud Run / Railway). Propongo Cloud Run + Cloud SQL por backups y cero fierro, pero es plata y arquitectura | Clemente |
| D3 | Auth del dashboard: propongo Google OAuth restringido a dominio `nss.cl`, roles por lista en DB | Clemente |
| D4 | Cifrado en reposo de teléfono/RUT/email (Ley 21.719): pgcrypto en Postgres vs. cifrado a nivel de aplicación. Propongo columnas cifradas a nivel app con clave en env | Clemente |
| D5 | Habilitar `NSS-General/nss-skills` para la GitHub App de Claude — **no pude leerlo** (detalle abajo) y F2/F5 lo necesitan | Clemente |
| D6 | Confirmar SLAs de enfriamiento (tabla abajo) y canal del brief (`#claude-nss` o `#prospects`) | Esteban |
| D7 | ¿Quién confirma los leads de `#prospects`? (reacción ✅ crea el prospecto) | Esteban |
| D8 | Seed inicial F1: cargo a mano Cabrera, Ferraro y Santillana + lo que hoy esté vivo en `00. Comercial/*`, para que el dashboard nazca con data real. ¿Ok? | Esteban |
| D9 | Ruteo entre skills comerciales y NSS Flow (F5): qué skill pregunta qué | Ambos |

Nada de F0/F1 se bloquea por D5–D9; D1–D4 sí bloquean el arranque de F1 (D1) y de F0 (D2–D4).

---

## Qué se verificó contra los sistemas reales (08-08-2026)

Verificado en vivo vía MCP (sesión de Esteban), no de memoria:

**Vambe** ✅
- `NSS - Embudo Nacional` `aa4c5d7d-50d6-4d18-bf58-086d9cfba501`, tz `America/Santiago`.
  Etapas reales: Calificador · Cliente Nacional · Agendados Nacional · **`Reunión / Venta`**
  (con espacios alrededor de la barra — el prompt decía «Reunión/Venta») · Propuesta Enviada ·
  Ganado · No Calificado o Perdido · Asistencia Humana · Basura · +5 «Reforma Trib.»
  (quedan fuera del embudo comercial y de los KPIs).
- `NSS - Internacional` `a6aa49e1-52e6-4183-89bd-51fa6df1f886`. Etapas reales con sufijos
  **completos**: `Ganado - Internacional`, `No Calificado o Perdido - Internacional`,
  `Asistencia Humana - Internacional` (el prompt abreviaba «- Int.»).
- Canales WhatsApp id `18822` (+56 9 6399 9429) y `39877` (+56 9 7315 8437), ambos SOLO en
  el nacional. El internacional no tiene canal conectado. Confirmado.
- **El mapeo de etapas del código usará estos nombres reales**, no los del prompt.

**Slack** ✅ — los 8 canales del prompt existen con esos IDs exactos (`#prospects`
`GQ035N25Q`, `#claude-nss` `C0BJFRZU7AS`, etc.). Esteban `UFCDL8327`, Clemente `UFBN6P9NV`.

**Gmail — casilla de Esteban** ⚠️ hallazgo importante:
- **Las jerarquías `00. Comercial/` y `0. Skills/` NO existen en la casilla de Esteban.**
  Tampoco `0. Internacional/`. `Clientes NSS/` existe con solo 10 sub-etiquetas.
  La taxonomía madura del prompt es la de Clemente (no verificable desde esta sesión).
- Consecuencia de diseño: la sincronización de etiquetas (F2) necesita un **bootstrap
  idempotente por casilla** — crear las 7 etiquetas del mapeo si faltan, avisando. En F1 la
  ingesta es solo lectura y no depende de esto.

## Lo que NO pude leer o verificar — no lo reconstruyo de memoria

| Qué | Estado | Impacto |
|---|---|---|
| Repo `NSS-General/nss-skills` (CLAUDE.md, REGISTRO.md, las 4 skills comerciales) | La GitHub App de esta sesión no lo tiene habilitado (D5) | Las convenciones las tomé de `nss-skills-contabilidad` (sí leído: mismo motor de linter, REGISTRO.md, cultura de 0 errores). Los REG citados en el prompt (REG-001, REG-021, REG-032, REG-035) van asumidos del prompt, sin re-verificar |
| Casilla de Clemente | No accesible desde la sesión de Esteban | Sus etiquetas y volumen (780/374/93) quedan como dato del prompt |
| Webhook de Webflow | No existe integración verificada; hoy los formularios llegan por correo | F1 ingiere Webflow vía webhook nuevo + poll de respaldo; mientras no haya credenciales, el fallback es el correo del formulario en Gmail |
| Programación 07:45/17:30 | No existe hoy | Llega en F3 (BullMQ); en F1 el barrido se corre a mano |

---

## Arquitectura

Stack fijado por el prompt: TypeScript, Node 20+, PostgreSQL + Prisma, Redis + BullMQ,
Zod en todo input externo, **Fastify** (más liviano que NestJS y sin magia — elección mía,
reversible), Next.js + Tailwind + shadcn/ui, Vitest + Playwright.

```
nss-flow/
├── packages/
│   ├── core/      # dominio puro: entidades, máquina de estados, reglas, identidad. CERO I/O
│   ├── db/        # Prisma client + repositorios + AuditLog  ← agrego este paquete al árbol
│   │              #   del prompt para que core siga sin infraestructura (lo demás, igual)
│   ├── ingest/    # un adaptador por canal, misma interfaz
│   ├── engine/    # barridos, secuencias, SLA, briefs (F1: solo barrido manual)
│   ├── api/       # Fastify: REST + webhooks + RBAC + audit middleware
│   ├── web/       # dashboard Next.js
│   └── mcp/       # F5 — servidor MCP (vacío hasta F5)
├── prisma/        # schema + migraciones
├── plugin/        # F5 — skills que consumen el MCP
└── tests/         # integration + e2e + fixtures
```

Reglas de la casa que hereda el repo desde el día 0: linter de existencia (adaptado del
`validate.py` de la familia nss-skills: nada puede citar archivos/IDs inexistentes),
`REGISTRO.md` de discrepancias, CI verde obligatorio, **branch protection en `main` con PR
requerido** (explícito del prompt: acá no se desactiva), CODEOWNERS solo con handles
reales.

---

## F0 — Esqueleto (todo compila, nada opera)

Entregable: repo con CI verde, DB migrada, RBAC y auditoría funcionando, dashboard que
levanta con login y página vacía de prospectos, healthcheck.

Archivos concretos:

| Área | Archivos |
|---|---|
| Raíz | `pnpm-workspace.yaml`, `package.json`, `tsconfig.base.json`, `.env.example` (documentado), `.gitignore`, `docker-compose.dev.yml` (Postgres+Redis efímeros), `README.md`, `REGISTRO.md`, `CLAUDE.md`, `.github/CODEOWNERS` |
| CI | `.github/workflows/ci.yml` — lint → typecheck → test → build (e2e se suma en F1); resultado posteado a `#claude-nss` |
| Linter | `scripts/validate.py` portado de la familia nss-skills + `scripts/linter-config.json` |
| Prisma | `prisma/schema.prisma` (abajo), `prisma/migrations/`, `prisma/seed.ts` (identidades + roles) |
| core | `src/domain/types.ts` (enums y entidades), `src/stateMachine.ts` (etapas y transiciones válidas), `src/rules/loader.ts` + `core/rules/*.yaml` (declarativas, editables sin tocar TS), `src/identity/{normalize.ts,rut.ts,resolve.ts}` — con tests unitarios desde el primer commit (TDD, cobertura ≥90% en core) |
| db | `src/client.ts`, `src/repos/*.ts`, `src/audit.ts` (toda escritura pasa por acá y deja `AuditLog`) |
| api | `src/server.ts`, `src/plugins/{auth.ts,rbac.ts,audit.ts}`, `src/routes/healthz.ts` |
| web | Next.js con auth (D3), layout, `/prospectos` vacío, `/salud` (última ingesta por identidad×canal — nace en F0 aunque muestre «nunca») |

RBAC desde F0: `socio` (Esteban, Clemente), `equipo`, `ops` (Steps, lectura de KPIs),
`readonly`, `bot`. Secretos solo por env; credenciales **separadas por identidad** — el
token de Esteban no puede leer la casilla de Clemente.

## F1 — Barrido e identidad (ya hay valor: nadie cuenta a mano)

Entregable: correr `pnpm sweep` (o `POST /api/sweep`) ingiere Gmail ×2, Vambe y Webflow,
deduplica cross-identidad, y **el dashboard muestra la tabla viva de prospectos con su
detalle** — la vista que pediste: en qué está cada prospecto, alimentada por todas las
fuentes.

| Área | Archivos |
|---|---|
| ingest | `src/adapter.ts` (interfaz común: `pull(identity, watermark) → RawEvent[]`), `src/gmail/{adapter.ts,parse.ts,webflowEmail.ts}`, `src/vambe/{adapter.ts,mapStages.ts}`, `src/webflow/{webhook.ts,poll.ts}` |
| core | `src/identity/resolve.ts` completo: orden (1) externalId visto (2) email (3) tel E.164 +56 (4) RUT con DV (5) dominio+Jaro-Winkler>0.9; entre 0.75–0.9 crea con `needsReview`, **nunca fusión automática bajo 0.9**. Regla de owner: primer saliente > primer receptor > empate=`needsReview`; nunca se reasigna solo |
| engine | `src/sweep.ts` — por (identidad × canal), watermark en DB, idempotente |
| api | `src/routes/{prospects.ts,sweep.ts,webhooks/webflow.ts}` (firma verificada + rate limit) |
| web | `/prospectos` (tabla: etapa, canal, owner, días en etapa, semáforo, filtros pipeline/portafolio/owner/canal) · `/prospectos/[id]` (timeline multicanal con link al original: hilo Gmail, conversación Vambe, formulario) · `/decisiones` (needsReview y merges pendientes — versión mínima) |
| tests | Fixtures reales ≥5 por canal (incl. malformados) en `tests/fixtures/<canal>/`; **test de idempotencia** (mismo barrido 2× = cero duplicados); **test cross-identidad** (mismo prospecto por las dos casillas = 1 `Prospect`, 2 `Interaction`); unit de RUT/E.164/similitud; integration con Postgres/Redis efímeros |

En F1 la ingesta es **solo lectura** (no etiqueta, no mueve Vambe, no manda nada). La
escritura hacia Gmail/Vambe llega en F2 con el mapeo normativo y el test de exclusividad
de etiquetas. Los clasificadores LLM también son F2 — en F1 la calificación es
heurística simple + `needsReview`.

## Esquema Prisma propuesto

```prisma
enum PipelineKind { NACIONAL INTERNACIONAL }
enum Stage { NUEVO CALIFICANDO CALIFICADO REUNION_AGENDADA REUNION_HECHA
             PROPUESTA_ENVIADA CONTRATO_ENVIADO GANADO PERDIDO DORMIDO
             ASISTENCIA_HUMANA BASURA }
enum Channel { GMAIL VAMBE WEBFLOW SLACK CALENDAR ADS MANUAL }
enum IdentityKey { ESTEBAN CLEMENTE }
enum Direction { IN OUT }
enum ActorType { HUMAN AGENT }
enum TriggeredBy { RULE HUMAN AGENT }
enum Currency { CLP UF USD }
enum Role { SOCIO EQUIPO OPS READONLY BOT }

model Contact {
  id           String  @id @default(cuid())
  fullName     String
  rut          String? @unique          // cifrado (D4)
  passport     String?
  linkedinUrl  String?
  mergedIntoId String?
  mergedInto   Contact?  @relation("Merge", fields: [mergedIntoId], references: [id])
  mergedFrom   Contact[] @relation("Merge")
  emails       ContactEmail[]
  phones       ContactPhone[]
  prospects    Prospect[]
  referrals    Referral[] @relation("Referrer")
  createdAt    DateTime @default(now())
}
model ContactEmail { id String @id @default(cuid()); contactId String; contact Contact @relation(fields:[contactId],references:[id]); email String @unique }   // normalizado, cifrado (D4)
model ContactPhone { id String @id @default(cuid()); contactId String; contact Contact @relation(fields:[contactId],references:[id]); e164 String @unique }    // cifrado (D4)

model Company {
  id String @id @default(cuid())
  legalName String; tradeName String?; rut String? @unique; domain String? @unique
  country String; industry String?; headcountBand String?; revenueBand String?
  prospects Prospect[]
}

model Prospect {
  id            String @id @default(cuid())
  contactId     String;  contact Contact @relation(fields:[contactId],references:[id])
  companyId     String?; company Company? @relation(fields:[companyId],references:[id])
  portfolio     String?               // uno de los 7; validado en core, no enum DB (cambian)
  pipeline      PipelineKind
  stage         Stage @default(NUEVO)
  source        Channel
  sourceDetail  String?
  ownerIdentity IdentityKey?
  serviceInterest String[]
  estimatedValue Decimal?             // INTERNO. Jamás sale a cliente (regla de oro 1)
  currency      Currency @default(CLP)
  stageEnteredAt DateTime @default(now())
  nextActionAt  DateTime?
  nextActionType String?
  closedAt      DateTime?
  closeReason   String?
  referredById  String?
  needsReview   Boolean @default(false)
  interactions  Interaction[]
  transitions   StageTransition[]
  tasks         Task[]
  proposals     Proposal[]
  externalLinks ExternalLink[]
  campaignTouches CampaignTouch[]
  @@index([stage, nextActionAt])
  @@index([ownerIdentity, stage])
}

model Interaction {
  id          String @id @default(cuid())
  prospectId  String; prospect Prospect @relation(fields:[prospectId],references:[id])
  channel     Channel
  direction   Direction
  ingestedVia IdentityKey
  occurredAt  DateTime
  externalId  String
  summary     String
  rawRef      String                  // link/ID al original — el dato crudo no se copia
  intentTags  String[]
  isNameSource Boolean @default(false)
  @@unique([channel, externalId])     // la dedup de eventos es de DB, no de código
}

model StageTransition {
  id String @id @default(cuid())
  prospectId String; prospect Prospect @relation(fields:[prospectId],references:[id])
  from Stage; to Stage; at DateTime @default(now())
  triggeredBy TriggeredBy
  ruleId String?
  evidenceInteractionId String?
  confidence Float
  promptVersion String?
}

model Task     { id String @id @default(cuid()); prospectId String; prospect Prospect @relation(fields:[prospectId],references:[id]); type String; dueAt DateTime; assigneeIdentity IdentityKey; status String; createdBy String }
model Proposal { id String @id @default(cuid()); prospectId String; prospect Prospect @relation(fields:[prospectId],references:[id]); docUrl String; version Int; amount Decimal?; currency Currency; services String[]; sentAt DateTime?; status String }
model Referral { id String @id @default(cuid()); referrerContactId String; referrer Contact @relation("Referrer", fields:[referrerContactId],references:[id]); prospectId String; agreedPct Float; commission Decimal?; status String }
model CampaignTouch { id String @id @default(cuid()); prospectId String; prospect Prospect @relation(fields:[prospectId],references:[id]); platform String; campaignId String; utm Json; costAttributed Decimal? }

model AuditLog {
  id String @id @default(cuid())
  actor String; actorType ActorType
  action String; entity String; entityId String
  before Json?; after Json?
  promptVersion String?
  at DateTime @default(now())
  @@index([entity, entityId])
}

model ExternalLink { id String @id @default(cuid()); prospectId String; prospect Prospect @relation(fields:[prospectId],references:[id]); system String; externalId String; url String; @@unique([system, externalId]) }
model SweepWatermark { id String @id @default(cuid()); identity IdentityKey; channel Channel; lastRunAt DateTime; cursor String?; @@unique([identity, channel]) }
model AppUser { id String @id @default(cuid()); email String @unique; name String; role Role; slackUserId String? }
```

Notas: nada se borra duro (merge deja `mergedIntoId`; cierres llevan motivo);
`ContactEmail`/`ContactPhone` como tablas (no arrays) para que la dedup tenga índice
único; `portfolio` validado en `core` y no como enum de DB para poder agregar portafolios
sin migración.

## Credenciales exactas que necesito de ustedes

| Credencial | Para | Fase | Quién la entrega |
|---|---|---|---|
| `DATABASE_URL` (Postgres) y `REDIS_URL` | Todo / colas | F0 | Clemente (según D2) |
| Gmail Esteban + Clemente — según D1: OAuth por socio con scope `gmail.readonly` (F1) → `gmail.modify` + `gmail.compose` (F2/F3); o service account con delegación y esos scopes | Ingesta ×2, luego etiquetas y borradores | F1 | Cada socio / Clemente |
| Google Calendar `calendar.readonly` ×2 (mismo mecanismo D1) | Señal REUNIÓN_AGENDADA | F2 | Ídem |
| API key de Vambe (la del MCP ya operativo) | Ingesta WhatsApp + etapas | F1 | Esteban |
| Webflow: API token del sitio + signing secret del webhook de formularios | Webhook + poll de respaldo | F1 | Matías |
| Slack bot token: `chat:write` (briefs), `channels:history` + `groups:history` de `#prospects` y `#claude-nss`, `reactions:read` | Briefs y confirmaciones | F3 | Clemente |
| Google Ads + Meta: tokens de lectura de campañas/costos | CAC | F4 | Matías |
| `ENCRYPTION_KEY` (si D4 = app-level) | Cifrado PII | F0 | Se genera y se guarda en el vault que digan |

Todo por env, `.env.example` documentado, nada en el repo (regla ya vigente en la familia
nss-skills).

## Supuestos declarados

1. El barrido corre **por identidad** con credenciales separadas; un solo despliegue, dos
   contextos de credencial (no dos instancias).
2. Los formularios de Webflow seguirán llegando también por correo — es el respaldo si el
   webhook se cae, además del poll.
3. Los nombres de etapas de Vambe verificados hoy son los vigentes; si alguien renombra
   una etapa en Vambe, el mapeo falla ruidosamente (test de contrato contra `list-stages`
   en cada barrido — no se "adivina" el mapeo).
4. F1 no escribe en ningún sistema externo. La primera escritura (etiquetas/Vambe) llega
   en F2 con sus tests de exclusividad.
5. El monto `estimatedValue` es interno; ninguna vista pública/export lo presenta como
   cotización (regla de oro 1).
6. Los 3 prospectos internacionales del 31-07 (Cabrera, Ferraro, Santillana) se cargan
   como seed manual en F1 (D8) — nombres a confirmar contra correos, no contra
   transcripciones (ya hubo «Ferrero»/«FNR» mal transcritos).

## Criterios de aceptación de F1 (del prompt, verbatim los que aplican)

1. Un lead de cualquier canal (Gmail ×2, Vambe, Webflow), a cualquiera de los dos socios,
   aparece en NSS Flow en el barrido siguiente, con contacto resuelto y sin duplicar.
2. El mismo prospecto escribiendo a los dos socios es UN registro con DOS interacciones.
3. El dashboard muestra la tabla y el detalle/timeline de cada prospecto con link al
   original en su canal.
4. Correr el barrido dos veces no duplica nada (test automático).
5. Si un canal se cae: se registra en `/salud`, se alerta, el resto sigue. Nada se inventa.

## Qué viene después (para contexto, no para construir ahora)

F2 embudo + sincronización etiquetas/Vambe + Kanban + bandeja de decisiones completa ·
F3 barridos programados 07:45/17:30 + secuencias + briefs · F4 KPIs/directorio + Ads/CAC ·
F5 MCP + enganche con las 4 skills comerciales · F6 referidos + toolkit SDR.
Cada fase termina de punta a punta antes de la siguiente — sin construir en paralelo.
