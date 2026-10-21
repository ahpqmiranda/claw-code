> Tradução para pt-BR de concept.md (commit 22fe360). Em caso de divergência, vale o original.

# Conceito do projeto Claw Code

Este documento registra os **objetivos**, a **arquitetura** e os **princípios** do repositório **Claw Code**, a implementação pública em Rust do agente de linha de comando **`claw`** e das ferramentas que o acompanham. Fonte da verdade sobre o código: o workspace no diretório [`rust/`](rust/README.md); cenários de uso: [`USAGE.md`](USAGE.md), [`how_to_run.md`](how_to_run.md) (claw-analog); backlog de ideias: [`futute.md`](futute.md).

A linha de produto separada "do CLI → para um assistente pessoal" (canais/memória/ferramentas/proatividade/sessões) está descrita em [`docs/personal-assistant-roadmap.md`](docs/personal-assistant-roadmap.md).

---

## 1. Propósito do produto

O **Claw Code** é:

1. **O CLI principal `claw`** (`rusty-claude-cli`): agente completo com REPL, OAuth, conjunto ampliado de ferramentas (incluindo bash, MCP, plugins etc.), streaming e integração com os provedores **Anthropic**, APIs **compatíveis com OpenAI** e **xAI**.
2. **`claw-analog`**: camada leve sobre a **mesma camada de API** (crate `api`): conjunto de ferramentas estreito e previsível, só para trabalhar com o sistema de arquivos do workspace, modos de permissão explícitos, adequado para **CI**, **scripts** e **agentes externos** (NDJSON).
3. **`claw-rag-service`**: processo separado que faz a **indexação** do repositório (chunks + embeddings em SQLite), oferece uma **API HTTP** de busca semântica e uma **interface web** mínima para conferir o índice manualmente.

Ideia geral: oferecer um jeito **seguro**, **auditável** e **reproduzível** de chamar LLMs sobre código e documentação, com um caminho de evolução do harness mínimo até o `claw` completo.

---

## 2. Público-alvo e cenários

| Segmento | Tarefa |
|---------|--------|
| Desenvolvedor | Trabalho diário com o código pelo `claw` completo: REPL, ferramentas, sessões. |
| Autor de automação | Prompts avulsos, pipelines com `--output-format json`, agentes embutidos sem bash. |
| Manutenção / auditoria | `claw-analog` em **read-only** + preset **audit**; limites e política explícitos. |
| Porte e parity | Comparação de comportamento com a referência (`PARITY.md`, mock-harness). |
| RAG sobre monorepo | `ingest` + `serve` separados; o agente puxa contexto via **`retrieve_context`** quando `RAG_BASE_URL` está definido. |

---

## 3. Arquitetura (lógica)

```text
                    ┌─────────────────────────────────────┐
                    │  Provedores (Anthropic / OpenAI / …) │
                    └─────────────────┬───────────────────┘
                                      │
       ┌──────────────────────────────┼──────────────────────────────┐
       │                              │                              │
       ▼                              ▼                              ▼
┌──────────────┐              ┌──────────────┐              ┌──────────────────┐
│  rusty-      │              │  claw-analog │              │ claw-rag-service │
│  claude-cli  │              │  (lean loop) │              │ HTTP + SQLite    │
│  («claw»)    │              │              │              │ ingest / query   │
└──────┬───────┘              └──────┬───────┘              └────────┬─────────┘
       │                              │                               │
       │          crates/api          │         retrieve_context      │
       │    runtime, tools, …         │         (POST /v1/query)      │
       └──────────────┬───────────────┴───────────────────────────────┘
                      │
                      ▼
               Sistema de arquivos / workspace (-w)
```

**Princípio de separação:** a indexação pesada e o armazenamento de embeddings **não** ficam embutidos no `claw-analog`; vivem no **`claw-rag-service`**. O agente só chama a recuperação (retrieval) por HTTP, o que facilita escalar, trocar o banco vetorial e as credenciais de embedding.

---

## 4. Princípios de projeto

