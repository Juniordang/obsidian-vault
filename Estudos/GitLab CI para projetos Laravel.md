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

- Regra prática para artefatos: use quando um job produz arquivos que outro job posterior precisa consumir, ou quando se quer guardar/baixar saídas importantes (pacote de deploy, assets, relatório de testes/cobertura, dotenv com versão). Não use artefato para dependências que só aceleram repetição — isso é cache — nem para segredos. Se nenhum job posterior ou usuário precisa do resultado e não há valor de auditoria, provavelmente não é necessário publicar como artefato.

- `workflow:auto_cancel:on_new_commit: interruptible` habilita cancelamento de pipelines redundantes em novos commits, desde que os jobs em execução sejam interruptible; jobs marcados `interruptible: false` não são cancelados. Útil para poupar runner em branches de desenvolvimento, mas avaliar antes de aplicar a deploys/ações com efeitos que não devam ser interrompidos. Referência consultada: documentação oficial GitLab CI/CD YAML e pipeline settings; Context7 indisponível por quota.

- No GitLab CI, `image:` seleciona a imagem de container (ambiente de execução) em que o job roda; ela fornece sistema base, linguagem/runtime e ferramentas disponíveis. Ex.: a imagem Laravel CI do Auto Car inclui PHP, Composer e extensões usadas no lint; o Integrador usa imagem com Node/npm para construir assets. A imagem do job não é a imagem do artefato da aplicação automaticamente. `services:` são containers auxiliares, como MariaDB/Redis, quando o job precisa deles. A execução por containers depende do executor/configuração do runner.

- Distinção importante: job com `image: docker:24-cli` + serviço `docker:24-dind` usa o cliente Docker do job para comandar um daemon Docker auxiliar e construir/publicar outra imagem. Um template `.build_ci_image` não prepara diretamente o ambiente do job que o executa; fabrica uma imagem CI/deploy customizada descrita no `Containerfile`, que outros jobs usarão depois. `extends` instancia o template com `IMAGE` e `CONTEXT`. O sucesso tolerado (`allow_failure: true`) significa que consumidores podem continuar com imagens antigas; considerar o risco de tag `latest` desatualizada.

- GitLab Container Registry é o repositório de imagens de container integrado ao projeto/grupo GitLab. `docker push` publica tags lá e jobs/runner fazem `docker pull` ou referenciam a imagem em `image:`; requer autenticação/permissões e Registry habilitado. Diferente de artefatos CI, que são outputs de jobs guardados/transferidos pela pipeline. Fonte oficial GitLab: user/packages/container_registry e build_and_push_images; Context7 indisponível por limite mensal.

- Na interface web do GitLab, abrir o projeto e navegar na barra lateral em Deploy > Container Registry para ver imagens e tags (por exemplo `latest` e SHA do commit). Se não aparecer, pode estar desabilitado, sem imagens publicadas ou faltarem permissões. Fonte: documentação oficial de Container Registry.

- Registry não é atualizado automaticamente por existir uma pipeline: só muda se um job executar `docker push` com sucesso. No exemplo Integrador, jobs de build de imagem têm regra automática quando Containerfile muda na branch padrão e são manuais nos demais casos; não há push em todo pipeline por padrão. Re-push da tag `latest` move essa tag para a imagem nova; tag SHA permanece associada àquele commit (repetição do mesmo commit pode atualizar a mesma tag).

- Em jobs GitLab, `variables:` define variáveis de ambiente do job; `IMAGE` e `CONTEXT` do job concreto são herdadas pelo template via `extends` e interpoladas nos comandos `script` pelo shell. `CI_COMMIT_BRANCH`, `CI_DEFAULT_BRANCH` e `CI_REGISTRY_IMAGE` são predefined variables preenchidas pelo GitLab, não valores guardados no repositório. `rules:changes` restringe uma regra a pipelines com alterações nos caminhos listados, comparados ao commit/branch base conforme a origem da pipeline. As regras são avaliadas em ordem e vence a primeira correspondida; fallback `- when: manual` cria job manual quando a regra anterior não corresponder (não significa que o job some). `ci-php` é apenas identificador escolhido para imagem/container, não um recurso especial do GitLab.

- No job de teste de exemplo: `rules: if CI_PIPELINE_SOURCE == 'push'` restringe execução a pipelines originadas de push; `image` escolhe ambiente runtime; `sh -x` imprime comandos do script conforme executa (evitar expor segredos via tracing). `artifacts: when: on_failure` guarda arquivos apenas se job falhar, `paths` seleciona logs, `expire_in` define retenção e `access` limita visibilidade conforme permissão/projeto. `cache` com chave `vendor` reutiliza dependências para acelerar; não é transferência garantida de resultado como artefato e chave genérica pode ser melhorada com lockfile. Fonte oficial GitLab YAML/artifacts/cache; Context7 indisponível por quota.

