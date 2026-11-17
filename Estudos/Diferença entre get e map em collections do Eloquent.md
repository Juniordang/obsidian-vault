# Diferença entre get e map em collections do Eloquent

## Pergunta
Qual a diferença entre retornar `Model::query()->orderBy(...)->get()` e aplicar `map()` ao resultado?

## Explicação
`get()` executa a consulta e retorna uma `Illuminate\Database\Eloquent\Collection` contendo modelos Eloquent. Retornar essa coleção normalmente serializa os atributos dos modelos para JSON, respeitando casts, accessors/appends, atributos ocultos e relações carregadas. Métodos comuns do modelo, porém, não são incluídos automaticamente como campos JSON.

`map()` transforma cada elemento já obtido. Se o callback retorna arrays, o resultado é uma coleção de arrays, não de modelos. Isso permite escolher os campos expostos e adicionar valores derivados, como um código fiscal com zeros à esquerda (`1` -> `"01"`). Em contrapartida, a resposta deixa de serializar cada modelo Eloquent e não carrega automaticamente relações ou comportamentos do model.

A variável `$t = ...->get()` apenas guarda o resultado; se o controller retorna `$t`, o resultado é equivalente a retornar diretamente a expressão `get()`.

## Conceitos principais
- `get()` executa a query e entrega uma coleção de modelos.
- `map()` projeta/transforma os elementos da coleção.
- `map()` não altera o banco; faz uma transformação em memória.
- Accessors/appends ou API Resources são alternativas para expor campos derivados mantendo uma camada de apresentação definida.

## Revisão
1. Que tipo de itens a coleção contém antes e depois de um `map()` que retorna arrays?
2. Um método PHP do model aparece automaticamente na serialização JSON?
3. Qual abordagem usar quando a API deve expor um formato próprio?

## Casts versus atributos derivados
Um cast transforma um atributo persistido do model ao ler/escrever (por exemplo, `tipo` para inteiro ou enum); não é a melhor representação para um segundo campo derivado com formatação diferente. Para gerar `codigo_nfe` a partir de `tipo` sem alterar o valor numérico persistido, use um accessor (`Attribute::get`). Inclua `codigo_nfe` em `$appends` se quiser que apareça na serialização do model; caso a formatação seja específica de um endpoint, use Resource ou `map()`. Assim `tipo` continua sendo `1` e `codigo_nfe` pode ser `"01"`.


## Exemplos no projeto
O projeto contém casts customizados em `app/Casts/BCNumber.php` e `app/Casts/ColorCast.php`, ambos implementam `CastsAttributes`. Também usa accessors baseados em `Attribute::get`, como `tokenPeek()` em `Modules/Fiscal/app/Models/FiscalConfig.php`. Para um campo derivado de apresentação como código fiscal com zero à esquerda, o padrão de accessor é mais apropriado que alterar o cast do campo persistido.


## Appends e nome camelCase de accessor
`$appends` inclui atributos calculados na serialização para array/JSON; não cria coluna nem grava no banco. Com `$snakeAttributes` habilitado (padrão do Eloquent), um accessor chamado `tPag()` é indexado como `t_pag`, então `$appends = ['tpag']` não corresponde a ele. Para uma chave `tpag`, nomeie o método `tpag()` e use `['tpag']`. Se o contrato externo exige exatamente `tPag`, prefira Resource/`map()` para controlar a chave sem depender da conversão de nomes do Eloquent.

