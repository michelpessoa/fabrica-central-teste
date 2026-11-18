---
id: SPEC-ZERO-0001
type: SPEC
title: "Aviso por e-mail de mudanças em tarefas, com histórico e opção de desligar por usuário"
status: approved
project: "ZERO"
owner: "Michel Pessoa"
created: "2026-11-18"
updated: "2026-11-18"
relates_to: []
parent_rfc: null
parent_adr: null
sizing: "medium"
approved_by:
  kind: "human"
  actor: "michelpessoa"
  at: "2026-11-18T23:37:33Z"
  content_sha256: "a468afff2a926c8113e8d0a50710cf454bcf53fcf89f7205abddbf23168666c0"
  evidence: "https://github.com/michelpessoa/fabrica-central-teste/pull/4#pullrequestreview-5966976811"
supersedes: null
superseded_by: null
tags: [fabrica-descoberta]
---

# Aviso por e-mail de mudanças em tarefas, com histórico e opção de desligar por usuário

## Objetivo

Criar o pacote Python 3 `tarefas`, sem dependências externas (só a biblioteca padrão). Ele cadastra usuários e tarefas em arquivos JSON locais, guarda o histórico das mudanças de cada tarefa e avisa por e-mail o responsável quando a tarefa muda. Cada usuário pode desligar e religar o próprio aviso. Nesta primeira entrega o uso é por um processo só, sem administrador.

## Requisitos funcionais

| RF-ID | Requisito | Critério de aceite (EARS) | Arquivos |
|---|---|---|---|
| RF01 | Cadastro de usuários | Quando o chamador usar `cadastrar_usuario(diretorio, login, email)`, o sistema deve gravar o login e o e-mail em `<diretorio>/usuarios.json`, com o aviso ligado por padrão. | `tarefas/__init__.py`, `tarefas/usuarios.py`, `tests/test_usuarios.py` |
| RF02 | Criação de tarefa | Quando o chamador usar `criar_tarefa(diretorio, titulo, responsavel, descricao="", prazo=None, status="pendente")` com dados válidos, o sistema deve gravar `<diretorio>/tarefa_<id>.json` com histórico vazio e devolver o id inteiro sequencial (1, 2, 3...), sem enviar aviso. Se o título for vazio, o prazo não for data `AAAA-MM-DD`, o status não estiver em `pendente`, `em_andamento` ou `concluida`, ou o responsável não estiver cadastrado, o sistema deve levantar `ValueError` sem gravar nada. | `tarefas/cadastro.py`, `tests/test_cadastro.py` |
| RF03 | Alteração com histórico | Quando o chamador usar `alterar_tarefa(diretorio, id, autor, enviador=..., **campos)` com campos entre `titulo`, `descricao`, `prazo`, `status` e `responsavel`, o sistema deve gravar os novos valores e, para cada campo cujo valor mudou, um registro no histórico com `campo`, `valor_antigo`, `valor_novo`, `autor` (login) e `data_hora` em ISO 8601 UTC. Campo com valor igual ao atual é ignorado, sem histórico nem aviso. Se houver campo desconhecido, valor inválido (mesmas regras do RF02), autor ou responsável não cadastrado, ou tarefa inexistente, o sistema deve levantar `ValueError` sem gravar nada. | `tarefas/cadastro.py`, `tests/test_alteracao.py` |
| RF04 | Aviso por e-mail | Quando uma alteração mudar ao menos um campo, o sistema deve enviar na hora um único e-mail, pelo `enviador`, ao responsável. Se o responsável mudou, o e-mail vai ao antigo e ao novo. Ficam de fora o destinatário que é o próprio autor e quem está com o aviso desligado. O assunto deve ser `Tarefa <id> alterada: <título>` e o corpo deve ter uma linha por campo mudado (campo, valor antigo, valor novo) e o login do autor. | `tarefas/cadastro.py`, `tests/test_aviso.py` |
| RF05 | Desligar e religar aviso | Quando o chamador usar `desligar_aviso(diretorio, login)` ou `ligar_aviso(diretorio, login)`, o sistema deve gravar a opção só daquele login, valendo para todas as tarefas dele. Repetir a chamada não muda o resultado. Com login não cadastrado, o sistema deve levantar `ValueError`. | `tarefas/usuarios.py`, `tests/test_usuarios.py` |
| RF06 | Falha de envio | Se o `enviador` levantar exceção ao enviar para um destinatário, o sistema deve manter gravadas a mudança e o histórico, registrar a falha com `logging` (só o login, sem e-mail), seguir para os demais destinatários e fazer `alterar_tarefa` devolver a lista dos logins cujo envio falhou (lista vazia se nada falhou), sem levantar exceção. | `tarefas/cadastro.py`, `tests/test_aviso.py` |
| RF07 | Consulta do histórico | Quando qualquer chamador usar `historico(diretorio, id)`, o sistema deve devolver a lista de registros da tarefa em ordem cronológica. Com tarefa inexistente, o sistema deve levantar `ValueError`. | `tarefas/cadastro.py`, `tests/test_alteracao.py` |
| RF08 | Envio por SMTP | O sistema deve oferecer a interface abstrata `Enviador` com `enviar(destinatario, assunto, corpo)` e a implementação `EnviadorSMTP`, configurada no construtor (`host`, `porta`, `remetente` e credenciais opcionais), que envia por `smtplib`. Os testes usam `EnviadorFalso`, que só registra as mensagens. | `tarefas/envio.py`, `tests/test_envio.py` |