- No exemplo `ci-test.sh` do Integrador: shebang seleciona Bash; `PATH` acrescenta `vendor/bin` para invocar ferramentas Composer locais; `set -e` encerra no primeiro comando com falha e `-x` imprime comandos (cuidado com segredos nos logs). `composer install --no-progress` instala dependências sem barra de progresso; `php artisan optimize` prepara caches Laravel; `npm install` instala dependências JS e `npm run build` executa o script build de package.json. `composer test:ci` é script customizado definido no composer.json do projeto Integrador e seu conteúdo precisa ser conferido ali. Context7 indisponível; descrição baseada no script e convenções de shell/Composer.

- No job `build integrador...`: só roda em pipeline originada de push na branch `test`. Usa imagem CI com PHP/Node; prepara ssh-agent/chave para que `ci-build.sh` possa fazer SCP. O build gera `version.env` e declara-o como `artifacts:reports:dotenv`, tornando `VERSION` disponível como variável em jobs posteriores; caches `vendor` e `node_modules` para acelerar. No script examinado, o pacote tar.gz é copiado por SCP ao servidor durante o próprio build (não é esse pacote que o bloco `artifacts:reports:dotenv` transfere). `ci-build.sh` também baixa `.env` remoto e inclui arquivos da pasta no tar; revisar cuidadosamente segredos/acesso antes de reproduzir essa estratégia em outro sistema.

- SSH prep no job: `ssh-agent -s` inicia o agente e imprime exports do socket/PID; `eval "$(...)"` aplica esses exports ao shell atual. `chmod 400 "$SSH_KEY_RUNNER"` restringe a chave privada a leitura pelo dono; a variável GitLab pode ser do tipo File, caso em que contém o caminho temporário do arquivo. `ssh-add` carrega a chave no agente para autenticar chamadas SSH/SCP seguintes sem colocar a chave no comando. `&&` só executa o próximo passo se o anterior teve sucesso. A chave deve ser protegida e nunca exibida, copiada para artefatos ou impressa em logs; o agente vive no job.

## Snapshot da pasta pipeline-renave (18/11/2026)

- A pasta fornecida reúne configurações dos projetos `maisgerenciaautocar.api` e `maisgerencia.renave`; `LEIA-ME.txt` afirma que segredos/envs foram excluídos. As duas pipelines são distintas: API tem stages `test, deploy` e teste atual só Composer install + parallel-lint; RENAVE tem `images, test, build, deploy`, com imagem PHP customizada, PHPUnit e deploy em `main`.
- Pipeline RENAVE: `.build_ci_image` constrói e publica `ci-php`; teste roda em push com essa imagem e guarda logs no fracasso/cache vendor; build e deploy são apenas push para `main`, passando `version.env` dotenv ao deploy. Há scripts E2E no diretório, mas nenhum job E2E aparece no YAML lido; `scripts/ci-e2e.sh` existe porém não é referenciado por job.
- `maisgerencia.renave/scripts/ci-test.sh`: Composer install, copia `.env.example`, gera APP_KEY, gera docs Scribe e roda `composer test`. `Containerfile` usa PHP 8.4 CLI Bookworm, Composer 2, extensões pdo_sqlite, pdo_mysql e zip.
- `scripts/e2e.sh` sobe app e fakes locais do RENAVE e RENAVE Total em portas livres, aguarda banco MySQL, roda migrations e `composer test:e2e`, e tem cleanup por trap. `scripts/ci-e2e.sh` instala deps/config e chama E2E. Não há job YAML chamando esse script no snapshot.
- `maisgerenciaautocar.api/scripts/e2e-renave.sh` limita endpoints E2E a localhost, sobe fake hub e Laravel e chama phpunit.e2e.xml. `e2e-full.sh` orquestra ERP+hub+fakes+worker, inicializa hub em SQLite e ERP em MySQL, mas chama `migrate:fresh --force` no DB ERP configurável (default `maisgerencia_e2e`); esse comando apaga/recria tabelas. Antes de executar, confirmar com rigor que host/db são descartáveis; o script não tem allowlist de host.
- Deploy RENAVE: build baixa `.env` remoto por SCP, empacota árvore em tar.gz e envia diretamente ao servidor; deploy remoto extrai release, otimiza, valida config, storage:link, migra e delega ativação ao script sudo. `version.env` é o artefato dotenv, não o tarball. Em deployment e logs com `set -x`, revisar cuidadosamente exposição de configuração e credenciais.
- Para API Auto Car, deploy ainda usa `git pull`, Composer, migrations, caches, Scribe em não-produção e restart da queue. Scripts de permissão aplicam owner/group/chmod; systemd instala e habilita serviço de queue. Essas operações só descrevem conteúdo, não foram executadas.

