---
id: SPEC-COBAIA-0002
type: SPEC
title: "Histórico de movimentações de estoque e consultas de saldo calculadas a partir dele"
status: approved
project: "COBAIA"
owner: "Michel Pessoa"
created: "2026-11-23"
updated: "2026-11-23"
relates_to: []
parent_rfc: null
parent_adr: null
sizing: "medium"
approved_by:
  kind: "human"
  actor: "michelpessoa"
  at: "2026-11-23T21:23:49Z"
  content_sha256: "a9767c05663a208d1589540ddb10fd71321f6925e42a5959c60541932b0ea6f6"
  evidence: "https://github.com/michelpessoa/fabrica-central-teste/pull/11#pullrequestreview-6029752464"
supersedes: null
superseded_by: null
tags: [fabrica-descoberta]
---

# Histórico de movimentações de estoque e consultas de saldo calculadas a partir dele

## Objetivo

Registrar as entradas e saídas de estoque por item num histórico em memória (uma lista de dicts que quem chama cria e passa às funções) e fazer `saldo_total` (`estoque/totais.py`), `itens_zerados` (`estoque/consultas.py`) e `itens_com_saldo` (`estoque/disponiveis.py`) calcularem o saldo a partir desse histórico, e não mais do saldo final guardado em `Estoque`. A classe `Estoque` (`estoque/itens.py`) e os outros módulos não mudam.

# Parte 1 — Requisito (o QUÊ)

## Requisitos funcionais

| RF-ID | Requisito | Critério de aceite (EARS) | Arquivos |
|---|---|---|---|
| RF01 | `registrar_entrada(historico, item, quantidade, data)` acrescenta uma movimentação de entrada ao histórico | Quando `registrar_entrada` for chamada com `item` str não vazio, `quantidade` int > 0 (não bool) e `data`, o sistema deve acrescentar ao fim de `historico` o dict `{"tipo": "entrada", "item": item, "quantidade": quantidade, "data": data}` e devolver None; se `quantidade` não for int (incluindo bool e float), for <= 0, ou se `item` for vazio ou só espaços, o sistema deve levantar ValueError sem alterar `historico`. | estoque/movimentacoes.py, tests/test_movimentacoes.py |
| RF02 | `registrar_saida(historico, item, quantidade, data)` acrescenta uma movimentação de saída sem deixar saldo negativo | Quando `registrar_saida` for chamada com entrada válida e `quantidade` <= saldo atual do item (saldo de `saldo_por_item(historico)` sobre todas as movimentações), o sistema deve acrescentar `{"tipo": "saida", "item": item, "quantidade": quantidade, "data": data}` e devolver None (saída igual ao saldo deixa o saldo em 0); se a saída deixar o saldo negativo, ou valerem as mesmas regras de entrada inválida de RF01, o sistema deve levantar ValueError sem alterar `historico`. | estoque/movimentacoes.py, tests/test_movimentacoes.py |
| RF03 | `saldo_por_item(historico, ate=None)` devolve dict item -> saldo | Quando `saldo_por_item` for chamada, o sistema deve devolver um dict item -> soma das quantidades de entrada menos as de saída, considerando só movimentações com `data <= ate`, ou todas quando `ate` for None. | estoque/movimentacoes.py, tests/test_movimentacoes.py |
| RF04 | `saldo_total(historico, nomes)` soma os saldos calculados do histórico | Quando `saldo_total(historico, nomes)` for chamada, o sistema deve devolver a soma dos saldos de `saldo_por_item(historico)` para os nomes em `nomes`, contando 0 para nome sem movimentação. | estoque/totais.py, tests/test_totais.py |
| RF05 | `itens_zerados(historico, nomes)` lista os nomes com saldo calculado 0 | Quando `itens_zerados(historico, nomes)` for chamada, o sistema deve devolver a lista dos nomes de `nomes`, na ordem de `nomes`, cujo saldo em `saldo_por_item(historico)` é 0, contando 0 para nome sem movimentação. | estoque/consultas.py, tests/test_consultas.py |
| RF06 | `itens_com_saldo(historico, nomes)` lista os nomes com saldo calculado maior que 0 | Quando `itens_com_saldo(historico, nomes)` for chamada, o sistema deve devolver a lista dos nomes de `nomes`, na ordem de `nomes`, cujo saldo em `saldo_por_item(historico)` é maior que 0; nome sem movimentação não entra. | estoque/disponiveis.py, tests/test_disponiveis.py |

