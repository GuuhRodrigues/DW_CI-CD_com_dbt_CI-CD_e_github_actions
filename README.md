# Documentação do Projeto de Data Warehouse (DW) com ELT usando Dbt, CI/CD e GitHub Actions

## Visão Geral do Projeto
Este projeto tem como objetivo implementar um Data Warehouse (DW) utilizando a abordagem ELT (Extract, Load, Transform) com a ferramenta Dbt (Data Build Tool). O pipeline de integração contínua (CI) e entrega contínua (CD) foi configurado utilizando GitHub Actions para garantir que as transformações e cargas de dados sejam executadas de maneira automatizada e segura.

## Estrutura do Projeto
Esquemas do Banco de Dados
O projeto utiliza três esquemas distintos no banco de dados para representar diferentes ambientes de processamento:

  1. raw: Onde é realizada a extração e carga dos dados brutos.
  2. dev: Onde são feitas as transformações, testes e validações dos dados.
  3. prod: Onde os dados transformados e testados são carregados após a conclusão das etapas de CI e CD.
     
## Armazenamento de Metadados
O Dbt utiliza o arquivo manifest.json para comparar mudanças no projeto. Este arquivo é salvo em um bucket S3 para facilitar a recuperação e comparação de estados anteriores durante o processo de CI/CD.

## Código de Extração dos Dados
O código a seguir realiza a extração dos dados de commodities usando a biblioteca yfinance e salva os dados no esquema raw do banco de dados PostgreSQL:

```bash
import yfinance as yf
import pandas as pd
from sqlalchemy import create_engine
from dotenv import load_dotenv
import os

load_dotenv()

DB_HOST = os.getenv('DB_HOST_PROD')
DB_PORT = os.getenv('DB_PORT_PROD')
DB_NAME = os.getenv('DB_NAME_PROD')
DB_USER = os.getenv('DB_USER_PROD')
DB_PASS = os.getenv('DB_PASS_PROD')
DB_SCHEMA = os.getenv('DB_SCHEMA_RAW')

DATABASE_URL = f'postgresql://{DB_USER}:{DB_PASS}@{DB_HOST}:{DB_PORT}/{DB_NAME}'

engine = create_engine(DATABASE_URL)

commodities = ['CL=F', 'GC=F', 'SI=F']

def buscar_dados_commodities(simbolo, periodo='5d', intervalo='1d'):
    ticker = yf.Ticker(simbolo)
    dados = ticker.history(period=periodo, interval=intervalo)[['Close']]
    dados['simbolo'] = simbolo
    return dados

def buscar_todos_dados_commodities(commodities):
    todos_dados = []
    for simbolo in commodities:
        dados = buscar_dados_commodities(simbolo)
        todos_dados.append(dados)
    return pd.concat(todos_dados)

def salvar_no_postgres(df, schema='raw'):
    df.to_sql('commodities', engine, if_exists='replace', index=True, index_label='Date', schema=schema)

if __name__ == "__main__":
    dados_concatenados = buscar_todos_dados_commodities(commodities)
    salvar_no_postgres(dados_concatenados, schema='raw')
```
## Configuração de CI com GitHub Actions
O arquivo de configuração de CI (CI_action.yml) define o fluxo de trabalho para integração contínua, que é executado em cada pull request na branch principal:
```bash
name: CI_action

on:
  pull_request:
    branches:
      - main

jobs:
  CI_job:
    runs-on: ubuntu-latest

    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      S3_BUCKET_NAME: ${{ secrets.S3_BUCKET_NAME }}
      S3_PATH_MANIFEST: ${{ secrets.S3_PATH_MANIFEST }}
      DB_HOST_PROD: ${{ secrets.DB_HOST_PROD }}
      DB_PORT_PROD: ${{ secrets.DB_PORT_PROD }}
      DB_NAME_PROD: ${{ secrets.DB_NAME_PROD }}
      DB_USER_PROD: ${{ secrets.DB_USER_PROD }}
      DB_PASS_PROD: ${{ secrets.DB_PASS_PROD }}
      DB_SCHEMA_PROD: ${{ secrets.DB_SCHEMA_PROD }}
      DB_SCHEMA_DEV: ${{ secrets.DB_SCHEMA_DEV }}
      DB_SCHEMA_RAW: ${{ secrets.DB_SCHEMA_RAW }}
      DB_THREADS_PROD: ${{ secrets.DB_THREADS_PROD }}
      DB_TYPE_PROD: ${{ secrets.DB_TYPE_PROD }}
      DBT_PROFILES_DIR: ${{ secrets.DBT_PROFILES_DIR }}

    steps:
    - name: Checkout repository
      uses: actions/checkout@v3

    - name: Set up Python
      uses: actions/setup-python@v3
      with:
        python-version: '3.12'

    - name: Install dependencies
      run: pip install -r src/requirements.txt

    - name: Install AWS CLI
      run: sudo apt-get update && sudo apt-get install -y awscli

    - name: Configure AWS Credentials
      run: |
        aws configure set aws_access_key_id ${{ env.AWS_ACCESS_KEY_ID }}
        aws configure set aws_secret_access_key ${{ env.AWS_SECRET_ACCESS_KEY }}
        aws configure set default.region us-east-1
        aws configure set output json

    - name: Verify AWS CLI Installation
      run: aws --version

    - name: Copy manifest.json from S3
      run: |
        aws s3 cp s3://teste-eng-dados-dbt/manifest.json ./ || echo "Manifest not found"

    - name: Get Schema ID
      id: schema_id
      run: echo "SCHEMA_ID=${{ github.event.pull_request.number }}__${{ github.sha }}" >> $GITHUB_ENV

    - name: Run dbt debug
      run: |
        cd datawarehouse
        dbt debug --target pr --vars "schema_id: $SCHEMA_ID"

    - name: Run dbt deps
      run: |
        cd datawarehouse
        dbt deps --target pr --vars "schema_id: $SCHEMA_ID"

    - name: Run dbt build
      run: |
        cd datawarehouse
        if [ -f "./manifest.json" ]; then
          dbt build -s 'state:modified+' --defer --state ./ --target pr --vars "schema_id: $SCHEMA_ID"
        else
          dbt build --target pr --vars "schema_id: $SCHEMA_ID"
        fi
```
## Configuração de CD com GitHub Actions
O arquivo de configuração de CD (CD_action.yml) define o fluxo de trabalho para entrega contínua, que é executado em cada push na branch principal:
```bash
name: CD_action

on:
  push:
    branches:
      - main

jobs:
  CD_job:
    runs-on: ubuntu-latest

    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      S3_BUCKET_NAME: ${{ secrets.S3_BUCKET_NAME }}
      S3_PATH_MANIFEST: ${{ secrets.S3_PATH_MANIFEST }}
      DB_HOST_PROD: ${{ secrets.DB_HOST_PROD }} 
      DB_PORT_PROD: ${{ secrets.DB_PORT_PROD }}
      DB_NAME_PROD: ${{ secrets.DB_NAME_PROD }} 
      DB_USER_PROD: ${{ secrets.DB_USER_PROD }}  
      DB_PASS_PROD: ${{ secrets.DB_PASS_PROD }}
      DB_SCHEMA_PROD: ${{ secrets.DB_SCHEMA_PROD }} 
      DB_SCHEMA_DEV: ${{ secrets.DB_SCHEMA_DEV }}
      DB_SCHEMA_RAW: ${{ secrets.DB_SCHEMA_RAW }}
      DB_THREADS_PROD: ${{ secrets.DB_THREADS_PROD }} 
      DB_TYPE_PROD: ${{ secrets.DB_TYPE_PROD }}
      DBT_PROFILES_DIR: ${{ secrets.DBT_PROFILES_DIR }}

    steps:
    - name: Checkout repository
      uses: actions/checkout@v3

    - name: Set up Python
      uses: actions/setup-python@v3
      with:
        python-version: '3.12'

    - name: Install dependencies
      run: pip install -r src/requirements.txt

    - name: Install AWS CLI
      run: sudo apt-get update && sudo apt-get install -y awscli

    - name: Configure AWS Credentials
      run: |
        aws configure set aws_access_key_id ${{ env.AWS_ACCESS_KEY_ID }}
        aws configure set aws_secret_access_key ${{ env.AWS_SECRET_ACCESS_KEY }}
        aws configure set default.region us-east-1
        aws configure set output json

    - name: Verify AWS CLI Installation
      run: aws --version

    - name: Copy manifest.json from S3
      run: |
        echo "Copying manifest.json from S3"
        aws s3 cp s3://${{ env.S3_BUCKET_NAME }}/${{ env.S3_PATH_MANIFEST }} ./ || echo "Manifest not found"
        ls -al

    - name: Run dbt debug
      run: |
        cd datawarehouse
        dbt debug --target prod

    - name: Run dbt deps
      run: |
        cd datawarehouse
        dbt deps --target prod

    - name: Run dbt build
      run: |
        cd datawarehouse
        if [ -f "./manifest.json" ]; then
          dbt build -s 'state:modified+' --state ./ --target prod
        else
          dbt build --target prod
        fi

    - name: Copy new manifest.json to S3
      run: |
        echo "Copying new manifest.json to S3"
        aws s3 cp ./datawarehouse/target/manifest.json s3://${{ env.S3_BUCKET_NAME }}/
        ls -al

    - name: Display environment variables
      run: |
        echo "AWS_ACCESS_KEY_ID: ${{ env.AWS_ACCESS_KEY_ID }}"
        echo "AWS_SECRET_ACCESS_KEY: ${{ env.AWS_SECRET_ACCESS_KEY }}"
        echo "S3_BUCKET_NAME: ${{ env.S3_BUCKET_NAME }}"
        echo "S3_PATH_MANIFEST: ${{ env.S3_PATH_MANIFEST }}"
```

