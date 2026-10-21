# 03 — Modelo local: Ollama + `claw-analog`

Objetivo: rodar o laço do agente com um modelo na própria GPU e ver, na prática, onde o
harness segura o modelo e onde ele falha.

Ambiente testado em 2026-10-21: Ollama 0.40.7, `qwen3:4b`, GTX 1660 Ti (6 GB), rustc 1.99.1.

## Ligar o `claw-analog` ao Ollama

```sh
cd ~/claw-code/rust
cargo build -p claw-analog
export OPENAI_BASE_URL=http://127.0.0.1:11434/v1   # endpoint compatível com OpenAI do Ollama
unset ANTHROPIC_API_KEY ANTHROPIC_BASE_URL
./target/debug/claw-analog --model qwen3:4b -w . "Responda apenas: OK"
```

Sem chave de API: em endereço local o claw aceita qualquer nome de modelo
(`docs/local-openai-compatible-providers.md:28`). Com `--output-format json`, cada evento
(chamada de ferramenta, resultado, resposta) sai como uma linha NDJSON.

## Experimentos realizados

| # | Pedido | Turnos | Tempo | Resultado |
|---|---|---|---|---|
| t1 | "Responda apenas: OK" | 1 | 7 s | Correto. ~1.200 tokens só de prompt de sistema + definições de ferramentas |
| t2 | Contar subdiretórios de `crates` com `list_dir` | 2 | 33 s | Correto (12) |
| t3 | Ler `claw-analog/Cargo.toml` e dar o `version` | 2 | 56 s | Correto: explicou que é herdado do workspace (`version.workspace = true`) |
| t4 | Ler `claw-analog/src/main.rs` e dar a linha de `async fn main` | 2 | 91 s | **Errado.** Respondeu outra coisa, em inglês |
| t5 | O mesmo que t4, com contexto de 12.288 tokens | 2 | 49 s | Correto: linha 388 (conferido em `main.rs:388`) |

### Por que o t4 falhou

O Ollama usa por padrão contexto de 4096 tokens nesta GPU (log do serviço:
`vram-based default context ... default_num_ctx=4096`). O arquivo lido tem ~5.300 tokens.
O Ollama corta o início da conversa para caber — e o início é justamente o prompt de sistema
e a pergunta. O modelo recebe só o código e "responde" a algo que ninguém perguntou.

O harness não percebeu: o `tool_result` saiu com `truncated: false`, porque o limite do
`claw-analog` (`--max-read-bytes`, 256 KiB) é muito maior que a janela do modelo. **O corte
aconteceu numa camada que o harness não enxerga.**

### Como o t5 corrigiu

Uma variante do modelo com janela maior, sem mexer no serviço:

```sh
printf 'FROM qwen3:4b\nPARAMETER num_ctx 12288\n' > Modelfile
ollama create qwen3-4b-12k -f Modelfile
./target/debug/claw-analog --model qwen3-4b-12k -w . "..."
```

Custo: o modelo passou de 3,2 GB para 4,5 GB e `ollama ps` mostrou `11%/89% CPU/GPU` —
parte já não coube na placa e foi para a CPU.

## Velocidade medida (`/api/generate`, sem modo de raciocínio)

| Situação | Leitura do prompt | Geração |
|---|---|---|
| Prompt curto (29 tokens) | 197 tok/s | 68,7 tok/s |
| Prompt longo (2.324 tokens) | 240 tok/s | 56,7 tok/s |

Leitura de ~240 tok/s explica os tempos acima: cada turno relê o histórico inteiro, então um
arquivo de 5 mil tokens custa ~20 s só para ser lido pelo modelo.

## Lições para um revisor documental

1. **A janela de contexto é um limite do harness, não só do modelo.** O limite de leitura da
   ferramenta deve ser menor que a janela, senão o corte acontece em silêncio.
2. **Ler o documento inteiro não escala.** Por isso o módulo 04 (RAG) busca só os trechos relevantes.
3. **Conferir a resposta contra a fonte** (como foi feito com `main.rs:388`) é o que separa
   acerto de alucinação plausível. Isso vira o método do módulo 06 (evals).

## Exercícios

1. Repita o t4 com `--max-read-bytes 8000`. O harness passa a marcar `truncated: true`? O modelo admite que não viu o arquivo todo?
2. Compare `qwen3:4b` e `qwen3-4b-12k` no t2: a janela maior deixa as respostas curtas mais lentas?
3. Rode `nvidia-smi` durante o t5 e anote memória e temperatura.
