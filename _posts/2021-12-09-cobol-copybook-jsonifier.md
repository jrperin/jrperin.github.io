---
layout: post
title: Python Vs. COBOL - Lib para Converter Arquivos Mainframe em Json
comments: true
categories:
    - Python
    - Cobol
    - Copybook
    - EBCDIC to ASCII
    - Data Migration
    - ETL
    - Mainframe
    - Data Migration
description: Criei a lib <b>Cobol Copybook Jsonifier</b> para converter arquivos Mainframe com padrão EBCDIC em JSON (ASCII).
image: /public/images/cobol-copybook-jsonifier_main2.png
---

![Datacenter](/public/images/cobol-copybook-jsonifier_main2.jpg)


-----

## Introdução

Vou tratar nesse artigo sobre as armadilhas e dificuldades para fazer a integração entre plataformas com diferentes **Encode de Dados**.

Focarei na transmissão de arquivos da plataforma **Mainframe** para plataforma **Cloud** e na **biblioteca que desenvolvi em python** para facilitar a conversão dos dados e que utiliza os **Copybooks Cobol** como _data squema_ para a formatação e conversão dos dados em JSON, diminuindo possíveis erros humanos de interpretação dos dados e desenvolvimento dos programas.


## O Problema

Para quem não tem familiaridade com a plataforma Mainframe pode estar pensando: Que negócio é esse de EBCDIC? 🤔

Pois é, o Mainframe utiliza um Code Page diferente para tratar os dados.

Nossos laptops, desktops, cloud etc, estão pautadas em sistemas operacionais Linux, Unix, Windows, macOS que utilizam codificação ASCII para apresentar e manipular os dados.

Essas diferenças acabam dificultando a integração entre essas plataformas.

Como exemplo, vemos na tabela abaixo parte das diferenças nos valores hexadecimais  entre os 2 padrões.


**Tabela 1:** Exemplo das diferenças entre EBCDIC e ASCII:

| Symbol | ASCII<br>Hex Value | EBCDIC<br>Hex Value | . | Symbol | ASCII<br>Hex Value | EBCDIC<br>Hex Value |
|:-----: | :-----: | :----: |:-:|:-----: | :-----: | :----: |
|    0   |    30   |   F0   | . |    i   |    69   |   89   | 
|    1   |    31   |   F1   | . |    j   |    6A   |   91   | 
|    2   |    32   |   F2   | . |    k   |    6B   |   92   | 
|    3   |    33   |   F3   | . |    l   |    6C   |   93   | 
|    4   |    34   |   F4   | . |    m   |    6D   |   94   | 
|    5   |    35   |   F5   | . |    n   |    6E   |   95   | 
|    6   |    36   |   F6   | . |    o   |    6F   |   96   | 
|    7   |    37   |   F7   | . |    p   |    70   |   97   | 
|    8   |    38   |   F8   | . |    q   |    71   |   98   | 
|    9   |    39   |   F9   | . |    r   |    72   |   99   | 
|    a   |    61   |   81   | . |    s   |    73   |   A2   | 
|    b   |    62   |   82   | . |    t   |    74   |   A3   | 
|    c   |    63   |   83   | . |    u   |    75   |   A4   | 
|    d   |    64   |   84   | . |    v   |    76   |   A5   | 
|    e   |    65   |   85   | . |    w   |    77   |   A6   | 
|    f   |    66   |   86   | . |    x   |    78   |   A7   | 
|    g   |    67   |   87   | . |    y   |    79   |   A8   | 
|    h   |    68   |   88   | . |    z   |    7A   |   A9   | 

