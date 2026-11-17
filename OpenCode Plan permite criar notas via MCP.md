# OpenCode Plan permite criar notas via MCP

## Decisão implementada

A configuração global do agente Plan permite consultar, criar e atualizar notas pelo servidor MCP `obsidian` usando as ações `obsidian_memory_search`, `obsidian_memory_read`, `obsidian_memory_create` e `obsidian_memory_update`. A ação `obsidian_memory_forget` permanece negada em Plan. `memory_update` permite append ou replace; leia a nota antes, prefira append e use replace só quando o trecho exato ocorrer uma vez.

## Organização do Vault

`memory_create` aceita caminhos relativos `.md` na raiz ou em qualquer subpasta permitida e cria diretórios intermediários. Não há categorias obrigatórias como `Estudos/` ou `LinkedIn/`; o caminho deve seguir a organização existente ou a solicitação do usuário. A área `.obsidian` e caminhos fora do Vault continuam protegidos.

## Instruções e permissões

As instruções do servidor e do agente devem orientar o filtro proativo de aprendizados relevantes, a busca por duplicatas e a criação somente de notas confirmadas e de baixo risco. Instruções não substituem permissões do agente: a configuração do OpenCode precisa permitir explicitamente a ação de criação.

## Verificação

Foram executados `go test ./...`, `go vet ./...`, compilação do binário e validação da configuração JSON. A conexão MCP aparece como conectada. A permissão de criação deve ser confirmada numa sessão Plan do OpenCode.

## Preferência do usuário

O usuário espera que aprendizados duradouros e confirmados sejam registrados ou acrescentados proativamente no Obsidian durante a conversa, seguindo o fluxo de busca de duplicatas, leitura da nota existente e `memory_update` quando apropriado. Não espere que ele peça novamente; registre aprendizados, não a conversa inteira.

