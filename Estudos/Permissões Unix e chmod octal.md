# Permissões Unix e chmod octal

As permissões Unix são frequentemente expressas em octal. Cada classe (dono, grupo, outros) tem três bits: leitura (`r` = 4), escrita (`w` = 2) e execução (`x` = 1). Somam-se os valores dos bits habilitados para obter um dígito de 0 a 7.

| Dígito | Bits `rwx` | Permissões |
|---:|:---:|---|
| 0 | 000 | nenhuma |
| 1 | 001 | execução |
| 2 | 010 | escrita |
| 3 | 011 | escrita e execução |
| 4 | 100 | leitura |
| 5 | 101 | leitura e execução |
| 6 | 110 | leitura e escrita |
| 7 | 111 | leitura, escrita e execução |

Os três dígitos de `chmod ABC` correspondem, nessa ordem, ao dono/usuário (`u`), grupo (`g`) e outros (`o`).

## Exemplo: `chmod 467 arquivo.txt`

- Dono: `4` = leitura.
- Grupo: `6` = leitura + escrita.
- Outros: `7` = leitura + escrita + execução.

Portanto, o comando define permissões `r--rw-rwx` para `arquivo.txt`. Atenção: conceder escrita e execução a todos os outros usuários (`7`) pode ser inseguro; use somente se isso for intencional.