1. **Seguro por padrão**: caminhos relativos, `..` proibido, verificação de saída do workspace canônico; os modos `PermissionMode` são os mesmos do CLI completo; em modo não interativo, os modos perigosos ficam bloqueados sem uma flag explícita.
2. **Limites explícitos**: tamanho de leitura, número de turnos, tetos de glob/grep, timeouts do RAG; as falhas são previsíveis, e não "estouro de memória ou laço infinito".
3. **Observabilidade para agentes**: NDJSON com `schema` e `format_version` no `run_start`, `tool_result` estruturados.
4. **Modularidade**: `api` comum a todos os provedores; o `claw-analog` não duplica a pilha de chaves do RAG, só tem um cliente HTTP para o serviço.
5. **Paridade e testes**: mock da Anthropic, cenários do harness, jobs de CI separados para os crates críticos.
6. **Documentação junto ao código**: `how_to_run.md`, `docs/rag-web-ui.md`, `docs/container.md` etc.

---

## 5. Componentes do workspace (resumo)

- **`rusty-claude-cli`**: o binário principal **`claw`**, o produto completo para o usuário.
- **`api`**: clientes dos provedores, streaming, tipos de requisição/resposta.
- **`runtime`**: sessões, configuração, **PermissionPolicy** / **PermissionEnforcer**, prompts, MCP etc.
- **`tools`**: ferramentas embutidas do CLI completo.
- **`claw-analog`**: laço mínimo — ferramentas de leitura/busca/escrita (conforme o modo), streaming e JSON, configuração TOML, sessões, doctor, config validate, **retrieve_context** quando há `RAG_BASE_URL` / `rag_base_url`.
- **`claw-rag-service`**: `ingest`, `serve`, rotas `/`, `/health`, `/v1/stats`, `/v1/query`; SQLite + embeddings compatíveis com OpenAI (ou mock para testes).
- **`mock-anthropic-service`**, **`compat-harness`** etc.: reprodutibilidade e migração.

Estrutura detalhada: [`rust/README.md`](rust/README.md).

---

## 6. Claw-analog: papel e limites

**Objetivo:** oferecer um "agente com ferramentas" sem aumentar a superfície de ataque (não há shell arbitrário no cenário básico).

**Ferramentas (conceitualmente):** leitura e navegação da árvore (`read_file`, `list_dir`, `glob_workspace`), busca literal (`grep_workspace` / `grep_search`), `write_file` opcional e **`retrieve_context`** opcional para o serviço de RAG.

**Fora do desenho mínimo:** MCP, plugins, bash — isso é território do **`claw` completo**.

---

## 7. Serviço de RAG: papel e evolução

**Agora (MVP):** reindexação completa a cada `ingest`, vetores em SQLite, busca linear por similaridade de cosseno sobre todos os chunks; serve para volumes moderados de código.

**Direções de crescimento (conceito):** indexação incremental, ANN (sqlite-vec, Qdrant/Chroma em Docker), limites de taxa para embeddings. A interface web em `GET /` é auxiliar; interface avançada e autenticação, conforme a necessidade.

Detalhes: [`docs/rag-web-ui.md`](docs/rag-web-ui.md).

---

## 8. O repositório fora do runtime principal

- **`src/`**, **`tests/`** (Python e outros): artefatos auxiliares/experimentais; o **runtime canônico** é **`rust/`**.
- Os documentos **PHILOSOPHY.md**, **ROADMAP.md** e **PARITY.md** complementam o conceito com processo e intenções da comunidade/mantenedores.

---

## 9. Conceitos relacionados (fora do núcleo do Claw Code)

Em **`docs/`** pode haver notas reaproveitáveis para **outros** produtos (por exemplo, um documento de "vision" local para aplicações NestJS [tradução incerta: pode ser visão de produto ou visão computacional]). Elas **não** definem comportamento obrigatório do `claw`, mas refletem interesses próximos dos contribuidores.

---

## 10. Formulação final

O **Claw Code** é um ecossistema **Rust** em torno do agente **`claw`**: CLI completo para desenvolvedores, **`claw-analog`** como agente mínimo controlado para automação e um **serviço de RAG separado** para busca semântica no código. O projeto se apoia em **permissões explícitas**, **limites**, **testabilidade** e **fronteiras HTTP claras** entre o agente e a indexação pesada.

---

*Atualize este arquivo quando mudarem decisões-chave de produto; o checklist detalhado de funcionalidades e o backlog estão em [`futute.md`](futute.md).*
