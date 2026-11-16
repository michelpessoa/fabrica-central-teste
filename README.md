# fabrica-central-teste

Central de teste do framework de rastreabilidade, usado só pela Fábrica Agêntica nos testes locais
(RFC-DTF-0009, decisão de 2026-11-16: testes híbridos).

- **Só dados fictícios.** Os projetos `ZERO` e `COBAIA` apontam para os repositórios de teste
  [`fabrica-zero`](https://github.com/michelpessoa/fabrica-zero) e
  [`fabrica-cobaia`](https://github.com/michelpessoa/fabrica-cobaia).
- **`_framework/` fixado na 2.3.2**, o mesmo commit (`dd4a510`) que a fábrica testa no CI. Não é espelho
  do kit: só muda quando a versão fixada da fábrica muda.
- **Pode quebrar.** A fábrica abre PRs aqui com a conta `fabricaagenticamichel` para testar a escrita no
  central, a aprovação por review e a release dupla. O central real só recebe o ensaio final.

Validação, como no central real:

```bash
git config core.hooksPath .githooks
python3 _framework/scripts/framework_check.py --auto
```
