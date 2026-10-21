# 00 — Ambiente

## Rust

Use o Rust stable recente (`rustup update stable`; testado com 1.99.1). O 1.89 falha ao compilar o crate `runtime`. O workspace fica em `rust/` (ver `rust/AGENTS.md`).

```sh
cd ~/claw-code/rust
cargo build -p claw-analog -p mock-anthropic-service   # [verificar] primeira compilação
cargo run -p claw-analog -- --help
cargo run -p claw-analog -- doctor --no-build          # diagnóstico sem rede
```

## Ollama (instalação feita pelo Alan)

```sh
curl -fsSL https://ollama.com/install.sh | sh
ollama --version
nvidia-smi        # com um modelo carregado, o processo do ollama deve aparecer aqui
```

Modelos candidatos para 6 GB **[conhecimento geral, conferir o consumo real com `nvidia-smi`]**:

| Uso | Modelo | Observação |
|---|---|---|
| Agente com ferramentas | `qwen3:4b` | cabe com folga, sobra memória para contexto |
| Agente com ferramentas | `qwen3:8b` | perto do limite; contexto curto |
| Embeddings (módulo 04) | `nomic-embed-text` | pequeno |

## Teste de fumaça do provedor local

Segundo `docs/local-openai-compatible-providers.md:16-30`: `OPENAI_BASE_URL` aponta para o
endpoint `/v1`; em endereço local, qualquer nome de modelo não vazio é aceito; prompts simples
podem funcionar mesmo quando chamadas de ferramenta falham.

```sh
export OPENAI_BASE_URL="http://127.0.0.1:11434/v1"
curl -s "$OPENAI_BASE_URL/models"     # lista os modelos que o servidor expõe
```

O uso com o `claw-analog` fica para o módulo 03.
