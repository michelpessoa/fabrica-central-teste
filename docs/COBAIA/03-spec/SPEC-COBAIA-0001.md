---
id: SPEC-COBAIA-0001
type: SPEC
title: "Mínimos de estoque com reposição sugerida e persistência em JSON"
status: in_review
project: "COBAIA"
owner: "Michel Pessoa"
created: "2026-11-21"
updated: "2026-11-21"
relates_to: []
parent_rfc: null
parent_adr: null
sizing: "medium"
supersedes: null
superseded_by: null
tags: [fabrica-descoberta]
---

# Mínimos de estoque com reposição sugerida e persistência em JSON

## Objetivo

Permitir que o código do pacote `estoque/` configure um mínimo e uma margem por item. Com isso, ele pode listar os itens abaixo do mínimo, ordenados pela falta, sugerir quanto repor de cada um (até o mínimo mais a margem) e salvar e carregar essa configuração num arquivo JSON. É uma biblioteca em Python 3 que usa só a biblioteca padrão. Não tem CLI nem banco. Segue o padrão de `Limites` (`estoque/limites.py`) e de `itens_zerados(estoque, nomes)` (`estoque/consultas.py`). O saldo vem da classe `Estoque` de `estoque/itens.py`: `estoque.saldo(nome)` devolve um int e devolve 0 para um item que nunca foi adicionado.

# Parte 1 — Requisito (o QUÊ)

## Requisitos funcionais

| RF-ID | Requisito | Critério de aceite (EARS) | Arquivos |
|---|---|---|---|
| RF01 | Classe `Minimos` com `definir_minimo(nome, minimo)`, `minimo(nome)`, `definir_margem(nome, margem)` e `margem(nome)`. Os métodos de leitura devolvem o valor ou None. | Quando `definir_minimo` receber minimo <= 0, o sistema deve lançar `ValueError("minimo deve ser positivo")`. Quando `definir_margem` receber margem < 0, o sistema deve lançar `ValueError("margem não pode ser negativa")`. Se o item não tiver o valor definido, `minimo(nome)` ou `margem(nome)` deve devolver None. | estoque/minimos.py, tests/test_minimos.py |
| RF02 | `itens_abaixo_do_minimo(estoque, minimos, nomes)` em `estoque/consultas.py` | Quando chamada, a função deve devolver uma lista de tuplas `(nome, falta)`, com falta = minimo - saldo, só para os itens de `nomes` que têm mínimo e saldo < minimo. A ordem vai da maior falta para a menor, com empate resolvido pelo nome em ordem alfabética crescente. Itens sem mínimo ficam de fora, sem erro. Sem nenhum item abaixo do mínimo, a função deve devolver `[]`. | estoque/consultas.py, tests/test_consultas.py |
| RF03 | `sugerir_reposicao(estoque, minimos, nomes)` em `estoque/reposicao_sugerida.py` | Quando chamada, a função deve devolver um dicionário nome -> minimo + margem - saldo (margem 0 quando não definida), só para os itens de `nomes` que têm mínimo e saldo < minimo. Itens sem mínimo ou com saldo >= minimo ficam de fora. Sem nenhum item abaixo do mínimo, a função deve devolver `{}`. | estoque/reposicao_sugerida.py, tests/test_reposicao_sugerida.py |
| RF04 | `salvar_minimos(minimos, caminho)` e `carregar_minimos(caminho)` em `estoque/persistencia_minimos.py` | Quando `salvar_minimos` for chamada, o sistema deve gravar em UTF-8 o JSON `{"nome": {"minimo": N, "margem": M}}`, com as chaves em ordem alfabética, só com os itens que têm mínimo e com `margem` omitida quando não definida. Quando `carregar_minimos` for chamada, o sistema deve devolver um `Minimos` novo, montado por `definir_minimo` e `definir_margem`. Se uma entrada não tiver `minimo`, deve lançar `ValueError("minimo ausente para <nome>")`. | estoque/persistencia_minimos.py, tests/test_persistencia_minimos.py |

## Requisitos transversais (sweep)

| Categoria | Destino (RF-ID ou n/a) | Motivo (obrigatório se n/a) |
|---|---|---|
| Autorização / permissão | n/a | É uma biblioteca em memória, sem usuários nem acesso externo. A biblioteca não trata dado pessoal nem dinheiro. |
| Concorrência | n/a | Uso síncrono em memória e no mesmo processo, como `Estoque` e `Limites`. Nenhum requisito pede acesso concorrente. |
| Idempotência | RF04 | Salvar o mesmo `Minimos` duas vezes gera o mesmo arquivo, porque as chaves vão em ordem alfabética. Carregar o arquivo devolve um `Minimos` equivalente. |
| Observabilidade | n/a | Biblioteca pura, sem logging nem métricas no padrão do pacote. Os erros chegam como exceções. |
| Falha de dependência externa | RF04 | A única dependência externa é o sistema de arquivos. `FileNotFoundError` e `json.JSONDecodeError` passam sem conversão. |
| Validação de entrada | RF01 | Mínimo <= 0 e margem < 0 lançam ValueError. O tipo não é checado, como em `Limites.definir_limite`. RF04 reaproveita essa validação ao carregar e exige a chave `minimo`. |
| Limite de volume / rate | n/a | Nenhum limite foi pedido. O volume é o da lista `nomes` passada pelo chamador. |

