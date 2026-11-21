---
id: SPEC-ZERO-0004
type: SPEC
title: "Resumo das tarefas por responsável e status"
status: approved
project: "ZERO"
owner: "Michel Pessoa"
created: "2026-11-21"
updated: "2026-11-21"
relates_to: []
parent_rfc: null
parent_adr: null
sizing: "medium"
approved_by:
  kind: "human"
  actor: "michelpessoa"
  at: "2026-11-21T01:44:55Z"
  content_sha256: "6f11dd568b6eaa4b6d2c333d87ede58163e65c7399e73b8a00165c7efe302365"
  evidence: "https://github.com/michelpessoa/fabrica-central-teste/pull/8#pullrequestreview-6001919393"
supersedes: null
superseded_by: null
tags: [fabrica-descoberta]
---

# Resumo das tarefas por responsável e status

## Objetivo

Criar o pacote `tarefas` com a função de biblioteca `resumir(diretorio)` e o comando `python3 -m tarefas resumo [--diretorio DIR]`. A função lê os arquivos `tarefa_<id>.json` (formato da SPEC-ZERO-0001) de um diretório e conta as tarefas por responsável (login) e status. O comando imprime uma linha por login e uma linha de total. O pacote ainda não existe no repositório e é criado por esta mudança.

# Parte 1 — Requisito (o QUÊ)

## Requisitos funcionais

| RF-ID | Requisito | Critério de aceite (EARS) | Arquivos |
|---|---|---|---|
| RF01 | `resumir(diretorio)` lê todos os `tarefa_<id>.json` do diretório e conta as tarefas por `responsavel` e `status` (`pendente`, `em_andamento`, `concluida`). Arquivos cujo nome não casa com `tarefa_<id>.json` são ignorados. | Quando `resumir(diretorio)` for chamada com um diretório que tem tarefas válidas, o sistema deve devolver um dict que leva cada login com ao menos uma tarefa a um dict com as chaves `pendente`, `em_andamento` e `concluida` (inteiros, zeros incluídos), e deve ignorar os arquivos de outro nome. | tarefas/__init__.py, tarefas/resumo.py, tests/test_resumo.py |
| RF02 | `resumir` valida o diretório e cada arquivo e levanta `ValueError` sem resultado parcial. | Se o diretório não existir ou não for diretório, o sistema deve levantar `ValueError("diretório <dir> não existe")`; se um `tarefa_<id>.json` não abrir, não for JSON válido, não for objeto, faltar `id`, `titulo`, `descricao`, `prazo`, `status` ou `responsavel`, tiver tipo errado, `titulo` ou `responsavel` vazio, `prazo` fora de `AAAA-MM-DD` e não null, `status` fora dos três valores, `id` diferente do número do nome ou `id` repetido por zeros à esquerda, o sistema deve levantar `ValueError("tarefa_<id>.json ilegível: <motivo>")`. | tarefas/resumo.py, tests/test_resumo.py |
| RF03 | O comando `python3 -m tarefas resumo [--diretorio DIR]` imprime uma linha por login e a linha de total. | Quando o usuário rodar `python3 -m tarefas resumo` sobre um diretório com tarefas válidas, o sistema deve imprimir na saída padrão, em ordem alfabética exata do login (sem casefold), uma linha `<login>: pendente <n>, em_andamento <n>, concluida <n>, total <n>` por login, seguida de `total: pendente <n>, em_andamento <n>, concluida <n>, total <n>` com a soma de todos os logins, sem cabeçalho nem linha em branco, e sair com 0; sem `--diretorio`, o sistema deve usar o diretório atual. | tarefas/__main__.py, tests/test_cli_resumo.py |
| RF04 | Diretório existente sem nenhum `tarefa_<id>.json` imprime a mensagem de vazio. | Quando o diretório existir e não tiver nenhum `tarefa_<id>.json` (vazio ou só com arquivos de outro nome), o sistema deve imprimir exatamente `nenhuma tarefa encontrada` na saída padrão, sem linha de total, nada na saída de erro, e sair com 0; e `resumir` deve devolver `{}`. | tarefas/resumo.py, tarefas/__main__.py, tests/test_resumo.py, tests/test_cli_resumo.py |
| RF05 | Erros do comando vão para a saída de erro, com códigos 2 e 1. | Se o diretório não existir ou não for diretório, o sistema deve imprimir `erro: diretório <dir> não existe` (com `<dir>` como informado, ou `.` sem `--diretorio`) na saída de erro e sair com 2; se houver opção desconhecida ou subcomando ausente, o sistema deve sair com 2 com a mensagem padrão do argparse; se um arquivo for ilegível, o sistema deve imprimir `erro: tarefa_<id>.json ilegível: <motivo>` na saída de erro e sair com 1; em todos os casos, nada na saída padrão. | tarefas/__main__.py, tests/test_cli_resumo.py |