### Detalhamento do Fluxo de Trabalho

#### CI (Continuous Integration)

1. **Checkout do Repositório**: Faz checkout do código-fonte do repositório.
2. **Configuração do Python**: Instala a versão especificada do Python.
3. **Instalação de Dependências**: Instala as dependências necessárias listadas no arquivo `requirements.txt`.
4. **Instalação e Configuração do AWS CLI**: Instala e configura a CLI da AWS para interação com S3.
5. **Cópia do Manifesto do S3**: Faz o download do arquivo `manifest.json` do bucket S3.
6. **Definição do Schema ID**: Gera um ID único para o schema com base no número do pull request e no SHA do commit.
7. **Debug e Deps do Dbt**: Executa os comandos `dbt debug` e `dbt deps` para verificar a configuração e baixar as dependências.
8. **Build do Dbt**: Executa o comando `dbt build` para construir os modelos de dados, utilizando o estado deferido se o manifesto existir.

#### CD (Continuous Deployment)

1. **Checkout do Repositório**: Faz checkout do código-fonte do repositório.
2. **Configuração do Python**: Instala a versão especificada do Python.
3. **Instalação de Dependências**: Instala as dependências necessárias listadas no arquivo `requirements.txt`.
4. **Instalação e Configuração do AWS CLI**: Instala e configura a CLI da AWS para interação com S3.
5. **Cópia do Manifesto do S3**: Faz o download do arquivo `manifest.json` do bucket S3.
6. **Debug e Deps do Dbt**: Executa os comandos `dbt debug` e `dbt deps` para verificar a configuração e baixar as dependências.
7. **Build do Dbt**: Executa o comando `dbt build` para construir os modelos de dados, utilizando o estado deferido se o manifesto existir.
8. **Cópia do Novo Manifesto para o S3**: Faz o upload do novo arquivo `manifest.json` para o bucket S3.

### Conclusão

Esta documentação cobre a implementação de um Data Warehouse utilizando a abordagem ELT com Dbt e a configuração de pipelines de CI e CD utilizando GitHub Actions. A arquitetura do projeto com esquemas separados para raw, dev e prod, juntamente com a automação dos fluxos de trabalho de CI e CD, garante uma integração contínua e um processo de entrega contínua eficiente e confiável. 

Se precisar de mais alguma informação ou ajuste na documentação, por favor, me avise!
