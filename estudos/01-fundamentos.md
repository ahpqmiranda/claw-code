# 01 — Fundamentos: harness, ferramentas e alucinação

## Três coisas diferentes

| Conceito | O que é | Muda os pesos do modelo? |
|---|---|---|
| Harness | Programa em volta do modelo: monta contexto, oferece ferramentas, executa as chamadas, aplica permissões e limites, guarda a sessão | Não |
| MCP | Protocolo para um harness descobrir e chamar ferramentas/dados de servidores externos | Não |
| Fine-tuning | Treino adicional dos pesos com exemplos próprios | Sim |

**[conhecimento geral]** O harness reduz alucinação porque troca "lembrar" por "consultar":
o modelo lê o arquivo, busca o trecho, recebe o resultado real e só então responde.
Fine-tuning ajusta estilo e formato; não dá ao modelo fatos que não estão no contexto.

## O laço do agente, no código

O `claw-analog` é o laço mais curto do repositório. A descrição oficial está em
`how_to_run.md` (seção "Принципы работы", item 4): pedido ao provedor → se
`stop_reason == tool_use`, executa as chamadas, devolve os resultados como `tool_result`
→ próximo turno.

Onde isso acontece:

| Passo | Local |
|---|---|
| Entrada do programa | `rust/crates/claw-analog/src/main.rs:388` |
| Função que roda o agente | `rust/crates/claw-analog/src/lib.rs:1220` (`run`) |
| Laço de turnos, limitado por `max_turns` | `lib.rs:1285` |
| Condição de parada: sem chamadas ou `stop_reason` diferente de `tool_use` | `lib.rs:1349` |
| Resultado da ferramenta devolvido ao modelo (`tool_result`) | `lib.rs:1397` |
| Extração das chamadas da resposta | `lib.rs:1580` (`collect_tool_uses`) |

## Limites que o harness impõe (por que ele é "seguro")

Fonte: `how_to_run.md`, seções de ferramentas e princípios.

- Não há shell livre: só `read_file`, `list_dir`, `glob_workspace`, `grep_workspace`,
  `git_diff`, `git_log`, `retrieve_context` (com RAG) e `write_file` (só com permissão de escrita).
- Todo caminho é relativo ao workspace (`-w`), sem `..`; o caminho real é conferido após resolver symlinks.
- Permissão padrão `read-only`; `danger-full-access` exige confirmação explícita fora de terminal interativo.
- Limites numéricos: `max_turns` (24 no exemplo de TOML), bytes por leitura, linhas de grep, caminhos de glob.

## Perguntas para conferir o entendimento

1. Por que o laço precisa de `max_turns`? O que aconteceria com um modelo que sempre pede outra ferramenta?
2. Qual a diferença entre o modelo *pedir* `write_file` e o harness *executar* `write_file`? Onde a permissão é checada?
3. Num revisor documental, que ferramenta substituiria `read_file`, e que limite você colocaria nela?