- Correção importante sobre fallback `rules: - when: manual`: quando `when: manual` está dentro de `rules`, o default de `allow_failure` é `false`, portanto o job manual é blocking e pode deixar pipeline parada nesse stage até ser executado (salvo configuração explícita `allow_failure: true`). Isso deve ser considerado no job `build:ci-php`, cuja segunda regra é manual para demais casos. Fonte oficial GitLab job_rules/job_control.

- Escopo confirmado pelo usuário para as próximas explicações: focar somente no diretório `/mnt/c/Users/dange/Downloads/pipeline-renave/pipeline-renave/maisgerencia.renave`; não misturar os arquivos do projeto irmão `maisgerenciaautocar.api`, a menos que o usuário peça comparação.

- Preferência de aprendizagem explicitada em 19/11/2026: explicar o diretório `maisgerencia.renave` arquivo por arquivo e linha por linha, com exemplos e analogias; esclarecer conceitos básicos como `workflow` antes de seguir. Avançar sequencialmente e não sobrecarregar com resumo genérico.

- Explicação pedagógica do job RENAVE `test`: stage test, interruptible true, regra para `CI_PIPELINE_SOURCE == push`, usa `$CI_REGISTRY_IMAGE/ci-php`, chama `sh -x scripts/ci-test.sh`, coleta `storage/logs` em falha por 7 dias e mantém `vendor` como cache. O script copia `.env.example`, gera chave, gera docs Scribe e executa `composer test`; a expansão exata de `composer test` depende do composer.json do app original, não incluído no pacote de pipeline analisado.

- Pedido a avaliar para pipeline ERP: quando o job de teste falhar, permitir decisão manual para prosseguir ao job seguinte. Conceitualmente `allow_failure: true` deixa a pipeline avançar apesar da falha, mas faz isso automaticamente; para exigir confirmação, o job seguinte pode ser manual/bloqueante. Esse gate tende a aparecer também quando testes passam; não presumir que YAML `rules` consiga ramificar pelo status anterior. Confirmar se a escolha manual deve existir só após falha ou sempre antes de deploy. Não implementado (Plan mode).

- Limite importante: artefato `artifacts:reports:dotenv` pode entregar variáveis a jobs posteriores para uso nos scripts, mas não pode alimentar `rules`, porque o GitLab avalia `rules` ao criar a pipeline, antes dos jobs produzirem artefatos. Para decisão manual apenas após falha, permitir falha + gate manual simples deixa o gate manual também em sucesso; condição dinâmica pode exigir child pipeline com YAML gerado após conhecer resultado, uma solução bem mais complexa.

- Complexidade/custo de pipeline filha dinâmica para selecionar deploy manual só após falha: complexidade média-alta versus gate manual sempre. Exige capturar resultado do teste de forma confiável, gerar e validar YAML, publicar artefato de configuração, trigger de child pipeline e propagar resultado/status. Custo runner incremental pequeno (job gerador e eventuais jobs adicionais); os jobs reais de build/deploy seguem consumindo o tempo normal. Em runner próprio o custo é infraestrutura/tempo; GitLab SaaS pode consumir minutos conforme jobs executados e plano. Child pipelines dinâmicas são suportadas oficialmente.

- `CI_JOB_STATUS` é status do job atual, não permite consultar diretamente o status de outro job. É útil especialmente em `after_script` (valores como `success`, `failed`, `canceled`). Para compartilhar resultado do teste com job posterior, gravar o resultado em artefato/dotenv; dotenv pode ser consumido por script subsequente, mas não por `rules`. Fonte Context7 `/websites/gitlab_ci`, documentação GitLab CI/CD.

- É possível copiar `CI_JOB_STATUS` para variável própria no `after_script` do job atual e publicar arquivo com formato `TEST_JOB_STATUS=...` em `artifacts:reports:dotenv`; job posterior recebe a variável e pode ramificar comandos dentro de `script`. Isso ainda não transforma o job em manual nem alimenta `rules`; comportamento de agendamento precisa estar definido ao criar a pipeline.
