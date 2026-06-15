# Arquitetura

## Princípios

1. **Comece simples, escale por evidência** — monólito modular agora; microserviço quando o domínio exigir.
2. **DDD tático** — módulos por bounded context, baixo acoplamento, alta coesão.
3. **12-Factor App** — config no ambiente, stateless, logs como stream, paridade dev/prod.
4. **Stateless + escala horizontal** — qualquer instância atende qualquer request.
5. **Event-driven onde agrega** — notificações, auditoria e intake via fila.

## C4 — Nível 1: Contexto

```mermaid
C4Context
    title DevFlow — Contexto
    Person(solicitante, "Solicitante", "Áreas de negócio")
    Person(timeTI, "Time de TI", "Triador, Dev, Gestor")
    Person(admin, "Admin", "Governança")
    System(devflow, "DevFlow", "Portal de Gestão de Demandas de TI")
    System_Ext(whatsapp, "WhatsApp Business API", "Intake")
    System_Ext(email, "E-mail", "Intake e notificações")
    System_Ext(idp, "Identity Provider", "SSO OIDC")
    System_Ext(llm, "LLM Provider", "Classificação por IA")
    System_Ext(legado, "CRM / Legados (PHP)", "Integrações")
    Rel(solicitante, devflow, "Abre e acompanha")
    Rel(timeTI, devflow, "Tria, executa, homologa")
    Rel(admin, devflow, "Administra")
    Rel(whatsapp, devflow, "Webhook")
    Rel(devflow, email, "Notifica")
    Rel(devflow, idp, "Autentica (OIDC)")
    Rel(devflow, llm, "Classifica/prioriza")
    Rel(devflow, legado, "Integra (Anti-Corruption Layer)")
```

## C4 — Nível 2: Container

```mermaid
C4Container
    title DevFlow — Containers
    Person(user, "Usuário", "Solicitante / TI / Admin")
    System_Ext(whatsapp, "WhatsApp Business API", "Meta")
    System_Ext(llm, "LLM Provider", "IA")
    Container_Boundary(devflow, "DevFlow") {
        Container(spa, "Front-End", "Next.js 15 / React", "UI SSR + RSC")
        Container(api, "API / BFF", "NestJS", "Monólito modular por bounded context")
        Container(worker, "Workers", "NestJS + BullMQ", "Notificações, SLA, intake IA")
        ContainerDb(pg, "PostgreSQL 16", "RDBMS", "Demandas, usuários, auditoria, histórico")
        ContainerDb(redis, "Redis", "Cache + Fila", "Cache + filas BullMQ")
        ContainerDb(storage, "Object Storage", "S3/MinIO", "Anexos")
    }
    Rel(user, spa, "HTTPS")
    Rel(spa, api, "REST/JSON")
    Rel(api, pg, "SQL (Prisma)")
    Rel(api, redis, "Cache + enfileira")
    Rel(worker, redis, "Consome filas")
    Rel(worker, pg, "Persiste")
    Rel(api, storage, "Anexos")
    Rel(whatsapp, api, "Webhook")
    Rel(worker, llm, "Classifica")
```

## C4 — Nível 3: Módulos da API (bounded contexts)

```
src/modules/
├── identity/        # Usuários, departamentos, perfis, auth
├── demands/         # CORE: Demanda (aggregate root), status, prioridade, workflow
├── comments/        # Comentários
├── attachments/     # Anexos (S3)
├── sla/             # Cálculo e monitoramento de SLA
├── notifications/   # E-mail, WhatsApp, in-app (via fila)
├── audit/           # Trilha de auditoria append-only
├── intake/          # Webhook WhatsApp/e-mail + classificação IA
└── reports/         # Dashboards e exportações
```

Cada módulo é um bounded context isolado, comunicando-se por interfaces/eventos. Um módulo que
precise escalar sozinho (ex.: `intake`, `notifications`) já está pronto para ser extraído como
microserviço — trocando a chamada in-process por mensageria.

## Stack e justificativas

| Camada | Escolha | Justificativa |
|---|---|---|
| Front-end | **Next.js 15 (React)** | SSR/RSC para performance e SEO; App Router; mobile-first |
| Back-end | **NestJS (Node/TS)** | Modular nativo (DI) → DDD tático; TypeScript ponta a ponta |
| API | **REST/JSON + OpenAPI** | Contrato e documentação automática |
| DB principal | **PostgreSQL 16** | ACID no workflow; JSONB para flexibilidade; full-text nativo; RLS |
| Cache + Fila | **Redis + BullMQ** | Cache de leitura; jobs assíncronos resilientes |
| Object Storage | **S3 / MinIO** | Anexos fora do RDBMS; URLs pré-assinadas |
| Auth | **JWT + Refresh + OIDC** | Stateless (escala horizontal); SSO corporativo |
| IA | **LLM via API** | Classificação/priorização no intake |
| Infra | **Docker + cloud** | Paridade dev/prod, portabilidade |
| ORM | **Prisma** | Type-safe, migrations versionadas |

## Fluxo de criação de demanda

```mermaid
sequenceDiagram
    actor S as Solicitante
    participant FE as Next.js
    participant API as NestJS
    participant DB as PostgreSQL
    participant Q as Redis/BullMQ
    participant W as Worker
    S->>FE: Abre demanda
    FE->>API: POST /demands (JWT)
    API->>API: Valida (DTO) + autoriza (RBAC)
    API->>DB: INSERT demanda + histórico (transação)
    API->>Q: Enfileira "demanda.criada"
    API-->>FE: 201 Created
    Q->>W: Consome evento
    W->>DB: Registra notificação enviada
```

## Intake inteligente

```mermaid
flowchart LR
    WA[WhatsApp Business API] -->|webhook| IN[Módulo Intake]
    EM[E-mail] -->|parser| IN
    IN -->|texto| LLM[LLM: classifica + prioriza]
    LLM -->|categoria, depto, prioridade| DR[Rascunho de Demanda]
    DR -->|cria| BL[(Backlog estruturado)]
    BL --> TR[Triador valida/ajusta]
```

Demandas que hoje chegam de forma informal no WhatsApp passam a entrar estruturadas: a IA lê a
mensagem, classifica e cria a demanda já priorizada no backlog.

## Architecture Decision Records (ADRs)

### ADR-001 — Monólito modular em vez de microserviços
- **Contexto:** 100 usuários hoje, meta 50k; time enxuto.
- **Decisão:** Monólito modular (módulos por bounded context).
- **Alternativa rejeitada:** Microserviços desde o início — complexidade operacional, latência de rede e transações distribuídas sem necessidade.
- **Consequência:** Velocidade agora; caminho de extração claro depois.

### ADR-002 — PostgreSQL como banco principal
- **Decisão:** Postgres 16 relacional.
- **Porquê:** Workflow exige ACID; histórico/auditoria exigem integridade; JSONB cobre flexibilidade sem NoSQL dedicado.

### ADR-003 — Comunicação assíncrona por fila
- **Decisão:** Notificações, SLA e IA em workers via BullMQ/Redis.
- **Porquê:** Não bloquear o request; resiliência (retry); desacoplamento.

### ADR-004 — Auth stateless (JWT) + SSO OIDC
- **Decisão:** JWT curto + refresh + SSO corporativo.
- **Porquê:** Escala horizontal sem sessão pegajosa; reaproveita identidade corporativa.
