# Compactação automática do contexto no OpenCode

## Pergunta
A compactação acontece automaticamente ou precisa ser executada manualmente?

## Resposta
No OpenCode V2, a compactação automática fica ativada por padrão. Quando a conversa se aproxima do limite de contexto do modelo, o OpenCode resume o histórico mais antigo e mantém uma parte recente da conversa para continuar. Também pode haver recuperação automática e uma nova tentativa se o provedor rejeitar uma solicitação por excesso de contexto.

A compactação é com perda: detalhes antigos podem ser resumidos. É possível solicitar compactação manual antes do limite ou ajustar/desativar o comportamento em `compaction.auto`; a operação manual não exige que a automação esteja ativada.

## Conceitos principais
- Compactação automática: por padrão, `compaction.auto` é `true`.
- Janela de contexto: limite de conteúdo que o modelo pode processar de uma vez.
- Resumo: reduz o histórico antigo, mas não é uma reprodução literal.
- No OpenCode, mensagens antigas continuam armazenadas na sessão, embora deixem de ser enviadas integralmente ao modelo.

Fonte: https://opencode.ai/docs/compaction/