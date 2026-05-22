# gbrain-nan-docker

Deploy [gbrain](https://github.com/garrytan/gbrain) con [api.nan.builders](https://api.nan.builders) como proveedor de IA, todo en un solo contenedor.

## Stack

| Componente | Rol |
|---|---|
| **gbrain** | Cerebro persistente para agentes de IA (MCP + admin dashboard) |
| **LiteLLM** | Proxy que traduce OpenAI-compatible → nan.builders |
| **anthropic-shim** | Normalizador de paths para los SDKs Anthropic que usa gbrain |
| **nginx** | Reverse proxy con soporte SSE (embebido en el mismo contenedor) |
| **Postgres + pgvector** | Base de datos con búsqueda vectorial (embebida) |

## Requisitos

- Docker + Docker Compose
- API key de [api.nan.builders](https://api.nan.builders)

## Uso rápido

```bash
cp .env.example .env
# Editar .env con tu NAN_API_KEY y LITELLM_MASTER_KEY
docker compose up -d
```

- **Admin dashboard**: `http://localhost:8080/admin/`
- **MCP endpoint**: `http://localhost:8080/mcp`
- **Health**: `http://localhost:8080/health`

PostgreSQL arranca automáticamente dentro del contenedor. Los datos persisten en el volumen `gbrain-data` montado en `/data`.

```
/data/
├── pg/             ← cluster PostgreSQL (tablas, schemas, WAL)
└── gbrain-home/    ← configuración, auditoría, evaluaciones, clones
```

## Configuración

### `.env`

```env
NAN_API_KEY=sk-...                          # Tu API key de nan.builders (obligatorio)
LITELLM_MASTER_KEY=sk-local-...            # Clave maestra del proxy LiteLLM (obligatorio)
PUBLIC_URL=https://gbrain.tudominio.com    # Opcional. Para exponer con HTTPS
```

### Modelos

| Tier gbrain | Modelo nan.builders | Uso |
|---|---|---|
| `claude-opus-4-7` | `deepseek-v4-flash` | Razonamiento profundo |
| `claude-sonnet-4-6` | `qwen3.6` | Chat default |
| `claude-haiku-4-5-20251001` | `gemma4` | Tareas ligeras |
| — | `qwen3-embedding` (MRL 1536d) | Embeddings |

## Comandos comunes

```bash
# Meter contenido
docker compose exec gbrain gbrain put notas/mi-idea "# Título"

# Embeber para búsqueda
docker compose exec gbrain gbrain embed --all

# Buscar
docker compose exec gbrain gbrain query "qué dijo X sobre Y?"

# Síntesis con LLM
docker compose exec gbrain gbrain think "resume mi brain"

# Health check
docker compose exec gbrain gbrain doctor --fast
```

## Exponer a internet

El contenedor expone el puerto 80 internamente (mapeado a `8080` en el compose). Si lo pones detrás de un reverse proxy externo, define `PUBLIC_URL` para que el OAuth funcione correctamente:

```bash
PUBLIC_URL=https://gbrain.midominio.com
```

## Arquitectura

```
                          gbrain CLI
                              │
            ┌─────────────────┼──────────────────┐
            │                 │                  │
      embeddings           chat/expansion       subagent
            │                 │                  │
            │           ANTHROPIC_BASE_URL       │
            │          → localhost:4001           │
            │                 │                  │
            ▼                 ▼                  ▼
   LITELLM_BASE_URL    ┌──────────────────────────┐
   → localhost:4000    │ anthropic-shim (bun)     │  :4001
            │          │ /messages → /v1/messages │
            └─────┬────└──────────────────────────┘
                  │                  │
                  ▼                  ▼
          ┌──────────────────────────────┐
          │ LiteLLM proxy               │  :4000
          │ - openai/gemma4             │
          │ - openai/qwen3.6            │
          │ - openai_like/qwen3-embedding│
          │ - claude-opus→deepseek-v4   │
          │ - claude-sonnet→qwen3.6     │
          │ - claude-haiku→gemma4       │
          └──────────────┬───────────────┘
                         ▼
               api.nan.builders
```

## Licencia

MIT