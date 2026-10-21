# 02 — Laço mínimo com servidor falso

Objetivo: ver o laço do agente funcionando sem modelo, sem GPU e sem custo de API.

## O servidor falso

`rust/crates/mock-anthropic-service` imita a API de Mensagens da Anthropic.

- Sobe com `--bind HOST:PORT` (padrão `127.0.0.1:0`, porta aleatória): `src/main.rs:7-29`.
- Imprime `MOCK_ANTHROPIC_BASE_URL=...` (`rust/MOCK_PARITY_HARNESS.md`, seção "Manual mock server").
- Não "pensa": escolhe a resposta por um marcador no texto do último pedido,
  `PARITY_SCENARIO:<nome>` (`src/lib.rs:13` e `src/lib.rs:272-282`).
- Cenários disponíveis: `src/lib.rs:120-133` — por exemplo `streaming_text`,
  `read_file_roundtrip`, `write_file_allowed`, `write_file_denied`, `multi_tool_turn_roundtrip`.

## Exercício 1 — resposta simples **[verificar]**

```sh
# terminal 1
cd ~/claw-code/rust
cargo run -p mock-anthropic-service -- --bind 127.0.0.1:8099

# terminal 2
cd ~/claw-code/rust
export ANTHROPIC_BASE_URL="http://127.0.0.1:8099"
export ANTHROPIC_API_KEY="qualquer-coisa"
cargo run -p claw-analog -- -w . "PARITY_SCENARIO:streaming_text"
```

Observe no stderr as linhas `[claw-analog] turn N stop_reason=...` (`claw-analog/src/lib.rs:1315`).

## Exercício 2 — uma volta com ferramenta **[verificar]**

Troque o cenário por `read_file_roundtrip`. Esperado: o primeiro turno termina com
`stop_reason=tool_use`, o harness executa `read_file`, e o segundo turno termina sem ferramentas.

## Exercício 3 — permissão negada **[verificar]**

Rode `write_file_allowed` com `--permission read-only` e depois com `--permission workspace-write`.
Compare as saídas: onde o harness recusou, o modelo (falso) chegou a pedir a escrita?

## O que anotar

- Quantos turnos cada cenário levou.
- O formato do `tool_result` que voltou ao "modelo" (use `--output-format json` para ver em NDJSON).
- Em que ponto do código a recusa de permissão aparece.