**Fonte:** _[IBM - ASCII and EBCDIC character sets](https://www.ibm.com/docs/en/xl-fortran-aix/16.1.0?topic=appendix-ascii-ebcdic-character-sets)_


No Mainframe a forma mais comum de armazemamento de arquivos é em Flat Files, com tabanho fixo de registros e estrutura _(Fixo Blocado)_, podendo conter dados alfanuméricos ou nunéricos (binário)

Os programas que tratam esses arquivos no Mainframe geralmente são desenvolvidos em COBOL. 

O COBOL possui a entidade **CopyBook** para ser usada como "include" no programa e que tem o schema para divisão dos dados dentro do arquivo em campos interpretáveis.

**Figura 1:** Exemplo de Copybook Cobol

![imagem de exemplo cobybook cobol ](/public/images/cobol-copybook-example1.png)

**Fonte:** [IBM - Example COBOL copybook](https://www.ibm.com/docs/en/record-generator/3.0?topic=SSMQ4D_3.0.0/documentation/cobol_rcg_examplecopybook.html)


Aqui começam os problemas: 

* **O quê fazer quando queremos transferir arquivos entre as plataformas?** 🤔

As ferramentas de transferência de arquivos como FTP e o Connect Direct da IBM® dentre várias outras oferecem como opção default transmitir os arquivos no formato ASCII.

Isso é prático, mas só funciona direito quando os dados no arquivo Mainframe estão em formato "texto" (alfanumérico). 

Arquivos que possuem conteúdo "binário", como campos numéricos, se forem transmitidos com a opção ASCII serão corrompidos, pois a conversão para ASCII será aplicada em todo o conteúdo do arquivo.

Uma alternativa para esse problema é converter todo o conteúdo do arquivo para alfanumérico antes de transmitir, para que a ferramenta converta todo conteúdo Alfanumérico EBCDIC para Alfanumérico ASCII.

Outro problema é a formatação do conteúdo do arquivo que é constituído de um bloco contínuo de dados, que dificulta o parse pelos desenvolverdores.

Na Cloud, a preferência é por trabalhar de forma assíncrona, resiliente, reativa e não blocante.

Portanto, o ideal é dividir o conteúdo do arquivo em campos e o padrão mais amigável e cloudable para isso é converter o conteúdo no formato JSON para publicar em tópicos ou filas (AWS SNS/SQS, Kafka, RabbitMQ etc) e ser consumido por microsservicos ou functions (AWS lambda). Isso permitirá a resiliência e escalabilidade da plataforma.


**Temos aqui 2 grandes problemas:**

1. Transmissão do arquivo com conteúdo binário
1. Processar o arquivo para transformar ele em um conteúdo amigável para cloud.

**E temos outros problemas derivados desses:**

1. Na transmissão do arquivo:
    1. Converter o conteúdo do arquivo em alfanumérico no Mainframe para ser trasmitido em ASCII?
    1. Transmitir o arquivo como binário e fazer a interpretação do seu conteúdo na Cloud, convertendo o conteúdo e tratando os campos numéricos como binários?

1. No processamento do arquivo:

    1. Como reduzir o tempo gasto para o desenvolvimento do parsing do arquivo?
    1. Existe alguma forma de reutilizar o que já foi contruído no mainframe para agilizar o desenvolvimento?
    1. Como reduzir erros de parse do arquivo? 
    1. Como manter um contrato do layout do arquivo?
    1. Como mitigar a falta de familiaridade dos desenvolvedores com esses padrões do Mainframe? 


## A Solução

A alternativa para resolver a maior parte desses problemas seria **utilizar o próprio book cobol** como contrato para o arquivo. Ele também seria o insumo para que alguma LIB podesse interpretá-lo e fazer o parse do conteúdo do arquivo com base nele de forma automática.

Provavelmente alguém já deveria ter passado por situação parecida. Então fui procurar na internet o que existia para tratar esse problema.

Foi decidido que nossos processos de carga seriam feitos em Python, devido a afinidade com o processo de carga dos arquivos, então, procurei por LIBs em Python que pudessem ter esse propósito.

Encontrei algumas LIBs que chegaram perto da minha necessidade, mas elas estavam preparadas para tratar os aquivos já convertidos de EBCDIC para ASCII. Não tratavam os arquivos em sua forma original.

Com o conhecimento de Maiframe, Cobol, Python e Cloud que tenho, concluí que conseguiria criar uma LIB que tratasse o problema exatamente da forma que eu estava precisando. Fora o desafio de construir algo novo 😬. 

Durante as noites, após colocar minha filhinha para dormir, consegui fazer uma POC de uma LIB em Python com esse propósito.

Nasceu a LIB **[Cobol Jsonifier](https://github.com/jrperin/cobol-copybook.jsonifier)**.

![imagem da lib Cobol Jsonifier](/public/images/COBOL_JSONIFIER.svg)


**A lib tem de 2 métodos principais:**

1. A Criação do **Parser** que é feita com base no Book Cobol informado.
2. **Parse** do arquivo vindo do Mainframe com formato EBCDIC/ASCII e criação do JSON linha a linha


## Utilização

A documentação do projeto no Gitlab tem maiores informações de como utilizá-la, mas sua utilização é bem simples:

**Instalar a lib:**
``` shell
pip install coboljsonifier
```

**Usar a lib:**
``` python
import simplejson
from coboljsonifier.copybookextractor import CopybookExtractor
from coboljsonifier.parser import Parser
from coboljsonifier.config.parser_type_enum import ParseType

...

# Extracting copybook structure
dict_structure = CopybookExtractor(bookfname).dict_book_structure

# Building a Parser
parser = Parser(dict_structure, ParseType.BINARY_EBCDIC).build()

...

# Parsing the data
parser.parse(data)

# Getting the result (it is an dict type)
dictvalue = parser.value

# Showing the result as Json
print(simplejson.dumps(dictvalue))
```

**O Resultado será algo parecido com:**
``` json
{"DATA1-REGISTRY-TYPE": 2, "DATA1-COMPANY": 4, "DATA1-USER-ACCOUNT": "0040000000090001111", "DATA1-BIRTH-DATE": "1971-01-21", "DATA1-NAME": "JOHN ROBERT PERIN", "DATA1-CREDIT-LIMIT": 1001, "DATA1-LIMIT-USED": -1000.10, "DATA1-STATUS": [{"DATA1-STATUS-FLAG": "1"}, {"DATA1-STATUS-FLAG": "2"}, {"DATA1-STATUS-FLAG": "3"}, {"DATA1-STATUS-FLAG": "4"}], "FILLER-1": null}
```


<br>

## Saiba Mais

* Github do projeto: **[https://github.com/jrperin/cobol-copybook.jsonifier](https://github.com/jrperin/cobol-copybook.jsonifier)**.
* PyPi: **[https://pypi.org/project/coboljsonifier/](https://pypi.org/project/coboljsonifier/)**

<br>

## Conclusão

A utilização da lib **Cobol Jsonifier** deverá trazer ganhos de reutilização de código, automatização de processos, mitigação de possíveis problemas devido à falta de familiaridade com a plataforma Mainframe.

A lib permitirá maior facilidade de integração entre o Mainframe e a Cloud, seja para processos transacionais, ETLs para datalakes ou migração entre plataformas.

Por ser uma biblioteca em Python, liguagem que é bem difundida, deverá ser fácil utilizá-la em processos de ETL com ferramentas como AWS Glue, AWS lambdas e outros similares.

---

## NOTA!

O Blog ainda está sem a funcionalidade de comentários.

Críticas, sugestões, colaborar com o projeto etc, contate-me pelo **[Linkedin](https://www.linkedin.com/posts/jrperin_python-vs-cobol-lib-para-converter-arquivos-activity-6874699993261846528-C4qV)** que terei prazer em conversar. 🤘



