## Requisitos transversais (sweep)

| Categoria | Destino (RF-ID ou n/a) | Motivo (obrigatório se n/a) |
|---|---|---|
| Autorização / permissão | n/a | Biblioteca sem sessão nem administrador nesta entrega: o chamador passa o próprio login a `desligar_aviso` e `ligar_aviso`, que só alteram aquele login (RF05). O histórico pode ser lido por qualquer chamador, por decisão do owner. |
| Concorrência | n/a | O owner definiu uso por um processo só, sem concorrência nesta entrega. |
| Idempotência | RF03 | Valor igual ao atual é ignorado, sem histórico nem aviso. Ligar e desligar repetidos também não mudam nada (RF05). |
| Observabilidade | RF06 | Falhas de envio vão para o `logging` só com o login, nunca com o e-mail. |
| Falha de dependência externa | RF06 | Uma falha no SMTP não desfaz a alteração e aparece na lista devolvida. |
| Validação de entrada | RF03 | Campos, status, prazo, título e logins são validados, e uma entrada inválida levanta `ValueError` sem gravar nada (RF02 e RF03). |
| Limite de volume / rate | n/a | Não há limite de volume nem de taxa nesta entrega: o aviso sai na hora, um por alteração, sem fila. |

## Contratos técnicos

- Pacote `tarefas`. `tarefas/__init__.py` exporta `cadastrar_usuario`, `criar_tarefa`, `alterar_tarefa`, `historico`, `desligar_aviso`, `ligar_aviso`, `Enviador`, `EnviadorSMTP` e `EnviadorFalso`.
- Todas as funções recebem `diretorio` como primeiro parâmetro, e os testes usam `tempfile.TemporaryDirectory`.
- `<diretorio>/usuarios.json` guarda, para cada login, o e-mail e o booleano `aviso` (padrão `true`). É o único lugar onde o e-mail fica.
- `<diretorio>/tarefa_<id>.json` guarda `id`, `titulo`, `descricao`, `prazo` (string `AAAA-MM-DD` ou `null`), `status`, `responsavel` (login) e `historico` (lista de `{campo, valor_antigo, valor_novo, autor, data_hora}`). O id é o maior id existente + 1, começando em 1.
- `alterar_tarefa(diretorio, id, autor, enviador, **campos) -> list[str]` devolve os logins cujo envio falhou. O parâmetro `enviador` é uma decisão de desenho desta SPEC, necessária para injetar o envio, e deve ser confirmada na revisão.
- `Enviador.enviar(destinatario: str, assunto: str, corpo: str) -> None` levanta exceção em caso de falha. `EnviadorFalso.mensagens` é a lista de `(destinatario, assunto, corpo)`.
- `data_hora` é gerada com `datetime.now(timezone.utc).isoformat()`.

## Estratégia de teste

- `unittest` da biblioteca padrão, rodado com `python3 -m unittest discover -s tests -v`.
- Cada RF tem testes no seu arquivo `tests/test_*.py`, todos usando diretório temporário.
- Os avisos são verificados pelo `EnviadorFalso`. A falha de envio é simulada por um enviador que levanta exceção, e o log é conferido com `assertLogs`.
- `EnviadorSMTP` é testado com `unittest.mock.patch('smtplib.SMTP')`, sem rede.
- Um teste confere que nenhum e-mail aparece nos arquivos `tarefa_<id>.json`.

## Fora de escopo

- Administrador e controle de acesso com autenticação.
- Fila, lote ou reenvio de e-mails que falharam.
- Regra de retenção do histórico.
- Concorrência entre processos ou threads.
- Aviso na criação da tarefa.
- Comportamento de `cadastrar_usuario` com login já existente, que ainda precisa ser definido pelo owner.
- Remoção de tarefas e de usuários.
