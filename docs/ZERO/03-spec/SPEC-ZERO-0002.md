---
id: SPEC-ZERO-0002
type: SPEC
title: "ZERO: listagem de tarefas vencidas e a vencer por responsável"
status: in_review
project: "ZERO"
owner: "Michel Pessoa"
created: "2026-11-19"
updated: "2026-11-19"
relates_to: []
parent_rfc: null
parent_adr: null
sizing: "medium"
supersedes: null
superseded_by: null
tags: [fabrica-descoberta]
---

# ZERO: listagem de tarefas vencidas e a vencer por responsável

## Objetivo

Hoje ninguém sabe o que está atrasado sem abrir as tarefas uma por uma. Esta entrega usa o campo `prazo` que a SPEC-ZERO-0001 já define (texto ISO `AAAA-MM-DD`, opcional, validado em `criar_tarefa` e `alterar_tarefa`, com histórico na alteração) e acrescenta ao pacote `tarefas`:

- a função de biblioteca `vencidas(diretorio, dias=7, hoje=None)` em `tarefas/prazos.py`, exportada em `tarefas/__init__.py`;
- o comando `python -m tarefas vencidas [--dias N] [--diretorio <dir>]` em `tarefas/__main__.py`, que imprime as tarefas vencidas e as que vencem nos próximos N dias, agrupadas pelo login do responsável.

Quem usa é uma equipe interna pequena, no mesmo computador, um processo por vez. Só a biblioteca padrão do Python 3.11+, arquivos JSON locais (`<diretorio>/tarefa_<id>.json`) e um processo só. Não há API HTTP nem web.

## Requisitos funcionais

| RF-ID | Requisito | Critério de aceite (EARS) | Arquivos |
|---|---|---|---|
| RF01 | Selecionar e agrupar as tarefas abertas com prazo dentro da janela | Quando `vencidas(diretorio, dias, hoje)` for chamada, o sistema deve devolver `dict[str, list[tuple[dict, int]]]` só com as tarefas de `status` `pendente` ou `em_andamento`, com `prazo` preenchido e `prazo - hoje <= dias` (o que inclui as vencidas e as que vencem hoje). A chave é o login do `responsavel`, `dias` é `(prazo - hoje).days`, as chaves vêm em ordem alfabética do login e cada lista vem ordenada por `prazo` crescente e, no empate, pelo título. Tarefas `concluida` ou sem prazo ficam fora, e quando nada se encaixa o retorno é `{}`. | tarefas/prazos.py, tests/test_prazos.py |
| RF02 | Data de referência e validação de `dias` na biblioteca | Quando `hoje` for `None`, o sistema deve usar `datetime.date.today()` no fuso local; quando `hoje` for informado, deve usar esse valor. Se `dias` for negativo, o sistema deve levantar `ValueError`. | tarefas/prazos.py, tests/test_prazos.py |
| RF03 | Arquivo de tarefa ilegível aborta a listagem | Se um arquivo `tarefa_<id>.json` do diretório não puder ser lido ou interpretado como JSON, ou tiver `prazo` fora do formato `AAAA-MM-DD` ou com data inexistente, o sistema deve levantar `ValueError("tarefa_<id>.json ilegível: <motivo>")` sem devolver lista parcial. Arquivos cujo nome não casa com `tarefa_<id>.json` devem ser ignorados. | tarefas/prazos.py, tests/test_prazos.py |
| RF04 | Formato da saída do comando | Quando o usuário rodar `python -m tarefas vencidas` e houver tarefas no critério, o sistema deve imprimir na saída padrão, para cada login em ordem alfabética, uma linha `<login>:` seguida de uma linha `  - <título>: <situação>` por tarefa, na ordem de RF01, com uma linha em branco entre os grupos, e sair com código 0. A situação é `vencida há N dias`, `vencida há 1 dia`, `vence hoje`, `vence em 1 dia` ou `vence em N dias`. | tarefas/__main__.py, tests/test_cli_vencidas.py |
| RF05 | Mensagem de lista vazia | Quando nenhuma tarefa estiver no critério, inclusive com diretório existente e vazio, o sistema deve imprimir `nada vencido nem a vencer em N dias` (N > 1), `nada vencido nem a vencer em 1 dia` (N = 1) ou `nada vencido nem a vencer hoje` (N = 0) e sair com código 0. | tarefas/__main__.py, tests/test_cli_vencidas.py |
| RF06 | Argumentos padrão e erros do comando | Quando `--dias` for omitido, o sistema deve usar 7; quando `--diretorio` for omitido, deve usar o diretório atual. Se `--dias` for negativo ou não inteiro, o sistema deve imprimir `erro: --dias deve ser um inteiro >= 0` na saída de erro e sair com 2. Se o diretório não existir, deve imprimir `erro: diretório <dir> não existe` na saída de erro e sair com 2. Se RF03 ocorrer, deve imprimir `erro: tarefa_<id>.json ilegível: <motivo>` na saída de erro e sair com 1. Nos três casos, nada é listado na saída padrão. | tarefas/__main__.py, tests/test_cli_vencidas.py |
| RF07 | Exportação na biblioteca | O sistema deve permitir `from tarefas import vencidas`, com a mesma função de `tarefas.prazos.vencidas`. | tarefas/__init__.py, tests/test_prazos.py |

