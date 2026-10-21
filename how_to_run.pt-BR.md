> Tradução para pt-BR de how_to_run.md (commit 22fe360). Em caso de divergência, vale o original.
# claw-analog — como executar e como funciona

Agente mínimo sobre a mesma stack de API do CLI principal [`claw`](rust/README.md): os providers Anthropic / compatíveis com OpenAI / xAI são escolhidos conforme o modelo e as variáveis de ambiente (consulte [USAGE.md](USAGE.md)).

Nos exemplos a seguir, o **diretório de trabalho** é a pasta **`claw-code-main\rust`** (dentro do clone do repositório). Se o prompt do PowerShell já mostra `…\claw-code-main\rust>`, **não** execute `cd rust` novamente (isso levaria a `rust\rust` e causaria um erro de caminho).

## Requisitos

- **Rust** e **cargo** instalados (no PATH: geralmente `%USERPROFILE%\.cargo\bin` no Windows).
- Chave de API do provider escolhido (por exemplo, `ANTHROPIC_API_KEY`).

## Compilação e ajuda

```powershell
cd D:\path\to\claw-code-main\rust
cargo build -p claw-analog
cargo run -p claw-analog -- --help
```

### Diagnóstico (`doctor`)

O subcomando **`claw-analog doctor`** (com seu próprio `--help`, separado do modo principal):

