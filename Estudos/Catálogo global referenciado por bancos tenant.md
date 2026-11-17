# Catálogo global referenciado por bancos tenant

## Pergunta
Como uma tabela de formas financeiras em bancos tenant pode apontar para um catálogo fiscal compartilhado no banco global?

## Explicação
O catálogo compartilhado pode residir no banco central, enquanto cada banco tenant armazena a chave do tipo fiscal selecionado pela forma financeira. O relacionamento ORM deve consultar a conexão central explicitamente, e a migration tenant pode declarar uma chave estrangeira para a tabela global quando ambas as conexões usam o mesmo servidor de banco e o SGBD permite referências entre databases.

Quando o código fiscal tem zeros à esquerda (por exemplo, 01), um campo inteiro armazena 1; a aplicação deve formatá-lo como texto com largura fixa ao apresentar ou enviar à integração fiscal, ou guardar o código como string.

## Conceitos principais
- Catálogo global como fonte de dados compartilhada.
- Dados de configuração específicos no tenant.
- Foreign key entre databases depende do SGBD/servidor e da arquitetura de conexões.
- Tipos/códigos fiscais podem exigir formatação textual.

## Revisão
1. Qual parte do dado é compartilhada e qual é específica da empresa?
2. O banco utilizado suporta foreign keys entre databases?
3. Como preservar zeros à esquerda em códigos fiscais?

## Conexão em migrations centrais e tenant
Migrations centrais do projeto seguem o padrão `Schema::create(...)` sem selecionar conexão explicitamente; a migration de `fiscal_configs` é um exemplo. Migrations em `database/migrations/tenant` rodam no contexto da conexão tenant. Portanto, use o padrão da migration e do comando que a executa: não adicione `Schema::connection(...)` a uma migration central quando a execução já usa a conexão central por convenção. Se migrations forem executadas com conexão padrão diferente, explicite a conexão adequada.


## Catálogo global em uma rota autenticada tenant
“Global” descreve onde os dados residem (banco central compartilhado), não exige que a rota fique fora dos grupos de autenticação tenant. Uma rota dentro de `auth:sanctum,empresa` pode consultar o catálogo central, desde que o model use a conexão central explicitamente. Esse grupo também inicializa o tenant para o restante da requisição.


## Seeder e rota do catálogo global
Um catálogo fiscal compartilhado pode ser mantido por migration/seeder centrais e lido por um model que usa explicitamente a conexão central. A rota pode permanecer autenticada no grupo da empresa; autenticação e localização dos dados são responsabilidades diferentes. Códigos inteiros como `1` devem ser serializados no formato fiscal `01` quando o consumidor exigir dois dígitos.


## Vínculo tenant e validação no catálogo central
Uma coluna FK no banco tenant pode referenciar uma chave natural do catálogo central, não necessariamente uma coluna chamada `id`; use tipos compatíveis e declare `references('tipo')`. No Eloquent, a forma financeira usa `belongsTo` com a coluna local e a chave proprietária do catálogo. A regra de validação `Exists(Model::class, 'tipo')` resolve a conexão do model, permitindo validar contra a conexão central. Durante a migração de dados legados, a coluna pode começar nullable, enquanto novas gravações já exigem uma classificação válida; o `NOT NULL` pode ser aplicado após o backfill.


## Inventário tenant para classificação manual
Para classificar tipos fiscais de formas já existentes, o inventário deve percorrer os bancos tenant (o banco central serve somente para enumerar as empresas) e exportar dois CSVs: resumo com nomes agrupados, ocorrências, empresas de origem e coluna para preencher `tipo_pagamento_nfe`; detalhado com empresa/tenant, banco, ID e nome de cada forma. A exportação é somente de leitura. Incluir registros soft-deleted é importante se a etapa futura aplicar `NOT NULL`, pois a restrição também alcançaria esses registros.


## Formas financeiras padrão em novos tenants
O `SetupFinanceiroDatabaseSeeder` do módulo Financeiro é chamado nos fluxos atuais de criação de empresa. O seeder de formas padrão deve ser incluído nele e usar `firstOrCreate` para evitar duplicatas sem sobrescrever customizações. Mapeamento implementado: Dinheiro 1, Cheque 2, Cartão de Crédito 3, Cartão de Débito 4, Crediário da loja 5, Boleto bancário 15, Depósito bancário 16, Transferência bancária 18, PIX estático 20, PIX dinâmico 17 e Outros 99. O seeder destina-se a novas empresas; não classifica automaticamente os registros legados.

