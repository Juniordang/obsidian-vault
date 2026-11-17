# Como o Git funciona por dentro

O Git é um banco de objetos imutáveis identificado por hashes, junto com referências que apontam para commits.

- **Blob**: conteúdo de um arquivo.
- **Tree**: nomes, modos e referências aos blobs e subárvores, representando uma pasta.
- **Commit**: aponta para uma tree e registra metadados e commits pais.
- **Branch**: referência leve para um commit; criar um commit avança essa referência.
- **Index (staging area)**: prepara o conteúdo que fará parte do próximo commit.
- **Packfiles**: agrupam e comprimem objetos para economizar espaço.

O diretório `.git` guarda esses dados e referências do repositório. O Git não é o GitHub: Git é o sistema de controle de versão; GitHub é um serviço que hospeda repositórios Git.