## Fora de escopo

- CLI, banco de dados ou qualquer interface fora da biblioteca.
- Margem em percentual. A margem é sempre um número inteiro de unidades.
- Checagem de tipo dos valores. Um `TypeError` que surja na comparação passa sem tratamento, e não há teste para tipos que não sejam int.
- Converter `FileNotFoundError` ou `json.JSONDecodeError` em outro erro.
- Mudanças em `Estoque`, `Limites` ou `itens_zerados`.

# Parte 2 — Desenho (o COMO e o ONDE)

## Contratos técnicos

- `estoque/minimos.py`: `class Minimos`. Guarda internamente `_minimos: dict[str, int]` e `_margens: dict[str, int]`.
  - `definir_minimo(nome, minimo)`: se `minimo <= 0`, lança `ValueError("minimo deve ser positivo")`; senão, grava o valor.
  - `minimo(nome)`: devolve o valor ou None.
  - `definir_margem(nome, margem)`: se `margem < 0`, lança `ValueError("margem não pode ser negativa")`; senão, grava o valor (0 é aceito). Também aceita um item que ainda não tem mínimo.
  - `margem(nome)`: devolve o valor ou None.
- `estoque/consultas.py`: acrescentar `itens_abaixo_do_minimo(estoque, minimos, nomes) -> list[tuple[str, int]]`, sem alterar `itens_zerados`. Para cada nome de `nomes`, usa `m = minimos.minimo(nome)`; se `m is None`, ignora o item. Usa `s = estoque.saldo(nome)` e inclui `(nome, m - s)` se `s < m`. Ordena com a chave `(-falta, nome)`.
- `estoque/reposicao_sugerida.py`: `sugerir_reposicao(estoque, minimos, nomes) -> dict[str, int]`, com o valor `m + (minimos.margem(nome) or 0) - s` para os itens com `s < m`.
- `estoque/persistencia_minimos.py`:
  - `salvar_minimos(minimos, caminho)`: monta o dicionário só com os itens que têm mínimo, com `{"minimo": m}` mais `"margem": g` quando a margem está definida. Grava com `open(caminho, "w", encoding="utf-8")` e `json.dump(..., sort_keys=True)`. Para isso, lê os dados internos de `Minimos`, que está no mesmo pacote.
  - `carregar_minimos(caminho) -> Minimos`: lê o arquivo em UTF-8 com `json.load`. Para cada `nome, dados`: se `"minimo"` não estiver em `dados`, lança `ValueError(f"minimo ausente para {nome}")`; senão, chama `definir_minimo` e, se houver `"margem"`, `definir_margem`. Não captura `FileNotFoundError` nem `json.JSONDecodeError`.
- Só a biblioteca padrão (`json`). Nenhuma dependência nova.

## Estratégia de teste

- Os testes usam `unittest`, em `tests/test_*.py`, e a suíte roda com `python3 -m unittest discover -s tests -v` na raiz do repositório.
- RF01: `tests/test_minimos.py` (novo) cobre definir e ler, None para valor não definido, mínimo 0 e negativo com ValueError e a mensagem, margem 0 aceita e margem negativa com ValueError e a mensagem. Comando: `python3 -m unittest tests.test_minimos -v`.
- RF02: estender o `tests/test_consultas.py` que já existe (não criar `tests/test_consultas_minimos.py`). Cobre a ordenação por falta, o desempate alfabético, saldo igual ao mínimo fora da lista, item sem mínimo fora da lista, item nunca adicionado com falta = minimo e lista vazia. Comando: `python3 -m unittest tests.test_consultas -v`.
- RF03: `tests/test_reposicao_sugerida.py` (novo) cobre o cálculo com margem, a margem ausente tratada como 0, os itens fora do resultado e `{}`. Comando: `python3 -m unittest tests.test_reposicao_sugerida -v`.
- RF04: `tests/test_persistencia_minimos.py` (novo), com `tempfile`. Cobre o conteúdo gravado (chaves ordenadas, margem omitida, item só com margem fora do arquivo), salvar e carregar sem perda, `ValueError("minimo ausente para X")`, ValueError em valor inválido, `FileNotFoundError` e `json.JSONDecodeError` passando sem conversão. Comando: `python3 -m unittest tests.test_persistencia_minimos -v`.
- Regressão: `python3 -m unittest discover -s tests -v` termina com exit 0, incluindo os testes que já existem de `itens_zerados`, `Estoque` e `Limites`.
