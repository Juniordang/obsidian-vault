# Último acesso da empresa

Decisão confirmada: para identificar há quanto tempo uma empresa contratante não utiliza o sistema, registrar apenas o último acesso consolidado da empresa em um campo `ultimo_acesso_em` na tabela central `empresas`. Atualizar quando qualquer usuário da empresa utilizar o sistema, com limitação de frequência (por exemplo, uma vez a cada cinco minutos por empresa) para evitar gravações excessivas. Não é requisito manter o último acesso individual dos usuários nem identificar qual usuário gerou o registro. Expor a data na listagem administrativa de empresas para exibir a inatividade; `null` representa ausência de registro histórico.


Regra adicional confirmada: acessos realizados por personificação não devem atualizar `ultimo_acesso_em`; a personificação não conta como uso real da empresa pelo próprio usuário.


Atualização da decisão: também será armazenado `ultimo_acesso_usuario_id` em `empresas`, apontando para o usuário associado ao último acesso registrado. Com a gravação limitada a uma vez a cada cinco minutos por empresa, esse usuário pode não ser quem fez a requisição mais recente dentro do intervalo; se a identificação exata for necessária, será preciso atualizar com maior frequência.


Decisão final de implementação: usar o login bem-sucedido como evento de acesso da empresa. Atualizar `ultimo_acesso_em` e `ultimo_acesso_usuario_id` somente após autenticação bem-sucedida; não atualizar em cada requisição autenticada, nem em personificação. Assim, os campos representam o último login, não a atividade mais recente durante uma sessão.


Escopo confirmado para a implementação atual: somente backend/API; alterações na interface do Admin ficam fora desta tarefa.


Ao registrar login bem-sucedido, atualizar a empresa via Eloquent para que `updated_at` também avance junto com `ultimo_acesso_em` e `ultimo_acesso_usuario_id`; a empresa está sendo modificada.


Implementação do timestamp deve usar `DateContext::ensureNowLocal()` para obter o horário atual no fuso do cliente, em vez de `now()` diretamente.