- **prévia da configuração** — resultado da mesclagem de **`.claw-analog.toml`** (caminho `<workspace>/.claw-analog.toml` ou **`--config`**) com **as mesmas flags** do run principal: **`--model`**, **`--permission`**, **`--preset`**, **`--output-format`**, **`--stream`**, **`--no-stream`**, **`--no-runtime-enforcer`**, **`--accept-danger-non-interactive`**, além de **`--profile`** para mostrar o caminho do perfil. São exibidos o contrato NDJSON (`schema`, `format_version`), os campos efetivos e as linhas de **provenance** (qual fonte prevaleceu: CLI, TOML ou default);
- status das variáveis comuns (**sem** os valores: apenas `set` / `unset` e o comprimento da string);
- busca do workspace subindo a partir do cwd (ou de **`--manifest-dir`**) e, por padrão, **`cargo check -p claw-analog`** (apenas compilação, **não** sobrescreve `target\debug\claw-analog.exe` — no Windows, durante `cargo run … doctor`, um `cargo build` aninhado costuma gerar “Acesso negado”);
- **`--release-build`** — **`cargo build --release -p claw-analog`** (binário em `target\release\`, sem conflito com o executável de debug em uso);
- **`--no-build`** — pula o cargo;
- **`--tcp-ping`** (alias **`--mock`**) — TCP **`connect`** ao host e à porta de **`ANTHROPIC_BASE_URL`** (ou ao padrão `https://api.anthropic.com`); não verifica HTTP/TLS nem o corpo da resposta.

Exemplos (a partir do diretório `…\claw-code-main\rust`):

```powershell
cargo run -p claw-analog -- doctor
cargo run -p claw-analog -- doctor --no-build
cargo run -p claw-analog -- doctor --tcp-ping
cargo run -p claw-analog -- doctor -w D:\path\to\repo --preset implement
cargo run -p claw-analog -- doctor --release-build
```

### Validação da configuração sem API (`config validate`)

O subcomando **`claw-analog config validate`**:

- faz o parse de **`.claw-analog.toml`** (por padrão `<workspace>/.claw-analog.toml`, com substituição por **`--config`**) e exibe uma breve **merge preview** (como em `doctor`, mas **apenas TOML + defaults**, sem as flags do run principal);
- verifica **`profile.toml`**: a mesma ordem do run (`--profile`, campo `profile` no TOML ou, caso contrário, o padrão `~/.claw-analog/profile.toml`, se o arquivo existir);
- **nenhuma** requisição ao LLM ou à rede da API.

**`--strict`** — erro (código de saída 1) se o arquivo de configuração não existir ou o perfil não puder ser lido.

```powershell
cargo run -p claw-analog -- config validate -w D:\path\to\repo
cargo run -p claw-analog -- config validate --strict -w .
```

### Autocompletar do shell (`complete`)

Script de autocompletar em **stdout** (redirecione para um arquivo conforme a documentação do seu shell):

```powershell
cargo run -p claw-analog -- complete powershell >> $PROFILE
# bash:zsh:fish — veja a saída de `complete --help`
```

Valores disponíveis: **`bash`**, **`zsh`**, **`fish`**, **`powershell`** (alias **`pwsh`**).

## Comandos principais

Uma tarefa como argumento (ou texto via **stdin**):

```powershell
# a partir de ...\claw-code-main\rust
cargo run -p claw-analog -- -w D:\path\to\repo "Descreva brevemente a estrutura de rust/crates"
```

Com **saída em tempo real** (SSE por `stream_message`):

```powershell
cargo run -p claw-analog -- --stream -w . "Explique o claw-analog em duas frases"
```

Permitir **gravação de arquivos** no workspace:

```powershell
cargo run -p claw-analog -- --permission workspace-write -w . "Adicione um comentário no início de crates/claw-analog/Cargo.toml"
```

Desativar a verificação por **`runtime::PermissionEnforcer`** (resta apenas o isolamento de caminhos próprio; não recomendado):

```powershell
cargo run -p claw-analog -- --no-runtime-enforcer -w . "…"
```

Limites úteis (o CLI **sobrescreve** os valores de `.claw-analog.toml`; veja abaixo):

| Flag | Valor padrão | Finalidade |
|------|------------------------|------------|
| `--max-read-bytes` | 262144 | Máximo de bytes para `read_file` / `grep_workspace` / `git_diff` / `git_log` |
| `--max-turns` | 24 | Máximo de rodadas “modelo → ferramentas → modelo” |
| `--max-list-entries` | 500 | Limite de linhas de `list_dir` |
| `--grep-max-lines` | 200 | Limite superior do **total** de linhas com correspondências em `grep_workspace` (inclusive entre vários arquivos; em um arquivo, é possível definir um valor menor com `max_lines`) |
| `--glob-max-paths` | 2000 | Máximo de caminhos retornados por `glob_workspace` e pela expansão de `glob` dentro de `grep_workspace` |
| `--glob-max-depth` | 32 | Profundidade de varredura dos diretórios para glob (via `walkdir`), sem recursão infinita |
| `--output-format` | `rich` | `json` — NDJSON em stdout para scripts e agentes |
| `--print-tools` | — | Lista as ferramentas efetivas para o `permission` / enforcer resultante e encerra (**sem** prompt ou API) |
| `--lang` | `en` | Indicação no system: `en` ou `ru` (idioma das respostas; **não** altera o id do modelo na API) |
| `--preset` | — | `none` \| `audit` \| `explain` \| `implement` — veja a seção abaixo |
| `--session` | — | Caminho da sessão JSON (relativo a `-w`, se não for absoluto): salva o histórico e permite retomar |
| `--save-session` | — | Caminho adicional: o mesmo snapshot da sessão é gravado ali a cada salvamento (pode ser usado **sem** `--session`, apenas para exportar o JSON após a execução) |
| `--profile` | — | TOML com o campo `line` (incluído no system). Sem a flag: tenta `%USERPROFILE%\.claw-analog\profile.toml` (Windows) / `~/.claw-analog/profile.toml` |
| `--permission` | `read-only` | Veja abaixo: `read-only`, `workspace-write`, `prompt`, `danger-full-access`, `allow` |
| `--accept-danger-non-interactive` | — | Permite `danger-full-access` / `allow` quando stdin **não** é TTY (CI; risco assumido). No TOML: `accept_danger_non_interactive = true` |

Por padrão, a configuração é lida de **`<workspace>/.claw-analog.toml`**, se o arquivo existir. Outro caminho: **`--config PATH`**. Chaves TOML desconhecidas causam erro de parse (esquema estrito).

Exemplo de `.claw-analog.toml`:

```toml
model = "sonnet"
stream = true
output_format = "rich"
permission = "read-only"
language = "en"
preset = "audit"
session = ".claw-analog.session.json"
profile = "~/.claw-analog/profile.toml"
no_runtime_enforcer = false
accept_danger_non_interactive = false
max_read_bytes = 262144
max_turns = 24
max_list_entries = 500
grep_max_lines = 200
glob_max_paths = 2000
glob_max_depth = 32
# Opcional: RAG (`claw-rag-service`) — veja a seção sobre RAG abaixo
# rag_base_url = "http://127.0.0.1:8787"
# rag_timeout_secs = 30
# rag_top_k_max = 32
```

**RAG (`retrieve_context`):** se **`RAG_BASE_URL`** (via variável de ambiente) ou um **`rag_base_url`** não vazio estiver definido em `.claw-analog.toml`, **`retrieve_context`** é adicionado ao conjunto de ferramentas (busca semântica no workspace já indexado). O valor é a raiz HTTP do serviço, sem o sufixo `/v1` (a requisição vai para `{base}/v1/query`). O timeout e o limite superior de **`top_k`** são definidos por **`rag_timeout_secs`** e **`rag_top_k_max`** (por padrão, 30 s e 32; teto absoluto de 256). A indexação continua sendo feita pelo comando separado **`claw-rag-service`**; consulte [`docs/rag-web-ui.md`](docs/rag-web-ui.md).

**`permission`** (como no `claw` completo, com os mesmos valores no TOML):

| Valor | Ferramenta `write_file` | Modo não interativo (stdin não é TTY) |
|----------|-------------------------|------------------------------|
| `read-only` | não | OK |
| `workspace-write` | sim (dentro de `-w`) | OK |
| `prompt` | não (neste harness, o Enforcer impede a gravação sem confirmação) | aviso em stderr; para gravação automática, use `workspace-write` |
| `danger-full-access`, `allow` | sim | **proibido** até que `--accept-danger-non-interactive` seja passado ou `accept_danger_non_interactive = true` seja definido no TOML |

**`--stream`** na linha de comando ativa o streaming; **`--no-stream`** o desativa explicitamente (útil para substituir `stream = true` no arquivo).

**`language`** no TOML: `en` ou `ru` (os mesmos valores de **`--lang`**); o CLI tem prioridade.

### Sessão (`--session`)

Arquivo JSON (versão `1`): metadados `workspace`, `model`, `preset` opcional e array `messages` no formato da API (`role` + `content`). Quando a execução começa com um arquivo existente, o histórico **é carregado**, e o texto da solicitação atual (argumento ou stdin) é adicionado como uma **nova** mensagem do usuário. O estado é salvo após cada rodada completa com ferramentas e ao terminar sem `tool_use`.

**`--save-session`** — mesmo formato de arquivo que `--session`: a gravação é duplicada em cada etapa em que o arquivo da sessão seria atualizado (se o caminho for igual ao de `--session`, não há segunda gravação). Sem **`--session`**, é possível reunir o histórico de uma execução em JSON para scripts ou para um **`--session`** posterior, sem montar `messages` manualmente.

**Riscos:** o arquivo pode conter **segredos** (saída de `read_file`, chaves em logs) e não é criptografado; um histórico longo **custa mais** tokens da API. Um lembrete é exibido em stderr com **`--session`** ou **`--save-session`**. Se `workspace` / `model` / `preset` não corresponderem à execução atual, é exibido um **aviso**, mas a execução continua.

### Presets (`--preset`)

Adicionam um parágrafo curto ao system prompt (auditoria / aprendizado / alterações). O conjunto de ferramentas continua determinado por **permission**: para **`implement`**, se nem o CLI nem o arquivo definirem `permission`, o padrão passa a ser **workspace-write** (para disponibilizar `write_file`). Um `permission = "read-only"` explícito no arquivo ou `--permission read-only` no CLI tem prioridade.

### Perfil (`profile.toml`)

Arquivo mínimo:

```toml
line = "Dica curta de estilo (uma linha no system)."
```

Limites: arquivo de no máximo **2048** bytes; comprimento da linha após trim de no máximo **512** caracteres Unicode (caso contrário, ela é truncada com um aviso). O conteúdo é adicionado ao system em uma única linha: `Learner hint: …`.

## Ferramentas (sem shell arbitrário)

| Nome | Modo | Descrição |
|-----|--------|----------|
| `read_file` | read-only+ | Leitura de arquivo UTF‑8 dentro de `-w` |
| `list_dir` | read-only+ | Listagem de diretório (sem recursão) |
| `glob_workspace` | read-only+ | Lista de **caminhos de arquivos** dentro de `-w`: argumentos `pattern` (glob relativo a `root`, com barras `/`), `root` opcional (padrão `.`), `max_paths` (limitado pelo CLI). O padrão não pode conter `..`. |
| `grep_workspace` | read-only+ | Busca pela mesma substring **literal**, linha por linha, de antes; exatamente um seletor: `path`, array `paths` ou `glob` (+ `glob_root` opcional). O limite total de linhas é definido por `max_lines` e `--grep-max-lines`. Em vários arquivos, o formato das linhas é: `caminho/relativo:numero_da_linha:conteudo` (no original, os nomes desses campos estão em russo). |
| `grep_search` | read-only+ | Mesmo handler de `grep_workspace` (compatibilidade de prompts com o `claw` completo). |
| `git_diff` | read-only+ | `git diff` (sem cores) dentro do repositório em `-w`. Opcionais: `cached` (staged), `rev_range`, `context_lines`, `paths`. Saída limitada por `--max-read-bytes`. |
| `git_log` | read-only+ | `git log` (sem cores) dentro do repositório em `-w`. Opcionais: `max_count` (padrão 20), `rev_range`, `paths`. Saída limitada por `--max-read-bytes`. |
| `retrieve_context` | read-only+ | Apenas se **`RAG_BASE_URL`** ou **`rag_base_url`** estiver definido no TOML: HTTP **`POST {base}/v1/query`** para `claw-rag-service`; a resposta contém caminhos e trechos dos chunks (veja os limites acima). |
| `write_file` | `workspace-write`, `danger-full-access` ou `allow` | Gravação de arquivo; os diretórios pais são criados quando necessário (`prompt` não permite gravar por meio do Enforcer) |

## Como funciona

1. A **raiz do workspace** (`-w`) é convertida para um caminho canônico; todos os caminhos nas ferramentas são **relativos**, sem `..` nem segmentos absolutos.
2. Antes de acessar um arquivo, verifica-se se o caminho real permanece **dentro** da raiz (symlink/`canonicalize`).
3. **Política de permissões** (se `--no-runtime-enforcer` não estiver ativo): as mesmas entidades do CLI principal — `PermissionPolicy` + `PermissionEnforcer::check` para a ferramenta e `check_file_write` para gravação.
4. **Loop do agente**: requisição ao provider → se `stop_reason == tool_use`, as chamadas são executadas, os resultados entram no histórico como `tool_result` → próxima rodada.
5. **Streaming**: com `--stream`, o texto do assistente é exibido conforme os deltas chegam; o histórico para a próxima rodada é montado a partir do SSE, como no pipeline completo (índices de blocos + entrada JSON das ferramentas). Para desativar o streaming configurado no arquivo, use **`--no-stream`**.

Logs no formato `[claw-analog] ...` são gravados em **stderr**. No modo **rich**, a resposta do modelo é texto comum em **stdout**; no modo **json**, **stdout** contém apenas **NDJSON** (veja abaixo).

## Saída JSON (CI e agentes externos)

A flag **`--output-format json`** muda stdout para um **fluxo de linhas JSON** (um objeto por linha). O significado dos campos é estável, mas outros campos podem ser adicionados.

Principais valores de `type`:

| `type` | Quando |
|--------|--------|
| `run_start` | Início da execução: **`schema`** (`claw-analog-ndjson`), **`format_version`**, depois `workspace`, `model`, `stream`, `permission`, `preset` opcional, `session`, `session_save` opcional e **`rag_enabled`** booleano (se há uma URL base para `retrieve_context`) |
| `turn_start` | Início de uma rodada com o modelo (`turn`) |
| `assistant_text_delta` | Apenas com `--stream`: trecho do texto do assistente |
| `assistant_turn` | Resultado da rodada: `stop_reason`, `usage`, `text` completo, array `tool_calls` |
| `tool_result` | Após executar uma ferramenta: `name`, `tool_use_id`, `is_error`, `output` (pode ser truncado), `truncated`, `output_len_chars` |
| `run_end` | Conclusão bem-sucedida (`ok: true`) |
| `error` | Erro (exibido em uma linha separada em caso de falha ou prompt vazio) |

Exemplo (PowerShell): **`jq`** ou qualquer parser JSON facilita o processamento linha por linha do fluxo.

```powershell
# a partir de ...\claw-code-main\rust
$env:ANTHROPIC_API_KEY = "sk-ant-..."
cargo run -p claw-analog -- --output-format json -w . "Summarize rust/README.md" 2>$null | ForEach-Object { $_ | ConvertFrom-Json | Select-Object -ExpandProperty type }
```

Com **`--stream`**, stdout recebe primeiro os eventos `assistant_text_delta` e depois, para a mesma rodada, uma linha `assistant_turn` com o `text` completo montado (útil para logs reproduzíveis).

### Limites e riscos para agentes

- Em **`tool_result.output`**, arquivos grandes são truncados (~32 KiB UTF‑8), com o campo **`truncated`: true**.
- **Segredos**: não redirecione stderr bruto para logs públicos sem filtragem; `output` pode, em tese, conter o conteúdo de arquivos lidos.
- Contrato para orquestradores: NDJSON em stdout, diagnóstico em stderr; código de saída ≠ 0 em caso de erro. Na primeira linha, **`run_start`**, convém conferir **`schema`** e **`format_version`**; **`run_start`** também revela o caminho do workspace e o modelo — leve isso em conta ao compartilhar logs.

## Testes automatizados sem rede real

Testes unitários e integração com o **mock-anthropic-service** local:

```powershell
# a partir de ...\claw-code-main\rust
cargo test -p claw-analog
```

No **GitHub Actions**, um job separado **`claw-analog (test + clippy -p)`** executa `cargo test -p claw-analog` e `cargo clippy -p claw-analog --no-deps` (além de `cargo test` / `clippy` completos no workspace).

Quando os testes rodam em paralelo, as variáveis de ambiente da Anthropic são isoladas por **mutex** apenas no cenário mock; em caso de falha, é possível executar `cargo test -p claw-analog -- --test-threads=1`.

## Separadamente: `claw-rag-service` (RAG)

A indexação do workspace e a API HTTP ficam em **`cargo run -p claw-rag-service`** (`ingest` + `serve`). Após `serve`, abra **`http://127.0.0.1:8787/`** — uma interface leve (estatísticas + busca). A conexão com `claw-analog` usa **`RAG_BASE_URL`** / `retrieve_context`. Detalhes e variáveis de ambiente: [`docs/rag-web-ui.md`](docs/rag-web-ui.md).

### Ingest (um ou mais repositórios)

`ingest` aceita **múltiplas ocorrências** de `--workspace` — isso permite fazer **cross-repo RAG** (vários repositórios em um único banco de dados/coleção).

```powershell
# a partir de ...\claw-code-main\rust

# um workspace
cargo run -p claw-rag-service -- ingest --workspace "D:\v\kria\s6"

# vários workspaces (cross-repo)
cargo run -p claw-rag-service -- ingest --workspace "D:\repo1" --workspace "D:\repo2"
```

Nas respostas, `path` tem o formato `repoId:relative/path` (para evitar colisões entre caminhos iguais de repositórios diferentes).

### Mock embeddings (sem chaves / sem rede)

Para execuções e testes locais, é possível ativar mock embeddings:

```powershell
$env:CLAW_RAG_MOCK_PROVIDERS = "1"
cargo run -p claw-rag-service -- ingest --workspace "D:\v\kria\s6"
```

### Qdrant (opção local recomendada) via Docker

Para repositórios grandes, é melhor executar o Qdrant localmente: isso elimina o custo da varredura linear do `SQLite` e acelera as consultas.

Iniciar o Qdrant (gRPC na porta 6334):

```powershell
docker run --rm -p 6333:6333 -p 6334:6334 -e QDRANT__SERVICE__GRPC_PORT=6334 qdrant/qdrant
```

#### Qdrant com volume persistente (para preservar o índice)

Opção com um volume Docker nomeado:

```powershell
docker volume create claw-qdrant-data
docker run --rm -p 6333:6333 -p 6334:6334 `
  -e QDRANT__SERVICE__GRPC_PORT=6334 `
  -v claw-qdrant-data:/qdrant/storage `
  qdrant/qdrant
```

Opção com bind mount (caminho no host):

```powershell
mkdir .claw-qdrant | Out-Null
docker run --rm -p 6333:6333 -p 6334:6334 `
  -e QDRANT__SERVICE__GRPC_PORT=6334 `
  -v "${PWD}/.claw-qdrant:/qdrant/storage" `
  qdrant/qdrant
```

Em seguida, defina as variáveis de ambiente e execute o ingest com a feature `qdrant-index`:

```powershell
$env:CLAW_RAG_QDRANT_URL = "http://127.0.0.1:6334"
$env:CLAW_RAG_QDRANT_COLLECTION = "claw_rag_chunks"

# (opcional) sem API real de embeddings
$env:CLAW_RAG_MOCK_PROVIDERS = "1"

cargo run -p claw-rag-service --features qdrant-index -- ingest --workspace "D:\v\kria\s6"
```

`ingest` cria a coleção automaticamente se ela ainda não existir (de acordo com a dimensão dos embeddings).

### Execução via Docker (Qdrant + claw-rag-service)

Se quiser iniciar tudo com um único comando, use `docker compose`.

1) Iniciar os serviços:

