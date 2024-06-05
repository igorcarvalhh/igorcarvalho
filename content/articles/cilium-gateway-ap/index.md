---
title: "ERROR: user is not in not in sudoers file"
date: 2024-05-15
draft: false
description: "a description"
tags: ["api", "tag", "http"]
views: 52
---

Normalmente, utilizo máquinas virtuais para desenvolver meus projetos pessoais e manter a organização deles. Essa abordagem permite que eu mantenha um ambiente isolado, com apenas as dependências necessárias para cada projeto específico.

Eventualmente, ao criar uma nova máquina virtual para um novo projeto, ao tentar executar algum comando com privilégios de administrador (sudo), deparo-me com o seguinte erro:

```sh
igor@ubuntu:~$ sudo ls
[sudo] password for igor:
igor is not in the sudoers file. This incident will be reported.
```

Esse erro indica que o usuário igor não possui permissões para utilizar o comando sudo. Para resolver esse problema, é necessário adicionar o usuário igor ao arquivo de configuração do sudoers, que especifica quais usuários possuem permissões de superusuário.

## Solução

Primeiramente, faça login como root:

```sh
su -
```

Em seguida, abra o arquivo /etc/sudoers com um editor de texto. Aqui, utilizamos o nano:

```sh
nano /etc/sudoers
```

Adicione a seguinte linha ao arquivo para conceder permissões sudo ao usuário igor:

```sh
igor    ALL=(ALL:ALL) ALL
```

Salve e feche o arquivo. No nano, você pode fazer isso pressionando CTRL + O para salvar e CTRL + X para sair.

A linha adicionada ao arquivo sudoers assegura que o usuário igor tenha permissões sudo completas, permitindo-lhe executar qualquer comando como qualquer usuário, inclusive o root.


## Considerações Finais
Após realizar as alterações no arquivo sudoers, você deve ser capaz de executar comandos com sudo sem enfrentar problemas.

Adicionar um usuário ao arquivo sudoers é uma tarefa crítica e deve ser realizada com cautela. Sempre verifique se a sintaxe no arquivo sudoers está correta para evitar problemas de configuração que podem bloquear o acesso root ao sistema. Recomenda-se utilizar o comando visudo em vez de editar diretamente o arquivo sudoers, pois ele realiza verificações de sintaxe antes de salvar as mudanças, garantindo a integridade do arquivo.

Com essas orientações, você será capaz de resolver o erro "usuário não está no arquivo sudoers" e continuar desenvolvendo seus projetos em um ambiente controlado e organizado.
