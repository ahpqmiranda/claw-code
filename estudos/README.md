# Estudos — harness de agente, MCP, RAG e fine-tuning em pequena escala

Material de estudo pessoal sobre o fork `ahpqmiranda/claw-code` (base: v0.1.7).
Objetivo: entender, e não só delegar, como se constrói um agente útil e confiável,
com vistas a MCPs próprios e a revisores documentais como a Arty.

## Convenções deste material

- Toda afirmação sobre o código aponta para `arquivo:linha` conferido na branch `estudos`.
- **[verificar]** marca o que ainda não foi testado nesta máquina.
- **[conhecimento geral]** marca o que vem de conhecimento geral e não do repositório.
- Idioma: português (o repositório original mistura inglês e russo; ver `how_to_run.md`).

## Máquina de laboratório

| Item | Valor | Implicação |
|---|---|---|
| GPU | GTX 1660 Ti, 6 GB, driver 580 | Inferência local de modelos 4–8B em 4 bits; QLoRA de 0,5–1,5B; sem bf16 nem FlashAttention 2 **[conhecimento geral]** |
| RAM | 31 GB | Folga para indexação e para descarregar camadas para a CPU |
| Rust | rustc 1.99.1 (stable) | O 1.89 não compila o crate `runtime` (`Duration::from_hours` em `policy_engine.rs:7`); testes do `claw-analog` passam no 1.99.1 |

## Módulos

| # | Módulo | Estado |
|---|---|---|
| 00 | [Ambiente](00-ambiente.md) | escrito |
| 01 | [Fundamentos: harness, ferramentas, alucinação](01-fundamentos.md) | escrito |
| 02 | [Laço mínimo com servidor falso](02-laco-minimo.md) | escrito, exercícios **[verificar]** |
| 03 | [Modelo local: Ollama + `claw-analog`](03-modelo-local.md) | escrito, experimentos rodados em 2026-10-21 |
| 04 | RAG: `claw-rag-service` sobre documentos próprios | a escrever |
| 05 | MCP: como o `runtime` consome servidores e escrever o primeiro | a escrever |
| 06 | Evals: medir acertos com casos reais | a escrever |
| 07 | Fine-tuning QLoRA pequeno, medido pelas evals do 06 | a escrever |

### Objetivos dos módulos futuros

- **03** — Apontar o `claw-analog` para o Ollama (`OPENAI_BASE_URL`, ver
  `docs/local-openai-compatible-providers.md`) e observar onde um modelo pequeno erra
  chamadas de ferramenta, e como os limites do harness o seguram.
- **04** — Indexar um conjunto de documentos com `claw-rag-service` (`ingest` + `serve`) e
  ligar a ferramenta `retrieve_context` via `RAG_BASE_URL`. Ensaio do núcleo de uma revisora documental.
- **05** — Ler a parte de MCP em `rust/crates/runtime` e escrever um servidor MCP mínimo
  (ex.: expor contratos de teste do MGC, sem dados reais).
- **06** — Montar 20–50 casos com resposta esperada e um script que pontua o agente.
- **07** — Treinar um LoRA pequeno e comparar antes/depois com as evals do 06.
