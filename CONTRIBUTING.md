# Contributing to resumoGrupo

Obrigado por ajudar a tornar este projeto mais confiável, claro e fácil de adaptar.

## PT-BR

### Antes de contribuir

- Leia o [README](README.md) e entenda o fluxo de Captura, Resumo Diário e Resumo Semanal.
- Abra uma issue antes de mudanças grandes, especialmente alterações no formato dos payloads ou no comportamento dos workflows.
- Mantenha o projeto agnóstico de provedor sempre que isso não reduzir a clareza ou a segurança.

### O que é bem-vindo

- Correções nos workflows n8n e nos exemplos de integração.
- Melhorias de confiabilidade, segurança, privacidade e tratamento de erros.
- Compatibilidade documentada com outros provedores de WhatsApp ou IA.
- Melhorias de documentação, tradução, testes locais e exemplos reproduzíveis.

### Segurança e privacidade

- Nunca faça commit de chaves de API, tokens, URLs de webhook reais, JIDs privados, exports de execução ou dados de conversas.
- Use apenas placeholders como `YOUR_API_KEY`, `YOUR_GROUP_JID@g.us` e URLs fictícias.
- Não inclua mensagens reais de grupos em issues, pull requests ou fixtures.
- Para uma vulnerabilidade, não publique detalhes exploráveis em uma issue aberta; entre em contato com o mantenedor por um canal privado antes de divulgar.

### Como validar uma mudança

1. Importe o workflow alterado em uma instância n8n descartável.
2. Confira se os placeholders estão explícitos e se nenhum segredo foi incluído.
3. Teste o caminho de sucesso, o grupo sem mensagens e a falha do provedor externo quando a mudança tocar esses fluxos.
4. Desative o node de envio e valide o texto gerado antes de qualquer teste em um grupo real.
5. Para mudanças no parsing, teste mensagens normais, mensagens `fromMe`, timestamps ausentes e payloads inesperados.
6. Atualize o README quando a instalação, os campos ou as limitações mudarem.

### Pull requests

- Use um título curto e descritivo, por exemplo `fix: handle missing message timestamp`.
- Explique o problema, a solução, como validou e quais limitações permanecem.
- Mantenha cada PR focado em uma mudança lógica.
- Não reescreva workflows inteiros por motivos de formatação; diffs pequenos são mais fáceis de revisar.
- Preserve os nomes dos nodes e a estrutura dos dados quando não houver motivo técnico para alterá-los.
- Se a mudança alterar comportamento, inclua antes/depois no texto do PR.

### Disciplina Git

- Trabalhe em uma branch própria baseada em `main`.
- Faça commits pequenos e lógicos.
- Revise `git status` e o diff antes de cada commit.
- Nunca use force push em `main`.
- Não inclua arquivos temporários, credenciais ou exports de execução.

### Licença

Ao contribuir, você concorda que suas contribuições sejam distribuídas sob a [licença MIT](LICENSE).

## English

Thank you for helping make this project more reliable, understandable, and easier to adapt.

### Before contributing

- Read the [README](README.md) and understand the Capture, Daily Summary, and Weekly Summary flows.
- Open an issue before large changes, especially payload-format changes or workflow behavior changes.
- Keep the project provider-agnostic when that does not reduce clarity or security.

### Welcome contributions

- Fixes to the n8n workflows and integration examples.
- Improvements to reliability, security, privacy, and error handling.
- Documented compatibility with other WhatsApp or AI providers.
- Documentation, translation, local testing, and reproducible-example improvements.

### Security and privacy

- Never commit API keys, tokens, real webhook URLs, private JIDs, execution exports, or conversation data.
- Use placeholders such as `YOUR_API_KEY`, `YOUR_GROUP_JID@g.us`, and fictional URLs.
- Do not include real group messages in issues, pull requests, or fixtures.
- For a vulnerability, do not publish exploitable details in an open issue; contact the maintainer through a private channel before disclosure.

### Validating a change

1. Import the changed workflow into a disposable n8n instance.
2. Check that placeholders are explicit and no secret was included.
3. Test the success path, an empty group day, and an external-provider failure when the change touches those flows.
4. Disable the sending node and validate the generated text before testing in a real group.
5. For parsing changes, test normal messages, `fromMe` messages, missing timestamps, and unexpected payloads.
6. Update the README when installation, fields, or limitations change.

### Pull requests

- Use a short, descriptive title, such as `fix: handle missing message timestamp`.
- Explain the problem, the solution, how you validated it, and any remaining limitations.
- Keep each PR focused on one logical change.
- Do not rewrite entire workflows for formatting reasons; small diffs are easier to review.
- Preserve node names and data structures unless there is a technical reason to change them.
- If behavior changes, include before/after behavior in the PR description.

### Git discipline

- Work on a branch based on `main`.
- Make small, logical commits.
- Review `git status` and the diff before each commit.
- Never force-push to `main`.
- Do not include temporary files, credentials, or execution exports.

### License

By contributing, you agree that your contributions will be distributed under the [MIT license](LICENSE).