```powershell
cd D:\path\to\claw-code-main
docker compose up --build
```

Observação: a imagem `rag-serve`/`rag-ingest` é compilada com uma versão suficientemente recente do Rust (veja `rust/crates/claw-rag-service/Dockerfile`), pois `qdrant-client` pode exigir uma versão mais nova do Rust do que tags antigas fixadas.

Se a compilação do Docker falhar e você vir linhas como `transferring context: 21.02GB`, confira se:

- você está executando o compose a partir da raiz do repositório (onde está `docker-compose.yml`)
- `.dockerignore` está sendo usado (reduz o build context, especialmente quando há `target/` e índices locais)

Se a compilação falhar logo com `EOF` na etapa `load local bake definitions`, tente:

```powershell
$env:COMPOSE_BAKE = "0"
$env:DOCKER_BUILDKIT = "0"
docker compose up --build
```

2) Ingest (execute separadamente, pois é um batch job). Exemplo para um workspace:

```powershell
docker compose run --rm rag-ingest ingest --workspace "/workspaces/main"
```

Por padrão, `rag-ingest` grava o índice em um volume compartilhado, de modo que `rag-serve` pode ver os chunks imediatamente.

### Conexão com `claw-analog`

```powershell
$env:RAG_BASE_URL = "http://127.0.0.1:8787"
cargo run -p claw-analog -- -w "D:\v\kria\s6" "Encontre onde o ingest está implementado no serviço de RAG"
```

## Auto‑TDD (verificações automáticas após `write_file`/`edit_file`)

No `claw` completo (e em outros consumidores de `runtime`), é possível ativar a execução automática do linter e dos testes após ferramentas de gravação bem-sucedidas por meio de `.claw/settings.json`:

```json
{
  "autoTdd": {
    "enabled": true,
    "tools": ["write_file", "edit_file"],
    "commands": [
      "cd rust && cargo fmt",
      "cd rust && cargo clippy --workspace --all-targets -- -D warnings",
      "cd rust && cargo test --workspace"
    ]
  }
}
```

## Diferenças em relação ao `claw` completo

- Conjunto restrito de ferramentas (sem bash/MCP/plugins).
- Mais simples de auditar e limitar com `--permission` e os limites.
- O produto principal continua sendo `cargo run -p rusty-claude-cli` → binário `claw`.

## Próximos passos de desenvolvimento

Plano e checklist de ideias (incluindo as inspiradas na camada de produto, como DeepTutor): [`futute.md`](futute.md) na raiz do repositório.
