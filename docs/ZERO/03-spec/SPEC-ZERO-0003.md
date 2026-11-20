---
id: SPEC-ZERO-0003
type: SPEC
title: "Listagem de tarefas com filtros (biblioteca e comando python -m tarefas listar)"
status: approved
project: "ZERO"
owner: "Michel Pessoa"
created: "2026-11-20"
updated: "2026-11-20"
relates_to: []
parent_rfc: null
parent_adr: null
sizing: "medium"
approved_by:
  kind: "human"
  actor: "michelpessoa"
  at: "2026-11-20T23:33:30Z"
  content_sha256: "50a26c48d9b0f747b5300cce9e29e2cac7b1821299de9ac848b1a9228d16e278"
  evidence: "https://github.com/michelpessoa/fabrica-central-teste/pull/7#pullrequestreview-6000806241"
supersedes: null
superseded_by: null
tags: [fabrica-descoberta]
---

# Listagem de tarefas com filtros (biblioteca e comando python -m tarefas listar)

## Objetivo

Criar o pacote `tarefas` (ainda inexistente no repositório) com a função `listar(diretorio, responsavel=None, status=None, texto=None)`, que lê os arquivos `tarefa_<id>.json` de um diretório, filtra e ordena as tarefas, e o comando `python3 -m tarefas listar`, que imprime o resultado em formato de linha fixo. Só biblioteca padrão do Python 3, sem recurso mais novo que 3.10.

# Parte 1 — Requisito (o QUÊ)

## Requisitos funcionais

| RF-ID | Requisito | Critério de aceite (EARS) | Arquivos |
|---|---|---|---|
| RF01 | `listar(diretorio, responsavel=None, status=None, texto=None)` lê os `tarefa_<id>.json` do diretório, filtra e ordena | Quando `listar` for chamada com um diretório existente, o sistema deve ler só os arquivos cujo nome casa com `tarefa_<id>.json` (os demais são ignorados), aplicar os filtros combinados com E — `responsavel` igual ao login exatamente (com diferença de maiúsculas, sem normalização; `None` ou `""` = sem filtro), `status` igual ao valor (`None` = sem filtro), `texto` contido no `titulo` ou na `descricao` comparando com `casefold` (`None`, vazio ou só espaços = sem filtro) — e devolver a lista de tarefas (dicts) em ordem crescente do campo `id` como inteiro; diretório existente e vazio devolve `[]`. | tarefas/__init__.py, tarefas/busca.py, tests/test_busca.py |
| RF02 | Validação de status, de diretório e de arquivo ilegível na biblioteca, sempre com `ValueError` | Se `status` não for `None` nem um de `pendente`, `em_andamento`, `concluida`, o sistema deve levantar `ValueError("status inválido: <valor>")` sem ler nada; se o diretório não existir ou não for diretório, deve levantar `ValueError("diretório <dir> não existe")`; se um `tarefa_<id>.json` não abrir, tiver JSON malformado, faltar campo obrigatório, tiver tipo diferente do descrito, tiver `id` diferente do número do nome, ou repetir id de outro arquivo (zeros à esquerda), deve levantar `ValueError("tarefa_<id>.json ilegível: <motivo>")` sem devolver lista parcial. | tarefas/busca.py, tests/test_busca.py |
| RF03 | Comando `python3 -m tarefas listar [--responsavel LOGIN] [--status STATUS] [--texto TEXTO] [--diretorio DIR]` com o formato de linha | Quando o comando for executado e houver tarefas no resultado, o sistema deve imprimir na saída padrão uma linha por tarefa, na ordem de `listar`, no formato `#<id> [<status>] <titulo> — <responsavel>, prazo <AAAA-MM-DD>` (ou terminando em `— <responsavel>, sem prazo` quando `prazo` for null), usando `--diretorio` com padrão `.`, e sair com 0. | tarefas/__main__.py, tests/test_cli_listar.py |
| RF04 | Lista vazia no comando | Quando o resultado do comando for vazio (inclusive diretório existente sem tarefas ou login sem tarefas), o sistema deve imprimir exatamente `nenhuma tarefa encontrada` na saída padrão e sair com 0. | tarefas/__main__.py, tests/test_cli_listar.py |
| RF05 | Erros do comando na saída de erro com códigos 2 e 1 | Se `--status` for inválido, o sistema deve imprimir `erro: --status deve ser pendente, em_andamento ou concluida` na saída de erro e sair com 2; se o diretório não existir, `erro: diretório <dir> não existe` e sair com 2; em erro de uso do argparse, a mensagem padrão do argparse e saída 2; se um arquivo for ilegível, `erro: tarefa_<id>.json ilegível: <motivo>` e sair com 1; em todos esses casos, nada na saída padrão. | tarefas/__main__.py, tests/test_cli_listar.py |

## Requisitos transversais (sweep)

| Categoria | Destino (RF-ID ou n/a) | Motivo (obrigatório se n/a) |
|---|---|---|
| Autorização / permissão | n/a | Comando local de leitura; o filtro de responsável não confere cadastro nem permissão de usuário. |
| Concorrência | n/a | Leitura única de arquivos locais num processo; não há escrita nem estado compartilhado. |
| Idempotência | n/a | `listar` só lê arquivos; repetir a chamada não altera nada. |
| Observabilidade | RF05 | Todo erro do comando vai para a saída de erro com mensagem e código de saída definidos. |
| Falha de dependência externa | RF02 | A única dependência é o sistema de arquivos: diretório inexistente e arquivo que não abre viram `ValueError` com mensagem definida. |
| Validação de entrada | RF02 | Status do filtro, diretório e conteúdo de cada `tarefa_<id>.json` são validados antes de devolver qualquer resultado. |
| Limite de volume / rate | n/a | Não há limite de quantidade de tarefas definido pelo owner; sem paginação nesta entrega. |

