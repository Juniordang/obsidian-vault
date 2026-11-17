# GitLab CI para projetos Laravel

## Conceitos da pipeline

- Um job é um bloco nomeado no `.gitlab-ci.yml`, como `pint` ou `testes`; `image`, `stage` e `script` são propriedades do job reconhecidas pelo GitLab. O nome do job é escolhido pelo projeto.
- Stages definem a ordem do fluxo: o GitLab só avança para o próximo stage depois que os jobs do stage anterior terminam com sucesso. Jobs dentro do mesmo stage podem executar em paralelo quando há runners compatíveis e capacidade disponível.
- Runners executam jobs elegíveis. A execução não deve ser entendida como uma fila FIFO global estrita; disponibilidade, tags, limites de concorrência e agendamento influenciam qual job é iniciado.
- `build` costuma preparar um artefato publicável, como assets compilados ou imagem de container. Instalar dependências não é, por si só, um build. Neste app de tarefas, os estilos estão inline no Blade, então `npm run build` não é necessário para a página atual; faz sentido adicionar build de Vite quando ela consumir assets compilados.

## Verificações do app

- Laravel Pint é o formatador PHP do ecossistema Laravel, não uma palavra-chave do GitLab. `vendor/bin/pint --test` verifica formatação sem modificar arquivos e pode ser usado em CI; `vendor/bin/pint` corrige a formatação localmente.
- `php artisan test --compact` executa os testes Laravel com saída compacta.
- No exemplo de pipeline, PHP 8.4 + SQLite em memória permite rodar os testes sem um serviço externo de banco; `pdo_sqlite` precisa estar habilitado.

## APP_KEY

- `APP_KEY` é a chave que Laravel usa para criptografia da aplicação e proteção de dados como cookies.
- Uma chave fixa de teste pode ser definida no job da pipeline, mas não deve ser reutilizada em produção. Em produção, gerar chave própria e armazená-la em variável CI/CD protegida, não no repositório.

## Leitura de configuração e falhas

- `variables`, `cache` e `before_script` são palavras-chave de configuração, não jobs. No nível global aplicam-se aos jobs por padrão; dentro de um job, seu escopo é local. Cada job executa os comandos do `before_script` antes do próprio `script`.
- A falha de um job não desfaz o push nem remove o commit. Por padrão, jobs de stages posteriores não iniciam após uma falha; políticas como `allow_failure` ou `needs` podem alterar o fluxo.
- Para diagnosticar logs, localizar o primeiro comando com falha e o resumo final (`FAIL`/`ERROR`); mensagens de preparação, avisos e instalação podem não ser a causa. O job só executou os testes se o comando de testes aparecer no log.
- Se o Pint estiver lento ou for necessário isolar um problema, apontar o caminho do arquivo é uma forma de reduzir o escopo, por exemplo `vendor/bin/pint --test app/Http/Controllers/TaskController.php`.

## Reexecutar uma pipeline

- Para testar a correção de código, faça commit e push; o push dispara uma nova pipeline. Repetir o job falho sem alterar o commit só repete o mesmo erro.
- Para repetir a execução do mesmo commit por falha transitória, use Build → Pipelines → abra a pipeline e escolha Retry nos jobs falhos, ou New pipeline para iniciar outra execução da branch/tag.


- Neste ambiente, `vendor/bin/pint --test` sem escopo não concluiu no tempo observado. O executável existe; uma checagem de arquivo com `php -d xdebug.mode=off vendor/bin/pint --test app/Http/Controllers/TaskController.php` terminou e identificou `not_operator_with_successor_space`. O aviso sobre JIT isoladamente não indica falha do Pint.


## Pacotes do container Linux

- Em imagem baseada em Debian, `apt-get update` atualiza os índices dos repositórios; não instala nem atualiza automaticamente todos os pacotes. Depois, `apt-get install -y ...` instala os pacotes pedidos e dependências (`-y` aceita as confirmações, útil em CI sem terminal interativo).
- Para o job PHP/SQLite: `curl` baixa o instalador do Composer, `git` e `unzip` auxiliam a obtenção/extração de dependências, e `libsqlite3-dev` fornece cabeçalhos para compilar `pdo_sqlite` com `docker-php-ext-install`. Os pacotes são instalados no container temporário do job.


## Contexto de aprendizagem do usuário

