<div align="center">

# Projeto Engenharia de Dados End-to-End com Azure
![Azure](https://img.shields.io/badge/Azure-0089D6?logo=microsoft-azure&logoColor=white)
![Databricks](https://img.shields.io/badge/Databricks-FF3621?logo=databricks&logoColor=white)
![Power BI](https://img.shields.io/badge/Power_BI-F2C811?logo=powerbi&logoColor=black)

</div>

---
## Índice
- [Visão Geral](#visão-geral)
- [Arquitetura](#arquitetura)
- [Tecnologias Utilizadas](#tecnologias-utilizadas)
- [Pipeline de Dados](#pipeline-de-dados)
- [Consumo Analítico](#consumo-analítico)
- [Considerações Técnicas](considerações-técnicas)
- [Referências](#referências)
---
## Visão Geral
Este projeto implementa um pipeline de dados na Azure usando a **Medallion Architecture**. O fluxo cobre desde a ingestão de dados transacionais do AdventureWorksLT até a disponibilização para consumo analítico no Power BI.

---
## Arquitetura
<div align="center">

  ![](docs/arquitetura.png)

Fluxo dos dados: SQL Server → ADF → ADLS (Bronze/Silver/Gold) → Databricks → Power BI
</div>

---
## Tecnologias Utilizadas:

- **SQL Server:** banco de dados (on-premises)
- **Azure Data Factory**: orquestração e ingestão de dados
- **Azure Data Lake Storage Gen2**: armazenamento em camadas
- **Azure Databricks Access Connector**: integração do ao Databricks
- **Azure Key Vault**: segurança e governança dos dados
- **Databricks**: processamento e transformação de dados
- **Power BI**: consumo analítico
---

## Pipeline de Dados
### Criar recursos no Azure
1. Storage account
```
/Armazenamento de dados
  ├──Contêineres
       ├──bronze
       ├──silver
       └──gold/
```
2. Data Factory
3. Databricks
4. Databricks Access Conector
5. Key Vault

### Vincular Serviços
- SQL Server
- Key Vault
- Storage Acount
- Databricks
### Ingestão dos dados
<details open>
<summary>No data Factory:</summary>

 
 LookUp
  - SQL Server
  
 ```
   SELECT
         s.name AS SchemaName,
         t.name AS TableName
         FROM sys.tables t
         INNER JOIN sys.schemas s
         ON t.schema_id = s.schema_id
         WHERE s.name = 'SalesLT'
   ```
 
 ForEach
  - Conectar ao LookUp: ```@activity('Look for all tables').output.value```
  - Copydata
    - _Copia os dados_: SQL Server -> Serviço Vinculado: (SHIR): `@{concat('SELECT * FROM ', item().SchemaName, '.', item().TableName)}`
     <img width="1085" height="166" alt="image" src="https://github.com/user-attachments/assets/f70f966e-a41d-4027-bf72-e18021f568e9" />
   
     - _Cola os dados_: Azure Data Lake Gen2 -> _Parquet(formato do arquivo)_
    <div align='center'><img width="973" height="254" alt="image" src="https://github.com/user-attachments/assets/1c81056a-4dbd-48bf-a9c1-c71ce6f1c87c" /></div>
 <img width="1837" height="772" alt="image" src="https://github.com/user-attachments/assets/36818e48-232f-4e19-b701-d4b035aa20f4" />
 
 </details>

 ### Processamento dos dados
 Autenticar e acessar aos dados armazendaos, além da aplicação de transformações de dados no Databricks
 - Iniciar o Databricks
 - Criate Compute
 - Workspace -> Shared -> Notebook
 #### Acesso ao armazenamento
 - Armazenamento das chaves cliente_id e cliente_secret no Secret Scope[^4]
 - Definição dos caminhos de acesso ao Azure Data Lake Gen2 utilizando o protocolo ABFSS[^5][^6]
 - [Storage_access:](storage_access.ipynb)
 #### Processamento dos Dados
 - Tratamento inicial dos dados
 - Padronização dos nomes das colunas (snake_case)
 - Conversão de campos de data para formato `yyyy-MM-dd`
 - Dados salvos no container **_silver_**
 - [Broze to Silver:](bronzetosilver.ipynb)
#### Polimento dos dados
- Reorganização dos dados por entidade
- Garantia de consistência de schema
- Dados salvos no container **_gold_**, disponibilizados para consumo.
- [Silver to Gold](silvertogold.ipynb)

### Conexão entre o Databricks e o Data Factory
- Data Factory -> Atividades -> Notebook
  - Bronze to Silver:
    - Serviço Vinculado ao Databricks
    - Caminho do notebook: `/Shared/silvertogold`
  - Silver to Gold:
    - Serviço Vinculado ao Databricks
    - Caminho do notebook: `/Shared/silvertogold`

![](docs/pipelineexecutada.png)

---
### Estrutura do Data Lake

```text
/Containers
  ├──bronze/
      └── SalesLT/
       ├── Address/
       ├── Customer/
       ├── Product/
  
  ├──silver/
      └── SalesLT/
       ├── Address/
       ├── Customer/
       ├── Product/
  
  └──gold/
      └── SalesLT/
       ├── Address/
       ├── Customer/
       ├── Product/
 ````

----
## Consumo Analítico

Transformar os dados em formato delta em tabelas sql
  ```
  gold_base_path = "abfss://gold@armazenamentodatalake26.dfs.core.windows.net/SalesLT/"
  spark.sql("USE CATALOG databricks_7405608788677390")
  spark.sql("USE SCHEMA gold")
  gold_tables = dbutils.fs.ls(gold_base_path)
  for item in gold_tables:
    table_name = item.name.replace("/", "")
    table_path = item.path

    if table_name.startswith("_"):
        continue

    print(f"Criando tabela: {table_name}")
    
    spark.sql(f"""
        CREATE TABLE IF NOT EXISTS {table_name}
        USING DELTA
        LOCATION '{table_path}'
    """)
  ```
- Conexão direta do Power BI ao Databricks
- Leitura dos dados da camada Gold
- Dados preparados para análise
- Possível continuação do projeto trazendo alguns insigts dos dados.

<img width="3484" height="644" alt="image" src="https://github.com/user-attachments/assets/ba18cf21-c8e3-4745-826f-f3669c71c309" />

---
## Considerações Técnicas
- O projeto foi adaptado para o ambiente **Azure Free Trial**
- Projeto desenvolvido a partir de um tutorial do canal [Brazil Data Guy](https://www.youtube.com/watch?v=viKANCDhOqo&list=PLjofrX8yPdUQl_Z5w6gM0yet_3XGPSqjV)
- Algumas configurações foram ajustadas em relação ao tutorial original
- Orquestração centralizada no Azure Data Factory
- Transformações concentradas no Databricks
- Parquet foi utilizado na camada Bronze por ser eficiente para ingestão em larga escala.
- Delta Lake foi adotado nas camadas Silver e Gold para garantir consistência de schema, controle transacional e facilidade de evolução do pipeline
- O foco do projeto é Engenharia de Dados, não modelagem analítica

---
## Referências
[Medallion Architecture](https://www.databricks.com/br/glossary/medallion-architecture)<br>
[Key Vault](https://learn.microsoft.com/pt-br/azure/data-factory/store-credentials-in-key-vault)<br>
[Token Databricks](https://docs.databricks.com/aws/pt/dev-tools/auth/pat)<br>
[Secret Scope](https://learn.microsoft.com/en-us/azure/databricks/security/secrets/)<br>
[Sistema de Arquivos de Blobs do Azure(ABFSS)](https://learn.microsoft.com/pt-br/azure/storage/blobs/data-lake-storage-abfs-driver)<br>
[Conexão Azure Data Lake](https://docs.databricks.com/aws/pt/connect/storage/azure-storage)