## Fora de escopo

- Criar, editar, concluir ou apagar tarefas; uso do campo `historico`.
- Paginação, ordenação diferente de `id` crescente, saída em JSON.
- Validação do formato da data de `prazo` além do tipo (string ou null) e conferência de cadastro de usuários.
- Exceções próprias; `choices` do argparse para `--status`.
- Dependências externas, pytest, arquivo de dependências.

# Parte 2 — Desenho (o COMO e o ONDE)

## Contratos técnicos

**Arquivo de tarefa** `tarefa_<id>.json`: nome casa com a regex `^tarefa_(\d+)\.json$`; outros nomes são ignorados. Conteúdo é um objeto JSON com:

| Campo | Tipo | Regra |
|---|---|---|
| id | inteiro (não booleano) | igual a `int(<id>)` do nome do arquivo |
| titulo | string | não vazia |
| descricao | string | pode ser `""` |
| prazo | string `AAAA-MM-DD` ou null | |
| status | string | `pendente`, `em_andamento` ou `concluida` |
| responsavel | string | login |
| historico | lista | não usado por `listar`, não validado |

Campo obrigatório ausente ou com tipo diferente, `id` diferente do nome, JSON malformado, erro ao abrir, ou dois arquivos com o mesmo id numérico (`tarefa_7.json` e `tarefa_007.json`) → `ValueError("tarefa_<id>.json ilegível: <motivo>")`, onde `tarefa_<id>.json` é o nome do arquivo (de um dos dois no caso de repetição) e `<motivo>` é texto livre. Nenhuma lista parcial é devolvida.

**Biblioteca** (`tarefas/busca.py`, reexportada em `tarefas/__init__.py`):

```python
def listar(diretorio, responsavel=None, status=None, texto=None) -> list[dict]
```

Ordem de validação: 1) `status` (antes de qualquer leitura) → `ValueError("status inválido: <valor>")`; 2) diretório inexistente ou não diretório → `ValueError("diretório <dir> não existe")`, com `<dir>` como foi informado; 3) leitura e validação de todos os arquivos; 4) filtros (E); 5) ordenação por `id` inteiro crescente. Retorna os dicts lidos do JSON.

Filtros: `responsavel` — igualdade exata (`"Ana" != "ana"`), `None` ou `""` = sem filtro. `status` — igualdade, `None` = sem filtro. `texto` — `None` ou `texto.strip() == ""` = sem filtro; senão `texto.casefold()` contido em `titulo.casefold()` ou em `descricao.casefold()`.

**Comando** (`tarefas/__main__.py`, argparse com subcomando `listar`):

- Opções: `--responsavel`, `--status`, `--texto`, `--diretorio` (padrão `.`), todas opcionais.
- O comando valida `--status` antes de chamar `listar` e diferencia o tipo de erro sem depender do texto da biblioteca além do necessário (ex.: checar `os.path.isdir` para o código 2).

| Situação | stdout | stderr | código |
|---|---|---|---|
| com resultado | uma linha por tarefa | — | 0 |
| resultado vazio | `nenhuma tarefa encontrada` | — | 0 |
| `--status` inválido | nada | `erro: --status deve ser pendente, em_andamento ou concluida` | 2 |
| diretório inexistente ou não diretório | nada | `erro: diretório <dir> não existe` (`.` se omitido) | 2 |
| erro de uso do argparse | nada | mensagem padrão do argparse | 2 |
| arquivo ilegível | nada | `erro: tarefa_<id>.json ilegível: <motivo>` | 1 |

Formato da linha: `#<id> [<status>] <titulo> — <responsavel>, prazo <AAAA-MM-DD>`; com `prazo` null: `#<id> [<status>] <titulo> — <responsavel>, sem prazo`. Separador: travessão U+2014 com um espaço de cada lado. Exemplos: `#3 [pendente] Revisar contrato — ana, prazo 2026-12-01` e `#4 [concluida] Pagar boleto — bruno, sem prazo`.

## Estratégia de teste

- Só `unittest` da biblioteca padrão; comando de testes: `python3 -m unittest discover -s tests -v`; build: `python3 -m compileall -q .`.
- `tests/test_busca.py` (RF01, RF02): cria arquivos `tarefa_<id>.json` em `tempfile.TemporaryDirectory` e cobre: ordem numérica (`tarefa_10` depois de `tarefa_9`); arquivo com outro nome ignorado; cada filtro isolado e combinados; `responsavel` exato com maiúsculas e `""` como sem filtro; `texto` com casefold em título e descrição e só espaços como sem filtro; diretório vazio → `[]`; status inválido → `ValueError` mesmo com diretório inexistente (prova que não leu nada); diretório inexistente e caminho que é arquivo → mensagem `diretório <dir> não existe`; JSON malformado, campo faltando, tipo errado, id diferente do nome, ids repetidos com zeros à esquerda → `ValueError` com `ilegível`.
- `tests/test_cli_listar.py` (RF03–RF05): `subprocess.run([sys.executable, "-m", "tarefas", "listar", ...])` com `PYTHONPATH` apontando para a raiz do repositório, `PYTHONIOENCODING=utf-8` e `encoding="utf-8"`; confere stdout, stderr e código de saída de cada linha da tabela do comando, inclusive o padrão `--diretorio .` rodando com `cwd` no diretório temporário e `--responsavel ""` como sem filtro.
- Critérios: `RF01`/`RF02`: `python3 -m unittest tests.test_busca -v` termina com 0; `RF03`–`RF05`: `python3 -m unittest tests.test_cli_listar -v` termina com 0; Regressão: `python3 -m compileall -q . && python3 -m unittest discover -s tests -v` termina com 0.