## Requisitos transversais (sweep)

| Categoria | Destino (RF-ID ou n/a) | Motivo (obrigatório se n/a) |
|---|---|---|
| Autorização / permissão | n/a | Biblioteca local sem usuários nem controle de acesso. |
| Concorrência | n/a | O histórico é uma lista criada e passada por quem chama; não há estado global nem singleton, e não foi pedido uso concorrente. |
| Idempotência | n/a | Cada chamada de registro é uma nova movimentação por definição; duas chamadas iguais registram duas movimentações. |
| Observabilidade | n/a | Funções puras de biblioteca; nenhum log ou métrica foi pedido. |
| Falha de dependência externa | n/a | Só biblioteca padrão, sem I/O, rede, arquivo ou banco. |
| Validação de entrada | RF01, RF02 | Quantidade não int (incluindo bool e float), quantidade <= 0, item vazio ou só espaços e saída acima do saldo levantam ValueError sem registrar. |
| Limite de volume / rate | n/a | Histórico em memória do processo; nenhum limite foi definido. |

## Fora de escopo

- Persistência do histórico (arquivo, banco), estado global ou singleton.
- Mudanças na classe `Estoque` (`estoque/itens.py`) e em outros módulos além dos listados.
- Validação do tipo de `data` e do tipo de `item` além de vazio/só espaços (não pedida).
- Uso do parâmetro `ate` nas três consultas: elas usam o saldo de todas as movimentações.
- Remoção, edição ou estorno de movimentações.

# Parte 2 — Desenho (o COMO e o ONDE)

## Contratos técnicos

- Python 3.10 ou mais novo, só biblioteca padrão; nenhuma dependência nova (sem pytest, pydantic etc.).
- Novo módulo `estoque/movimentacoes.py`:
  - `registrar_entrada(historico: list, item: str, quantidade: int, data: datetime.date) -> None`
  - `registrar_saida(historico: list, item: str, quantidade: int, data: datetime.date) -> None`
  - `saldo_por_item(historico: list, ate: datetime.date | None = None) -> dict[str, int]`
- Movimentação: dict com exatamente as chaves `tipo` (`"entrada"` ou `"saida"`), `item` (str), `quantidade` (int > 0) e `data` (`datetime.date`).
- Validação antes do append: se falhar, ValueError e o histórico fica intacto. `registrar_saida` calcula o saldo atual com `saldo_por_item(historico)` (sem `ate`).
- Novas assinaturas: `saldo_total(historico, nomes)`, `itens_zerados(historico, nomes)`, `itens_com_saldo(historico, nomes)`. Cada uma chama `saldo_por_item(historico)` e usa `.get(nome, 0)`. O parâmetro `nomes` e o formato do resultado não mudam: soma para `saldo_total`, lista na ordem de `nomes` para as outras duas.
- Nada é lido de estado global.

## Estratégia de teste

- `unittest` da biblioteca padrão; testes em `tests/`, um arquivo por módulo (`tests/test_<modulo>.py`), com classes `unittest.TestCase` e métodos `test_*`.
- `tests/test_movimentacoes.py`: append e formato do dict em entrada e saída; ValueError sem registrar para quantidade bool, float, 0, negativa, item vazio e só espaços; saída acima do saldo recusada; saída igual ao saldo aceita e saldo 0; `saldo_por_item` com e sem `ate`.
- `tests/test_totais.py`, `tests/test_consultas.py`, `tests/test_disponiveis.py`: passam a montar o histórico com `registrar_entrada` e `registrar_saida`, no lugar de `Estoque.adicionar`; cobrem nome sem movimentação (saldo 0) e ordem de `nomes`.
- Comandos, na raiz do repositório:
  - Build: `python3 -m compileall -q estoque tests`
  - Por módulo: `python3 -m unittest tests.test_movimentacoes -v` (idem `tests.test_totais`, `tests.test_consultas`, `tests.test_disponiveis`)
  - Suíte inteira (regressão): `python3 -m unittest discover -s tests -v`
