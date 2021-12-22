---
layout: post
title: Processando Arquivos Grandes em Paralelo - POC | AWS S3
comments: true
categories:
    - Python
    - Data Migration
    - ETL
    - Mainframe
    - Arquivo posicional
description: POC sobre o processamento em paralelo de grandes arquivos recebidos em buckets AWS S3 usando o recurso de leitura por range. 
image: /public/images/poc-huge-files/navio.jpg
---

![imagem capa](/public/images/poc-huge-files/navio.jpg)

## Introdução

No [post anterior](https://blog.jrperin.com/cobol-copybook-jsonifier/), comentei sobre a [lib em Python](https://blog.jrperin.com/cobol-copybook-jsonifier/) que fiz para converter arquivos do Mainframe em JSON. 

Continuando no tema de integração entre plataformas, vou falar sobre uma abordagem interessante para tratar arquivos grandes *(Big Files / Huge Files)*. 

Quando há transferência de dados entre plataformas existem vários desafios que podem ser encontrados e um deles pode ser a janela de tempo para processamento dos dados recebidos.

Sistemas legados, muitas vezes possuem processamentos em lote *(processamentos batch)* e muitas das integrações entre plataformas precisam ter o retorno do processamento para dar sequência nos fluxos e em muitos desses casos há limite máximo de tempo para o retorno.

Nesse artigo, fiz uma POC usando AWS Lambda, AWS S3 e um pouco de python para escalonamento horizontal de processamento de arquivos grandes com custo relativamente baixo e sem uso de ferramentas mais caras como AWS Glue, por exemplo.

## O Problema

* Um arquivo grande é disponibilizado em um bucket S3 para ser processado.

* Esse arquivo contém registros das compras que clientes fizeram em um e-commerce (_fictício_).

* Como melhorar a velocidade de processamento do arquivo? 

* **Nota:** O Layout do arquivo é sempre fixo e os campos possuem sempre o mesmo tamanho, conforme **tabela 1**.

    > _Arquivos com dados posicionais de tamanho fixo são comumente usados em sistemas IBM® Mainframe._


## A Solução

Como o arquivo possui layout fixo, isso permite uma abordagem onde pode-se usar um recurso interessante que o AWS S3 tem que é a **leitura por range**.

A leitura por range, permite que hajam vários processos em paralelo lendo o mesmo aquivo.

1. Será criado um lambda, que vou chamar de **"Strategist Lambda"**, que será acionado _("SNS event trigger")_ sempre que houver a criação de um arquivo no bucket _(file put)_.
    * Esse lambda será o responsável pela estratégia de particionamento do arquivo.
    * Uma variável de ambiente vai informar em quantas partes o arquivo deverá ser particionado.
    * Com base no tamanho do arquivo e na quantidade de partições cadastradas, o **Strategist Lambda** vai acionar outros lambdas (**Workers Lambdas**) que farão a leitura por range e salvarão os arquivos particionados.

    * Cada **Worker Lambda** será responsável por uma partição, permitindo paralelismo de processamento.

        > **Exemplo de leitura por range - AWS CLI:**
        >
        > **`aws s3api get-object --bucket bucket_name --key folder1/my_file.txt --range bytes=8888-9999 my_parset_file.txt`**

1. Cada **Worker Lamda** vai receber como evento o nome do arquivo e o range de bytes que deverá salvar na respectiva partição.
    
    * Como os Workers receberão ranges diferentes, cada processo vai ler um pedaço do arquivo origem em paralelo.

    * O case aborda sobre salvar partições do arquivo, mas a mesma abordagem poderá ser adaptada para permitir o uso de outras técnicas/ferramentas como: Gravar filas (SQS), gravar eventos (SNS), salvar em banco de dados, fazer upload via FTP, dentre outros.

**Figura 1:** Diagrama da Proposta de processamento do arquivo com múltiplas instâncias de Lambdas.

![Diagrama da Proposta de processamenot do arquivo com múltiplas instâncias de Lambdas](/public/images/poc-huge-files/s3-big-file-parallel-processing.png)


# Montando o LAB

**Obs.:** Os programas em Python e a Prova de Conceito foram testados na minha conta pessoal da AWS.


## Criando a massa - "The Huge File"

Como esse é um ambiente teórico, há necessidade de criar massa para fazer a POC (_Prove Of Concept_).

A massa foi criada com base no layout da **tabela 1**.

Para a massa ficar mais parecida com uma massa real, baixou-se dados públicos do IBGE uma lista com 16.000 nomes Brasileiros, conforme **figura 2**.

**Figura 2:** IBGE Dados Públicos - Google Big Query.

![Dados IBGE - Google Big Query](/public/images/poc-huge-files/ibge_dados_google_big_query.png)


O programa em python **`huge-file-generator.py`** cria um arquivo com mais de 12Gb de dados seguindo o layout apresentado na **tabela 1**.

**Tabela 1:** Layout fictício de compras no e-commerce.

| CAMPO               | TAMANHO   | FORMATO               |
| :----               | :-----:   | :------               |
| `TRANSACTION_ID`    |   `10`    | `-`                   |
| `TRANSACTION_VALUE` |   `17`    | `15,2 (sem vírgula)`  |
| `TRANSACTION_DATE`  |   `10`    | `AAAA-MM-DD`          |
| `CLIENT_ID`         |   `10`    | `-`                   |
| `CLIENT_NAME`       |   `40`    | `-`                   |
| `PRODUCT_ID`        |   `10`    | `-`                   |
| `PRODUCT_MODEL`     |   `40`    | `-`                   |
| `RESERVED`          |   `12`    | `-`                   |
| `FIM DE LINHA (CR)` |   `01`    | `-`                   |
| **Total**           | **150**   | **Bytes**             |


**NOTA:** Cada linha possui um carácter de fim de linha `<ENTER>` que ocupa 1 posição. Cada linha tem **150 bytes** no total.


**Script em Python para Gerar o Arquivo com mais de 12Gb**

**`huge-file-generator.py`**

``` Python
#!venv/bin/python3

import json
import random
from datetime import datetime

with open('query_ibge_nomes_brasil.json', 'r') as f:
    clients = json.loads(f.read())


# Adding IDs to clients
client_id = 0
for client in clients:
    client_id +=1
    client['id'] = client_id 

with open('products.json', 'r') as f:
    products = json.loads(f.read())

count = 0
transaction_id = 234000
random.seed(5)
 

# CALCULO DE EXECUCOES - MAIOR QUE 12Gb
# 16000 * 150 = 2400000 bytes = 2,28Mb / execucao
# 12Gb = 12 * 1024 * 1024 * 1024 = 1024 ^3 * 12 = 12884901888 bytes
# Execucoes para chegar em 12Gb = 5368,70912 ~ 5370 vezes

today = datetime.today().strftime('%Y-%m-%d')

with open('big-file.txt', 'w') as f:
        
    for i in range(0, 5730):
        
        print(f'Looping = {i}')

        for client in clients:
            transaction_id += 1
            product = random.choice(products)

            field_transaction_id = ('0000000000'+ str(transaction_id))[-10:]
            field_transaction_value = ('00000000000000000'+ str(round(float(product["price_discount"]),2)).replace(".",""))[-17:]
            field_transaction_date = today

            field_client_id = ('0000000000'+ str(client['id']))[-10:]
            field_client_name = (client['nome'] + " "*40)[0:40].upper()
        
            field_product_id = ('0000000000'+ str(product["id"].strip().replace(' ','').replace('-','')))[-10:]
            field_product_model = (product["model"] + " "*40)[0:40].upper()
            
            # Registro contem 150 bytes (149 characters + <ENTER> (\n))
            f.write(f'{field_transaction_id}'
                    f'{field_transaction_value}'
                    f'{field_transaction_date}'
                    f'{field_client_id}'
                    f'{field_client_name}'
                    f'{field_product_id}'
                    f'{field_product_model}'
                    f'{" "*12}\n')
            
            count += 1

    print(f'Lines = {count}')
```

Arquivo gerado e exemplo de conteúdo. **Figuras 3 e 4**.

**Figura 3:** Tamanho do arquivo gerado.

![Generated File Size](/public/images/poc-huge-files/huge-file-01.png)

**Figura 4:** Exemplo do conteúdo do arquivo _(head)_.

![Generated File Head](/public/images/poc-huge-files/big-file-head-img2.png)

## Criando os lambdas

Conforme apresentado no fluxo da **figura 1**, haverão 2 lambdas para processar o arquivo.

1. **"Strategist Lambda"** (`poc-huge-file-strategist.py`)

1. **"Worker Lambda"** (`poc-huge-file-worker.py`)

Não será detalhado sobre o preparo dos ambientes da AWS que foram montados, mas fiquem à vontade para me perguntarem caso haja alguma dúvida.

A opção por usar o trigger via SNS e não usar um trigger direto no lambda foi devido ao SNS permitir mais de um consumidor para o mesmo evento. A opção de trigger para lambda permitiria que apenas 1 lambda fosse iniciado por evento. Isso é uma boa prática para casos em que mais processos utilizarão o mesmo arquivo. 

### Lambda Strategista (Strategist)

**Lambda: `poc-huge-file-strategist.py`**

``` python
# ---------------------------------------------------------------------
#  Description: Strategisty - It is trigged by SNS event when some file
#               is put in the S3 bucket and, check some infos such as
#               file size and number of partitions. After that it
#               invokes other lambdas (workers) givin to then parameters
#               like filename, start and end bytes to read and split
#               file in parallel.
#  Author     : Joao Roberto Perin
#  CreatedAt  : 2021-10-26
# ---------------------------------------------------------------------

import json
import boto3
import urllib.parse
import os
import uuid
from datetime import datetime


SAVEBUCKETAME=os.getenv('SAVEBUCKETNAME', 'jrperin-huge-file')
SAVEPREFIXPATH=os.getenv('SAVEPREFIXPATH', 'processed/')
LINELEN = int(os.getenv('LINELEN', '150'))
NFILES = int(os.getenv('NPARTS', '3'))

def lambda_handler(event, context):
    tstart = datetime.now()
    print(f"Start Date/Time: {tstart.strftime('%Y-%m-%d %H:%M:%S')}")
    print("Event received:")
    print(event)
    s3_event = json.loads(event['Records'][0]['Sns']['Message'])
    
    bucket_orig = s3_event['Records'][0]['s3']['bucket']['name']
    key_orig = urllib.parse.unquote_plus(s3_event['Records'][0]['s3']['object']['key'], encoding='utf-8')

    s3 = boto3.client('s3')
    file_size = 0
    try:
        response = s3.get_object(Bucket=bucket_orig, Key=key_orig)
        print("CONTENT TYPE: " + response['ContentType'])
        file_size = int(response['ContentLength'])
        print(f'file_size = {str(file_size)}')
    except Exception as e:
        print(e)
        print(f'Error getting object {key_orig} from bucket {bucket_orig}. Make sure they exist and your bucket is in the same region as this function.')
        raise e


    nlines = file_size // LINELEN
    lines_per_file = nlines // NFILES
    lines_rest = nlines % NFILES

    file_lines = { x : lines_per_file + 1 if x < lines_rest else lines_per_file for x in range(NFILES) }

    strategy = {}
    previous_end = -1

    cid = str(uuid.uuid4())

    file_prefix = key_orig.split('/')[-1]
    if '.' in file_prefix: 
        file_extension = file_prefix.split('.')[-1]
        file_prefix = file_prefix.split('.')[-2]

    for key, value in file_lines.items():

        start = previous_end + 1
        end = start + (value * LINELEN) - 1
        strategy[key] = { 'cid': cid, 'part': key, 'invoker': 'poc-huge-file-strategist',
                            'start' : start, 'end': end, 'size': value * LINELEN,
                            'bucket_origin' : bucket_orig, 'key_origin': key_orig, 
                            'bucket_destiny': SAVEBUCKETAME, 'key_destiny': f'{SAVEPREFIXPATH}{file_prefix}_p{("000"+str(key))[-3:]}_{str(start)}-{str(end)}{"." + file_extension if file_extension else ""}' }
        previous_end = end
 
    print(f'CID: {cid}')
    print(f'Number of partitions: {NFILES}')
    print('Strategy:')
    for k, v in strategy.items():
        print(f'Partition: {k}, Size: {v["size"]}')

    invoking_workers_lambdas(strategy)

    tend = datetime.now()
    tdelta = (tend - tstart)
    print(f"End Date/Time: {tend.strftime('%Y-%m-%d %H:%M:%S')}")
    print(f'Time spent in process = {round(tdelta.total_seconds()/60, 2)} seconds')
    
    return {
        'statusCode': 200,
        'body': json.dumps({"message": "OK"})
    }

def invoking_workers_lambdas(strategy):

    lambda_client = boto3.client('lambda')
    print("Workers invocations started...")
    for key, value in strategy.items():
        result=lambda_client.invoke(
            FunctionName="poc-huge-file-worker",
            InvocationType='Event',
            Payload=json.dumps(value)
        )
        print(f'Worker {key} invokation result:')
        print(result)

```

### Lambda Trabalhador (Worker)

**Lamda: `poc-huge-file-worker.py`**

``` python
# ---------------------------------------------------------------------
#  Description: Worker - It is trigged by the Strategisty lambda that
#               gives to it some information such as filename, start 
#               and end bytes, destination bucket to get part of the
#               original file and save it. This lambda will run in
#               parallel with other workers lambdas.
#  Author     : Joao Roberto Perin
#  CreatedAt  : 2021-12-10
# --------------------------------------------------------------------

import json
import boto3
from datetime import datetime

def lambda_handler(event, context):
    strategy = json.loads(event)

    tstart = datetime.now()

    print(f'CID: {strategy["cid"]}')
    print('Strategy:')
    print(event)

    s3 = boto3.client('s3')
    
    resp = s3.get_object(Bucket=strategy['bucket_origin'], Key=strategy['key_origin'], Range=f'bytes={str(strategy["start"])}-{str(strategy["end"])}')
    content = resp['Body']
    
    with content._raw_stream as fin:
        s3.upload_fileobj(fin, strategy['bucket_destiny'], strategy['key_destiny'])

    tend = datetime.now()
    tdelta = (tend - tstart)
    print(f'Time spent in process = {round(tdelta.total_seconds()/60, 2)} seconds')

    return {
        'statusCode': 200,
        'body': json.dumps({'partition': strategy['part'], 'message': 'OK'})
    }
```

## Iniciando os testes

Antes de executar o teste no arquivo com os 12Gb, foi feito teste em um arquivo menor (`big-file-head.txt`), com apenas 20 linhas.

De acordo com o conteúdo do arquivo, o particionamento deveria ficar conforme a **figura 5**.


**Figura 5:** Particionamento do arquivo `big-file-head.txt`

![Particionamento do arquivo big-file-head.txt](/public/images/poc-huge-files/big-file-head-img3.png)


Após executar os testes, vê-se que o resultado foi conforme esperado.

As 2 primeiras partições ficaram com 1050 bytes e a última com apenas 900 bytes. Essa diferenca de tamanho entre as partições é devida a quantidade de registros no arquivo não ser divisível por 3 igualmente, veja **figura 6**.


**Figura 6:** Log da execução da lambda `poc-huge-file-strategist`

![Log da execução da lambda poc-huge-file-strategist](/public/images/poc-huge-files/big-file-head-img-log1.png)


Abaixo nas **figuras** de **7 a 9** pode-se ver o resultado das execuções de cada **Worker** com sua respectiva partição.


**Figura 7:** Log da execução da lambda `poc-huge-file-worker` para a **partição 0**.

![Log da execução da lambda poc-huge-file-worker para a partição 0](/public/images/poc-huge-files/big-file-head-img-log2.png)


**Figura 8:** Log da execução da lambda `poc-huge-file-worker` para a **partição 1**.

![Log da execução da lambda poc-huge-file-worker para a partição 1](/public/images/poc-huge-files/big-file-head-img-log3.png)


**Figura 9:** Log da execução da lambda `poc-huge-file-worker` para a **partição 2**.

![Log da execução da lambda poc-huge-file-worker para a partição 2](/public/images/poc-huge-files/big-file-head-img-log4.png)


Nas **figuras de 10 a 13** pode-se ver o resultado do Bucket S3 com os 3 arquivos salvos e o conteúdo de cada um deles.


**Figura 10:** Bucket S3 com as 3 partições geradas.

![Bucket S3 com as 3 partições geradas](/public/images/poc-huge-files/big-file-head-img-s3_bucket.png)


**Figura 11:** Conteúdo da **partição 0**.

![Conteúdo da partição 0](/public/images/poc-huge-files/big-file-head-img-s3_par1_result.png)


**Figura 12:** Conteúdo da **partição 1**.

![Conteúdo da partição 1](/public/images/poc-huge-files/big-file-head-img-s3_par2_result.png)


**Figura 13:** Conteúdo da **partição 2**.

![Conteúdo da partição 2](/public/images/poc-huge-files/big-file-head-img-s3_par3_result.png)


Vê-se pelos resultados que é possível seguir com essa abordagem, pois o conteúdo dos arquivos gerados foi igual ao do _"teste de mesa"_ (**figura 6**).


## Testando com o arquivo grande

Abaixo estão os resultados da execução do mesmo processo com o arquivo gerado no início do artigo com mais de 12Gb de dados. **Figuras 14 a 17**.


**Figura 14:** Log execução Lambda Strategist - Arquivo com 12Gb.

![ Log execução Lambda Strategist - Arquivo com 12Gb](/public/images/poc-huge-files/huge-file-execution-log0.png)


**Figura 15:** Log execução Lambda Worker. Partição-0.

![ Log execução Lambda Worker / Partição-0](/public/images/poc-huge-files/huge-file-execution-wk0_log1.png)


**Figura 16:** Log execução Lambda Worker / Partição-1.

![ Log execução Lambda Worker / Partição-1](/public/images/poc-huge-files/huge-file-execution-wk1_log1.png)


**Figura 17:** Log execução Lambda Worker / Partição-2.

![ Log execução Lambda Worker / Partição-2](/public/images/poc-huge-files/huge-file-execution-wk2_log1.png)


Pelas logs dos **Workers**, vê-se que os 3 iniciaram praticamente ao mesmo tempo (`18:41:23h` coincidindo com o final da execução do **Strategist**).

Cada Worker levou pouco mais de 1 minuto para encerrar.

O tempo total de processamento foi de 1 minuto e 35 segundos para particionar em 3 o arquivo de 12Gb.

----

# Conclusão

* Com uma solução simples é possível paralelizar o processamento de grandes volumes de dados. De acordo com a AWS as quotas de [Concurrent Executions](https://sa-east-1.console.aws.amazon.com/servicequotas/home/services/lambda/quotas) são atualmente de 1.000 e podem ser aumentadas. Isso permite muito poder de processamento. 

* A utilização de AWS Lambdas é bem difundidaesse o que permite soluções fáceis de serem implementadas sem necessidade de muita curva de aprendizado.

* A leitura por **Range** bem como o **Multipart Upload** parecem ser recursos que ainda não estão bem difundidos.

* **A solução casa muito bem com arquivos de layout fixo** _(arquivos posicionais)_ que são comumente utilizados nos sistemas Mainframe. Para casos onde os registros possuem layout variável, como arquivos separados por vígulas (`*.csv`), precisa haver entendimento mais profundo sobre a viabilidade de solução de leitura por range.

* A lógica de particionamento pode ser alterada. Nesse artigo a proposta foi tratar por quantidade de partiçoes, mas outra abordagem seria por quantidade de dados por partição.

* A POC não aborda sobre o tempo limite máximo de 15 minutos do AWS lambda. Esse ponto poderia ser resolvido de 2 formas: 
    * Aumentar o número de partições, para reduzir o tamanho da partição e caber dentro do limite de tempo.

    * Cada Worker antes de atingir seu limite de tempo de execução poderia iniciar uma nova instância do Worker, passando até o ponto onde foi processado para que o seguinte dê continuidade no processamento. Essa proposta depende de outro recurso do AWS S3 que é o Muiltpart Upload. 