O usuário está desenvolvendo o ambiente de teste do sistema da empresa e quer aprender, passo a passo, a desenhar a pipeline GitLab CI/CD e seus scripts. Tratar o app `teste-gitlab` como exercício; antes de propor configuração final para o sistema da empresa, confirmar stack, dependências, serviços de teste, runner e estratégia segura de deploy para o ambiente de teste.


## Exemplo do repositório Mais Gerencia Auto Car

- A CI existente em `.gitlab-ci.yml` usa uma imagem PHP 8.4.3 Alpine (`containers/ci-laravel/Containerfile`) e chama `scripts/ci-test.sh`; esse script instala dependências e executa `parallel-lint`, mas não chama PHPUnit.
- `composer test:unit` seleciona `Unit` e `(Modules) Unit`; uma execução observada teve 181 testes passando e 21 falhando. As falhas incluíram chamadas HTTP não simuladas para hosts `.test`, incompatibilidades entre alguns testes e código, teste sensível a cache/tempo e setup de teste que insere usuário sem preencher `id_empresa`.
- Logo, não basta incluir os testes na pipeline: primeiro é preciso tornar os testes previsíveis e independentes de serviços externos, resolver o setup dependente do banco e separar falhas de código/configuração. A mensagem de resumo do Composer (código 2) não explica a causa; usar o trecho anterior com o primeiro erro real.
- O repositório declara MariaDB 11.4 no Compose, mas `.env.testing.example` configura conexão `mysql` e `DB_DATABASE=auto_car_tester`. Confirmar que o banco selecionado é descartável antes de repetir testes que gravam dados.
- Ao orientar iniciantes, explicar o comando e o que sua saída significa, com cautela explícita antes de executar testes que possam gravar em banco; nunca pedir credenciais ou valores de segredos.

## Quando criar testes para novas mudanças

- Neste projeto, a orientação é acompanhar mudanças de comportamento com testes PHPUnit na suíte adequada (`AGENTs.md`). Isso não significa criar obrigatoriamente um arquivo de teste para cada módulo novo ou cada alteração mecânica: cobrir o comportamento/risco relevante, preferindo testes unitários para lógica isolada e feature/integration para fluxos que cruzam Laravel, HTTP, banco ou tenancy. Em mudança sem comportamento observável, pode não haver teste novo; justificar e manter as verificações existentes.
- Para módulo novo, planejar testes junto com os comportamentos e contratos principais do módulo; garantir que a suíte seja determinística, não dependa de chamadas reais a serviços externos e use banco descartável quando necessário. O comando `composer test` roda a suíte completa; `composer test:unit` cobre unitários, mas no checkout investigado alguns desses testes falharam.

## Exemplo externo de pipeline Integrador (referência, não configuração do Auto Car)

- O diretório `integrador-pipelines` contém `.gitlab-ci.yml` com stages `images`, `test`, `build`, `deploy`. A imagem PHP/Node pode ser reconstruída com Docker-in-Docker; teste usa `scripts/ci-test.sh`; build gera tarball versionado e transfere via SCP; deploy cria release remota e alterna por script sudo. O fluxo pausa/resume filas e scheduler, roda migrations e reinicia workers.
- `scripts/ci-test.sh` instala Composer, roda `artisan optimize`, `npm install`, `npm run build` e `composer test:ci`. A imagem `ci-laravel-node` adiciona Node/npm e `pdo_mysql`, mantendo PHP 8.4.3. Isso é um exemplo mais completo, mas depende do projeto Integrador e não deve ser copiado automaticamente ao Auto Car.
- Atenção pedagógica: o job de teste tem `allow_failure: true`, então falhas de teste não bloqueiam stages seguintes; deve-se confirmar que isso é uma decisão deliberada antes de adotá-lo. Build de imagens também usa `allow_failure: true`, possibilitando seguir com imagem previamente publicada. Docker-in-Docker pressupõe runner compatível. `ci-build.sh` busca `.env` do servidor via SCP durante build, comportamento que não se deve transportar sem avaliar exposição de segredos e reprodutibilidade.
- A pipeline publica tags de imagem `latest` e `$CI_COMMIT_SHORT_SHA`, com cache de build; teste e build são acionados em pushes, build/deploy limitados à branch `test`. O deploy consome artefato dotenv `version.env` do build.