## Requisitos transversais (sweep)

| Categoria | Destino (RF-ID ou n/a) | Motivo (obrigatório se n/a) |
|---|---|---|
| Autorização / permissão | n/a | Decisão do owner: qualquer chamador da biblioteca ou do comando pode listar as tarefas vencidas de todos. Não há login nem permissões. |
| Concorrência | n/a | Um processo por vez no mesmo computador, e a listagem só lê os arquivos, sem escrever nada. |
| Idempotência | n/a | A operação é só de leitura e não tem efeito colateral. Com os mesmos arquivos e o mesmo `hoje`, devolve sempre o mesmo resultado. |
| Observabilidade | RF06 | Erros vão para a saída de erro com mensagem fixa e com códigos de saída distintos (2 para uso inválido, 1 para arquivo ilegível). |
| Falha de dependência externa | RF03 | A única dependência é o sistema de arquivos local. Arquivo ilegível aborta com erro explícito, e diretório inexistente é tratado em RF06. |
| Validação de entrada | RF06 | `--dias` é validado no comando e na biblioteca (RF02), e `prazo` gravado inválido é tratado em RF03. A validação do `prazo` no cadastro continua na SPEC-ZERO-0001. |
| Limite de volume / rate | n/a | O owner não definiu limite. O uso é de uma equipe pequena, local, com leitura de um diretório de JSONs, sem rede. |

## Contratos técnicos

- **Biblioteca** (`tarefas/prazos.py`): `vencidas(diretorio, dias=7, hoje=None) -> dict[str, list[tuple[dict, int]]]`.
  - `diretorio`: caminho do diretório com os `tarefa_<id>.json` da SPEC-ZERO-0001.
  - `hoje`: `datetime.date` ou `None`, que significa `datetime.date.today()`.
  - `tarefa` no par é o dicionário como gravado no JSON (inclui `id`). `dias = (date.fromisoformat(prazo) - hoje).days`: negativo para as vencidas, 0 para hoje, positivo para as a vencer.
  - Entram só `status` em {`pendente`, `em_andamento`} com `prazo` não nulo e `dias_diff <= dias`.
  - Ordem: chaves pelo login em ordem alfabética, listas por `(prazo, título)`.
  - Erros: `ValueError` para `dias < 0` e `ValueError("tarefa_<id>.json ilegível: <motivo>")` para arquivo ilegível ou `prazo` inválido. O texto de `<motivo>` fica livre.
- **Comando** (`tarefas/__main__.py`): `python -m tarefas vencidas [--dias N] [--diretorio DIR]`, com `--dias` padrão 7 e `--diretorio` padrão o diretório atual. Saída na stdout conforme RF04 e RF05. Erros na stderr conforme RF06, com códigos 0, 1 e 2. A mensagem de `--dias` inválido tem de ser exatamente `erro: --dias deve ser um inteiro >= 0`, sem a mensagem padrão do argparse.
- **Situação**: com `d = dias`, `d < -1` vira `vencida há {-d} dias`, `d == -1` vira `vencida há 1 dia`, `d == 0` vira `vence hoje`, `d == 1` vira `vence em 1 dia` e `d > 1` vira `vence em {d} dias`.
- **Exemplo**: hoje 2026-11-18, `--dias 3`, tarefas de `ana` com prazo 2026-11-15 e 2026-11-20 e de `bruno` sem prazo. A saída é `ana:`, depois `  - <título1>: vencida há 3 dias`, depois `  - <título2>: vence em 2 dias`. `bruno` não aparece.

## Estratégia de teste

- `unittest` da biblioteca padrão. A suíte roda com `python3 -m unittest discover -s tests -v`.
- `tests/test_prazos.py`: monta diretórios temporários (`tempfile.TemporaryDirectory`) com `tarefa_<id>.json` e chama `vencidas` com `hoje` fixo. Cobre:
  - o filtro por status, por prazo e pela janela, incluindo `dias=0` com vencimento hoje;
  - a ordenação e o empate por título;
  - o retorno `{}`, o `ValueError` para `dias` negativo e para JSON corrompido ou `prazo` inválido;
  - os arquivos ignorados;
  - `from tarefas import vencidas`.
- `tests/test_cli_vencidas.py`: roda `python -m tarefas vencidas` via `subprocess` com `--diretorio` temporário e confere stdout, stderr e código de saída de RF04 a RF06. Os casos dependentes da data usam prazos relativos a `date.today()`, para não depender de data fixa.
- Comandos por requisito: `python3 -m unittest tests.test_prazos -v` (RF01, RF02, RF03, RF07) e `python3 -m unittest tests.test_cli_vencidas -v` (RF04, RF05, RF06). A regressão roda a suíte inteira.

## Fora de escopo

- Criar campo novo de vencimento: usa-se o `prazo` da SPEC-ZERO-0001.
- Mudar a validação e o histórico de `prazo` em `criar_tarefa` e `alterar_tarefa`, já cobertos pela SPEC-ZERO-0001.
- Avisos por e-mail ou notificações de prazo (`tarefas/envio.py` não muda).
- Filtro por responsável, saída em JSON, API HTTP, interface web, fusos horários diferentes do local.
- Concorrência entre processos e qualquer controle de acesso.
- Dependências fora da biblioteca padrão.
