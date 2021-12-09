---
layout: post
title: Docker - Acessar Banco de Dados na Máquina Host
comments: true
categories: 
    - Docker
    - MariaDB
    - DB fora do Cluster
    - Ambiente de Teste
description: Acessar um banco de dados externo ao ambiente do Docker tem suas pegadinhas. Aqui tem dicas para facilitar essa tarefa.
image: https://miro.medium.com/max/700/1*rG6GnMOz0nlRUUU1Va7aTA.jpeg
---

![Container House](https://miro.medium.com/max/700/1*rG6GnMOz0nlRUUU1Va7aTA.jpeg)

[**Acesse meu Medium e veja versão em Inglês**](https://medium.com/@jrperin1975/docker-accessing-host-database-49e1ff92a40)

-----

## Introdução

Atualmente ao desenvolver uma aplicação, **é impossível** não pensar em embarcá-la em um **contêiner**.

A conteinerização das nossas aplicações traz ganhos como padronização, portabilidade, automatização que facilitam em muito o deploy.
Mas uma questão sempre fica rondando, até onde devemos levar para contêineres?

Na minha opinião estruturas de bancos de dados como Mysql, Mariadb, MongoDb quando em produção, deveriam ter recursos mais dedicados a eles, mais “bare metal”. 

Não estou considerando recursos de bancos de dados gerenciados pelos provedores de cloud como AWS, Google, Azure. Estou considerando que a gestão dos recursos dos bancos de dados é feita por você.

Claro que se estivermos falando em ambiente de desenvolvimento, o cenário é outro.

Colocar os recursos de banco de dados em contêineres, facilita muito para o desenvolvedor ter um um ambiente rodando. Sem que ele tenha que se preocupar com a infraestrutura.

Para que um uma aplicação docker possa acessar o “mundo externo” é preciso informar para ela o IP do host (do hospedeiro).

No `docker-compose` isso é feito pelo membro `extra_hosts` ou via linha de comando com o parâmetro `--add-hosts`, que adiciona o endereço ao DNS interno do Docker.

Nesse _case_, vou focar no **docker-compose**.

**Resumo dos passos que serão dados:**

1. Instalar o banco de dados Mariadb em um computador com Ubuntu que será a máquina hospedeira.

1. Criar um contêiner com ferramentas para acessar o banco de dados Mariadb do hospedeiro (Dockerfile e docker-compose.yml).

1. Alterar as permissões do usuário root do banco de dados para permitir acesso do **IP docker** atribuído ao host.

1. Alterar o **bind-address** do banco de dados para permitir acesso através do **IP docker do host**.

1. Validar, fazendo o teste de conexão do contêiner para o host.

## Execução

### 1. Instalar MariaDB no hospedeiro

``` shell
sudo apt update
sudo apt install -y mariadb-server
```

### 2. Criar contêiner para acessar o DB no host

Criar o arquivo **Dockerfile**:

``` shell
FROM ubuntu
RUN apt update && apt install -y iputils-ping net-tools telnet vim mysql-client
```

Criar o arquivo **docker-compose.yml:**

``` yaml
version: "3.9"
services:
    ubuntu:
        build:
            context: ./
            dockerfile: Dockerfile
    extra_hosts:
        - "database:172.17.0.1"
```

O que permite acessar o host é a sessão **extra_hosts** no docker-compose.

Algumas abordagens sugerem para colocar o IP externo para **database**, mas essa abordagem pode ser perigosa, pois o IP do servidor ficará exposto para a internet.

Outra abordagem diz que as versões mais recentes do docker permitem usar o DNS **host.docker.internal** para acessar o IP da interface de network docker do host, mas estou usando a versão 19.03.8 do docker no linux ubuntu e não foi possível acessá-lo.

Por esse motivo, optei por **colocar o número de IP diretamente (172.17.0.1)**. Pelo que pude constatar o número de IP é sempre o mesmo nas instalações (_testei em 2 computadores direfentes…_).

### 3. Permissões do usuário root no Banco de Dados
Antes de fazer a alteração do bind-address (_no step seguinte…_) é necessário dar privilégio para o usuário root acessar através do IP **172.17.0.1**.

**Nota:** Uma instalação nova do Mariadb **não tem senha** para o usuário root. Caso você tenha definido uma senha, use o parâmetro **‘-p’** para que seja solicitada a senha no login.

No exemplo abaixo, vamos aproveitar e definir uma senha para o usuário root na rede **172.17.0.1**.

``` shell
mysql -u root
mysql> grant all privileges on *.* to ‘root’@’172.17.0.1' identified by ‘my_password’;
mysql> flush privileges;
```

Caso também queira definir a senha para o usuário root na rede localhost (127.0.0.1), repita o comando acima trocando `root`@`172.17.0.1` por `root`@`127.0.0.1`.

Caso deseje liberar o acesso ao usuario root para qualquer IP, troque o número do IP `172.17.0.1` por `%`que é o wild card para **“tudo”**, mas isso não é uma boa idéia, pois pode liberar acesso demais e colocar o sistema em risco.

### 4. Alterar o bind-address do Banco de Dados

Agora pode ser alterado o endereço do banco de dados para o da interface do docker (172.17.0.1).

Para isso, vamos editar o arquivo **.cnf** que pode variar de local dependendo da instalação do banco de dados e do sistema operacional.

``` shell
vim /etc/mysql/mysql.conf.d/mysqld.cnf        <-- Mysql
vim /etc/mysql/mariadb.conf.d/50-server.cnf   <-- Mariadb
vim /etc/my.cnf <-- Red Hat / CentOS`
# Dentro do arquivo, procurar por:
bind-address = 127.0.0.1
# E mudar para o gateway do doker:
bind-address = 172.17.0.1
# Reiniciar o Mariadb (ou Mysql):
sudo systemctl restart mysqld
```

### 5. Validar

Agora vamos validar nossas configurações / ambiente.

Primeiramente vamos preparar o ambiente.

``` shell
docker-compose up --build
```

Após esse comando o docker-compose faz o build, sobe e encerra. Isso é porque a nossa imagem Dockerfile não tem um ENTRYPOIT definido, o que é esperado para a imagem Ubuntu que usamos.

Vamos validar o a conexão:

``` shell
docker-compose run ubuntu bash
```

Com esse comando, abrimos a shell da imagem Ubuntu no docker.

Vamos conectar no Mariadb que está no host.

``` shell
mysql -h database -u root -p
```

A senha que você deverá digitar é a mesma que foi informada no passo 3 em `'my_password'`.

Se tudo ocorreu bem, estamos dentro do banco de dados e podemos validar com o comando:

``` shell
show databases;
```

Se conseguir ver os DBs, a validação foi bem sucedida!

![Gif Sr. Miyagi acenando positivamente](/public/images/sr_myiagi.gif)