## Requisitos transversais (sweep)

| Categoria | Destino (RF-ID ou n/a) | Motivo (obrigatório se n/a) |
|---|---|---|
| Autorização / permissão | n/a | Comando local de leitura, sem usuários nem controle de acesso; arquivo que não abre por permissão cai em ilegível (RF02). |
| Concorrência | n/a | Leitura única e síncrona de arquivos locais, sem escrita nem estado compartilhado. |
| Idempotência | n/a | Operação somente de leitura; repetir dá o mesmo resultado para os mesmos arquivos. |
| Observabilidade | RF05 | Erros vão para a saída de erro com mensagem e código de saída definidos. |
| Falha de dependência externa | n/a | Sem dependência externa; só sistema de arquivos local e biblioteca padrão. |
| Validação de entrada | RF02 | Diretório e cada arquivo são validados antes de qualquer saída. |
| Limite de volume / rate | n/a | Nenhum limite de volume foi pedido; o diretório é lido inteiro. |

## Fora de escopo

- Criar, editar ou remover tarefas; uso do campo `historico`.
- Filtros por login, status ou prazo; outras opções do comando além de `--diretorio`.
- Formatos de saída alternativos (JSON, CSV) e categoria "sem responsável".
- Leitura recursiva de subdiretórios.

# Parte 2 — Desenho (o COMO e o ONDE)

## Contratos técnicos

- Linguagem: Python 3, só biblioteca padrão (json, pathlib, argparse, re, sys); nada mais novo que Python 3.10. Sem dependências externas, sem pytest, sem arquivo de dependências.
- `tarefas/__init__.py`: marca o pacote e reexporta `resumir`.
- `tarefas/resumo.py`: `resumir(diretorio: str | os.PathLike) -> dict[str, dict[str, int]]`. Casamento de nome por `re.fullmatch(r"tarefa_(\d+)\.json", nome)` nos arquivos do próprio diretório. Valida todos os arquivos antes de devolver; no primeiro erro levanta `ValueError` com as mensagens de RF02 (o `<id>` da mensagem é o trecho do nome do arquivo; `<motivo>` é livre). Ids repetidos por zeros à esquerda (ex.: `tarefa_1.json` e `tarefa_01.json`) tornam ilegível o segundo arquivo na ordem de nome. `bool` não é aceito como inteiro em `id`. Diretório inexistente: `ValueError("diretório <dir> não existe")`.
- `tarefas/__main__.py`: argparse com subcomando obrigatório `resumo` e opção `--diretorio` (padrão `.`). Antes de chamar `resumir`, checa se o caminho é diretório; se não, imprime `erro: diretório <dir> não existe` na saída de erro e `sys.exit(2)`. `ValueError` de `resumir` vira `erro: <mensagem>` na saída de erro e `sys.exit(1)`. Toda a saída é montada antes de imprimir, sem resultado parcial. Todas as mensagens terminam com quebra de linha.
- Formato da saída: ver RF03 e RF04; exemplo com ana (2 pendentes, 1 concluída) e bruno (1 em andamento):
  - `ana: pendente 2, em_andamento 0, concluida 1, total 3`
  - `bruno: pendente 0, em_andamento 1, concluida 0, total 1`
  - `total: pendente 2, em_andamento 1, concluida 1, total 4`

## Estratégia de teste

- unittest da biblioteca padrão, rodado com `python3 -m unittest discover -s tests -v`; build com `python3 -m compileall -q .`.
- `tests/test_resumo.py`: cria os `tarefa_<id>.json` em `tempfile.TemporaryDirectory` e testa `resumir` (contagem, arquivos de outro nome ignorados, diretório vazio, cada caso de ilegível e diretório inexistente com `assertRaises(ValueError)`).
- `tests/test_cli_resumo.py`: roda `python3 -m tarefas resumo` com `subprocess` (cwd na raiz do repositório ou `--diretorio`), verificando saída padrão, saída de erro e código de saída para RF03, RF04 e RF05, inclusive o caso sem `--diretorio` (cwd no diretório temporário com `PYTHONPATH` apontando para a raiz).
- Regressão: `python3 -m unittest discover -s tests -v` com exit 0.
