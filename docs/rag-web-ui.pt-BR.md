> Tradução para pt-BR de rag-web-ui.md (commit 22fe360). Em caso de divergência, vale o original.

# RAG e interface web: arquitetura e fases

Objetivo: **não** inchar o `claw-analog` nem o `claw` principal — levar a indexação e (mais tarde) a interface para processos separados, com contratos HTTP/MCP explícitos.

## Princípios

1. **RAG como serviço**: binário separado (hoje `claw-rag-service`), com ciclo de vida, segredos (API de embeddings) e armazenamento próprios.
2. **O agente só chama a recuperação**: no **`claw-analog`**, a ferramenta **`retrieve_context`** → HTTP `POST {RAG_BASE_URL}/v1/query` (base sem o sufixo `/v1`); limites **`rag_timeout_secs`** e **`rag_top_k_max`** no `.claw-analog.toml`; a resposta para o modelo são fragmentos com `path` + `snippet` + `score`.
3. **Interface web**: página mínima **`GET /`** no `claw-rag-service` (estatísticas + formulário `POST /v1/query`); chat com o modelo e "reindexar" pelo navegador, se necessário, mais tarde.

## Componentes (visão-alvo)

```text
┌─────────────────┐     POST /v1/query      ┌──────────────────────┐
│  claw-analog    │ ──────────────────────►│  claw-rag-service    │
│  (+ tool)       │◄──────────────────────│  (embed + vector DB) │
└─────────────────┘     JSON hits          └──────────┬───────────┘
                                                      │
                                             ingest (watch / CLI)
                                                      ▼
                                             workspace files / git tree
```

- **Indexação**: comando ou worker separado (chunking, hash do arquivo, incremental). Armazenamento: no início, SQLite + `sqlite-vec` / cache de embeddings em arquivo; com o crescimento, Qdrant/Chroma em Docker.
- **Embeddings**: HTTP para um endpoint de embeddings compatível com OpenAI/Anthropic ou um modelo local (decisão à parte, por licença e tamanho).
- **Interface web**: autenticação (no mínimo: token + reverse proxy), SSE ou WebSocket para o streaming da resposta do modelo; a interface **não** guarda os segredos do provedor — se o produto decidir assim, passa por proxy no backend.

## Implementação atual

Crate **`rust/crates/claw-rag-service`** (a partir do diretório `rust/`):

### HTTP

- `GET /`: interface de página única (`static/index.html` embutido): contadores vindos de `/v1/stats`, busca via `/v1/query`.
- `GET /health`: `ok`.
- `GET /v1/stats`: `{ "chunks": N, "phase": "1-sqlite" }` (se o banco ainda não existe: `chunks: 0`, `phase`: `1-sqlite-no-db`).
- `POST /v1/query`: corpo `{"query":"...", "top_k":8}`; resposta `{"hits":[{"path","snippet","score"}], "phase":"1-sqlite"|"1-sqlite-empty"|"1-sqlite-no-db"}`.

Busca: **varredura linear** de todos os vetores no SQLite (MVP; para repositórios grandes, planejar Qdrant/sqlite-vec ou ANN em lote).

### Indexação (fase 1)

```powershell
cd D:\path\to\claw-code-main\rust
$env:OPENAI_API_KEY = "sk-..."
cargo run -p claw-rag-service -- ingest -w D:\path\to\repo --db D:\path\to\index.sqlite
cargo run -p claw-analog -- ...   # com RAG_BASE_URL ou rag_base_url no TOML — ferramenta retrieve_context
```

Variáveis de ambiente:

- **`OPENAI_API_KEY`** ou **`CLAW_RAG_OPENAI_API_KEY`**: para chamar `POST …/embeddings`.
- **`CLAW_RAG_EMBEDDING_BASE_URL`**: padrão `https://api.openai.com/v1`.
- **`CLAW_RAG_EMBEDDING_MODEL`**: padrão `text-embedding-3-small`.
- **`CLAW_RAG_DB`**: caminho do SQLite (no ingest/`serve`; o `serve` tem o padrão `.claw-rag/index.sqlite`).
- **`CLAW_RAG_PORT`**: porta HTTP (padrão `8787`).
- **`CLAW_RAG_MOCK_PROVIDERS=1`**: vetores determinísticos sem rede (para testes de CI).

Subir o servidor: `cargo run -p claw-rag-service` ou `cargo run -p claw-rag-service -- serve --db path\to\index.sqlite`.

### Próximas fases

| Fase | Conteúdo |
|------|------------|
| 1 | ~~Ingest + SQLite + embeddings~~ (o básico está feito; melhorias: indexação incremental, ANN, banco vetorial em Docker). |
| 2 | ~~Ferramenta `retrieve_context`~~: `RAG_BASE_URL` / `rag_base_url`, `rag_timeout_secs`, `rag_top_k_max` no `.claw-analog.toml`. |
| 3 | ~~Interface mínima~~: `GET /` + os mesmos `/v1/*` (próximo: chat, botão de reindexar na interface). |

## Riscos e limitações

- Segredos e dados pessoais (PII) no índice; tamanho do índice e custo dos embeddings.
- Coerência com o tratamento de symlink/jail do `claw-analog` — a recuperação não pode "vazar" para fora do workspace.
- Localização (locale) da interface: i18n separado do `AnalogLanguage` do CLI.

## Documentos relacionados

- Execução local de contêineres (se for subir um banco vetorial): [`container.md`](container.md).
- Visão geral do `claw-analog`: [`how_to_run.md`](../how_to_run.md) ([pt-BR](../how_to_run.pt-BR.md)).
