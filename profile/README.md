# Centinela

Plataforma científica multi-servicio para investigadores: búsqueda de literatura académica (integrada con Scopus), análisis de redes de coautoría, consenso social sobre publicaciones, predicción de producción científica y búsqueda semántica asistida por LLM (RAG).

El sistema está dividido en varios repositorios independientes, cada uno con su propio ciclo de vida, historial y pipeline de CI/CD, orquestados como microservicios sobre una red Docker compartida.

Los repos públicos de este org son de solo lectura para quien no sea miembro — se puede clonar y explorar libremente, pero escribir (push, merge) requiere ser parte de la organización. No hay flujo de contribución externa abierto.

## Arquitectura

```mermaid
flowchart TB
    Cliente(["Cliente / navegador"]) --> Frontend

    subgraph Frontend["centinela-front — Angular 16"]
        direction TB
        FE[Estáticos + proxy]
    end

    Frontend -->|"/api /ws /media"| Gateway

    subgraph Gateway["gateway-service — nginx"]
        direction TB
        GW["Routing central + auth_request"]
    end

    Gateway --> Identity
    Gateway --> Social
    Gateway --> Search
    Gateway --> SearchBff
    Gateway --> Predictive
    Gateway --> RAG

    subgraph Identity["identity-service"]
        direction TB
        ID["Django + SimpleJWT"] --> IDDB[(PostgreSQL)]
    end

    subgraph Social["social-service"]
        direction TB
        SO["Django + Daphne + Channels + Celery"] --> SODB[(PostgreSQL pgvector)]
        SO --> SORD[(Redis)]
    end

    subgraph Search["search-service"]
        direction TB
        SE["Django + Gunicorn"] --> SENEO[(Neo4j)]
        SE --> SEMONGO[(MongoDB)]
        SE -.->|Scopus API| SCOPUS[(Scopus)]
    end

    subgraph SearchBff["search-bff-service"]
        direction TB
        MS["FastAPI — bridge v2→v1"]
    end
    MS -.-> SE

    subgraph Predictive["predictive-service"]
        direction TB
        PR["FastAPI + LightGBM"] --> PRFILE[("Modelo pre-entrenado")]
    end

    subgraph RAG["rag-service"]
        direction TB
        RG["FastAPI + FAISS + reranking"] --> OLLAMA["Ollama — gemma3:4b"]
    end
```

Todos los backends comparten una red Docker externa y se resuelven entre sí por alias de servicio (nunca `localhost`). El frontend nunca habla directo con un backend — todo pasa por `gateway-service`, que centraliza el ruteo `/api/<servicio>/` y la validación de auth.

## Repositorios

| Repo | Rol | Stack |
|---|---|---|
| [`centinela-front`](https://github.com/PlataformaIntegradaInvestigadores/centinela-front) | SPA — UI de toda la plataforma | Angular 16 + Material + Tailwind |
| [`gateway-service`](https://github.com/PlataformaIntegradaInvestigadores/gateway-service) | Gateway central de routing y auth | nginx |
| [`identity-service`](https://github.com/PlataformaIntegradaInvestigadores/identity-service) | Autenticación e identidad de usuarios | Django 5 + SimpleJWT + PostgreSQL |
| [`social-service`](https://github.com/PlataformaIntegradaInvestigadores/social-service) | Feed social, consenso, jobs, comentarios (WebSocket) | Django 5 + Channels + Celery + pgvector + Redis |
| [`search-service`](https://github.com/PlataformaIntegradaInvestigadores/search-service) | Motor de búsqueda: grafo de autores/artículos/afiliaciones, integración Scopus, dashboards | Django 5 + Neo4j + MongoDB |
| [`search-bff-service`](https://github.com/PlataformaIntegradaInvestigadores/search-bff-service) | Fachada v2 estable sobre la API v1 de `search-service` | FastAPI |
| [`predictive-service`](https://github.com/PlataformaIntegradaInvestigadores/predictive-service) | Predicción de producción científica por afiliación | FastAPI + LightGBM |
| [`rag-service`](https://github.com/PlataformaIntegradaInvestigadores/rag-service) | Búsqueda semántica asistida por LLM sobre el corpus de Scopus | FastAPI + FAISS + Ollama (Gemma 3) |

## Patrón de arquitectura

Los 6 backends (Django y FastAPI) siguen el mismo esquema de capas — **Clean/Hexagonal Architecture** — para que moverse entre repos sea predecible:

```
<módulo>/
  domain/            # entidades y contratos de repositorio, sin dependencias externas
    entities/
    repositories/
  application/        # casos de uso y lógica de orquestación
    use_cases/
    services/
  infrastructure/      # el mundo exterior: HTTP, DB, migraciones
    api/v1/
      views/            # (o endpoints/ en FastAPI)
      serializers/       # (o schemas/ en FastAPI)
      urls/
    migrations/
  tests/               # test_*.py (pytest / pytest-django)
```

Los repos Django (`social-service`, `search-service`, `identity-service`) y los FastAPI (`search-bff-service`, `predictive-service`, `rag-service`) mapean este mismo patrón a su forma idiomática. `predictive-service` es una variante más liviana (sin `use_cases/` explícito, servicio de un solo propósito).

## CI/CD

Cada repo de servicio corre su propio pipeline en **GitHub Actions**:

1. **Tests unitarios** (+ de integración donde aplica) con bases de datos reales en contenedores efímeros del job.
2. **Gate de cobertura**: mínimo **90%**. El build falla si no se cumple.
3. **Build de imagen Docker**.
4. **Deploy automático a staging** al mergear a `develop`, con healthcheck y rollback automático si falla.

`main` se mantiene protegida y solo recibe cambios validados en `develop`.

## Convenciones

- **Branches**: `feature/*` → `develop` · `hotfix/*` → `main` · `chore/*` para tareas de mantenimiento.
- **Commits**: [Conventional Commits](https://www.conventionalcommits.org/) (`feat:`, `fix:`, `chore:`, `test:`, `refactor:`, `docs:`), en inglés, con el *por qué* en el cuerpo.
- **PRs**: requieren aprobación de miembros del equipo, todas las conversaciones resueltas. Plantilla centralizada en [`.github/pull_request_template.md`](../pull_request_template.md).
- **Código**: inglés para variables, funciones, comentarios, rutas. Sin comentarios decorativos — solo lo no obvio.
- Cada repo trae su propio `README.md` con instrucciones detalladas para levantarlo, sus variables de entorno y cómo correr sus tests.

## Cómo empezar

1. Clonar los repos que necesites junto a este org (mismo directorio padre).
2. Crear la red Docker compartida indicada en cada `docker-compose.yml`.
3. Seguir el `README.md` de cada repo — todos siguen la misma estructura (stack, cómo levantar, tests, CI/CD).
