# Handover — estudos de IA no claw-code (2026-10-21)

## Objetivo
Material de estudo pessoal (harness de agente, MCP, RAG, evals, fine-tuning pequeno) sobre o
fork `ahpqmiranda/claw-code`, rodando modelos locais na GTX 1660 Ti (6 GB).

## Estado
- Pronto: módulos 00–03 em `estudos/` (índice em `estudos/README.md`).
- Pronto: traduções pt-BR em arquivos paralelos — `how_to_run.pt-BR.md` (Codex, revisado),
  `concept.pt-BR.md`, `docs/rag-web-ui.pt-BR.md` (Claude, revisado pelo Codex).
- Pronto: comentários em chinês de `rusty-claude-cli/src/main.rs` traduzidos; palavras-chave em
  russo de `claw-analog/src/lib.rs` anotadas com `// pt-BR:` (strings intactas).
- Falta: exercícios do módulo 02 (servidor falso) ainda marcados [verificar]; módulos 04–07.

## Repositório
- Branch `estudos` (local, sem push). Base upstream: v0.1.7 (`22fe360`).

## Decisões
- Traduções em arquivos `*.pt-BR.md` ao lado do original: atualizações do upstream não conflitam.
- Strings em russo no `lib.rs` não foram traduzidas: são usadas pelo programa (`infer_preset_from_prompt`, `--lang ru`) e pelos testes.
- `Cargo.lock` não versionado nas mudanças: o build reescreve a versão 0.1.6→0.1.7; desfazer com `git checkout -- rust/Cargo.lock`.

## Ambiente
- Rust 1.99.1 (o 1.89 não compila `runtime`: `Duration::from_hours` em `policy_engine.rs:7`).
- Ollama 0.40.7 como serviço systemd; modelos `qwen3:4b` e `qwen3-4b-12k` (num_ctx 12288, criado via Modelfile).
- `claw-analog` com Ollama: `OPENAI_BASE_URL=http://127.0.0.1:11434/v1`, sem chave de API.
- Codex CLI 0.161.0 disponível (`codex exec -s read-only|workspace-write -C <dir> -o <arquivo>`).

## Próximo passo
Módulo 04 (RAG): `claw-rag-service ingest` + `serve` com embeddings locais do Ollama
(`ollama pull nomic-embed-text`; testar `CLAW_RAG_EMBEDDING_BASE_URL=http://127.0.0.1:11434/v1`
e `CLAW_RAG_EMBEDDING_MODEL`), depois `RAG_BASE_URL` no `claw-analog` para a ferramenta `retrieve_context`.
Verificação: `cd rust && cargo test -p claw-analog` (35 testes passando em 2026-10-21).

## Armadilhas
- Contexto padrão do Ollama nesta GPU é 4096: arquivo maior que isso é cortado em silêncio e o
  modelo alucina (módulo 03, t4). O harness não detecta.
- `num_ctx` 12288 já força ~11% do modelo para a CPU.
- No zsh, `echo ===` falha (expansão de `=`); usar `echo '---'`.
- Documentação original do `claw-analog` está em russo e usa caminhos de Windows/PowerShell.
