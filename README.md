<div align="justify">

# Projeto Engenharia de Dados End to End com Azure

## 1. Visão Geral
Este projeto demonstra a implementação de um pipeline de Engenharia de Dados de ponta a ponta *(end to end)* utilizando serviços da Microsoft Azure. O fluxo cobre desde a ingestão de dados transacionais até a disponibilização dos dados para consumo analítico no Power BI.

O cenário utiliza o banco de dados **AdventureWorksLT**, simulando um ambiente real de dados operacionais.

---
## 2. Objetivo do Projeto
Demonstrar a construção de um pipeline de Engenharia de Dados escalável em Azure, aplicando a arquitetura Medallion desde a ingestão até o consumo analítico.

---
## 3. Arquitetura

 A solução foi construída seguindo o padrão **Medallion Architecture[^1]**:
- **Bronze**: dados brutos ingeridos do SQL Server
- **Silver**: dados tratados e padronizados
- **Gold**: dados prontos para consumo analítico

<div align="center">

  ![](docs/arquitetura.png)

Fluxo dos dados: SQL Server → ADF → ADLS (Bronze/Silver/Gold) → Databricks → Power BI
</div>

### Tecnologias Utilizadas:
- **SQL Server:** banco de dados (on-premises)
- **Azure Data Factory**: orquestração e ingestão de dados
- **Azure Data Lake Storage Gen2**: armazenamento em camadas
- **Azure Databricks Access Connector**: integrar o Azure Data Factory ao Azure Databricks
- **Azure Key Vault**: segurança e governança dos dados
- **Azure Databricks**: processamento e transformação de dados
- **Power BI**: consumo analítico
---
## 5. Pipeline de Dados
### Criar recursos no Azure
#### Storage account
- Tipo de armazenamento preferencial: Armazenamento de Blobs do Azure ou Azure Data Lake Storage Gen 2
- Desempenho: Standard
- Replicação: LRS (armazenamento com redundância local)
- Habilitar namespace hierárquico
- Controle de acesso: Storage Blob Data Contributor
  - Data Factory
  - Databricks Access Conector
  - Microsoft Entra Id (Registro de aplicativo)
```
/Armazenamento de dados
  ├──Contêineres
       ├──bronze
       ├──silver
       └──gold/
```
#### Data Factory
- Habilitar a Rede Virtual Gerenciada no AutoResolveIntegrationRuntime padrão Habilitar namespace hierárquico
#### Databricks
- Tipo de Preço: Premium
- Tipo de workspace: Híbrido
#### Databricks Access Conector
- Configuração padrão
#### Key Vault
- Tipo de preço: Padrão
- Modelo de permissão: Política de acesso de cofre
- Politica de acesso:
  - Data Factory
  - Databricks

### Vinculação de serviços (SQL Server, Key Vault, Storage Acount, Databricks)
#### Conectar o Data Factory ao banco de dados SQL Server On-Premises
- [Configurar o SQL Server](sqlserveronprem)
- Serviço Vinculado: SQL Server
  - Runtime de integração: Self-Hosted (SHIR)
    - Configuração expressa
      - Tipo de Autenticação: Autenticação do SQL
      - Acesso Key Vault: Serviço vinculado do AKV[^2]
      - Trust server certificate
#### Conectar o Data Factory ao Storage Acount
- Runtime de integração: AutoResolveIntegrationRuntime (Criação interativa habilitado)
- Tipo de Autenticação: Chave da Conta
#### Conectar o Data Factory ao Databricks
- Runtime de integração: AutoResolveIntegrationRuntime (Criação interativa habilitado)
- Cluster Interativo Existente
- Tipo de autenticação: Acesso ao Token[^3]
### Ingestão dos dados
Ingestão dos dados do banco de dados on-prem na camada bronze do datalake
- Criar novo pipeline -> Atividades:
  - LookUp
    - Novo conjunto de dados: SQL Server
    - Usar consulta
         ```
         SELECT
         s.name AS SchemaName,
         t.name AS TableName
         FROM sys.tables t
         INNER JOIN sys.schemas s
         ON t.schema_id = s.schema_id
         WHERE s.name = 'SalesLT'
         ```
  - ForEach
    - Conectar ao LookUp
        - Itens: ```@activity('Look for all tables').output.value```
    - Copydata
       - Novo conjunto de dados: SQL Server -> Serviço Vinculado: (SHIR)
       - Consulta: `@{concat('SELECT * FROM ', item().SchemaName, '.', item().TableName)}`
         <div align='center'>
          <img width="1085" height="166" alt="image" src="https://github.com/user-attachments/assets/f70f966e-a41d-4027-bf72-e18021f568e9" />
         </div>
       - Conjunto de dados do coletor: Azure Data Lake Gen2
         - Formato: Parquet
           - Caminho do arquivo:
             - Sistema de arquivo: contêiner **bronze** já criado no StorageAcount
             - Diretório: `@{concat(dataset().schemaname, '/', dataset().tablename)}`
             - Nome do Arquivo: `@{concat(dataset().tablename, '.parquet')}`

 <img width="1837" height="772" alt="image" src="https://github.com/user-attachments/assets/36818e48-232f-4e19-b701-d4b035aa20f4" />

 ### Processamento dos dados
 Autenticar e acessar aos dados armazendaos, além da aplicação de transformações de dados no Databricks
 - Iniciar o Databricks
 - Criate Compute
   - Runtime do Databricks: Scala 2.12
   - Aceleração do Photon: Desativar
   - Terminar após: 15minutos de inatividade
   - Modo de acesso: Partilhado
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
- [Silver to Gold](bronzetosilver.ipynb)

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
## 6. Estrutura do Data Lake

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
## 7. Consumo Analítico (Power BI)
- Transformar os dados em formato delta em tabelas sql
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
## 8. Considerações Técnicas
- O projeto foi adaptado para o ambiente **Azure Free Trial**
- Algumas configurações foram ajustadas em relação ao tutorial original
- Orquestração centralizada no Azure Data Factory
- Transformações concentradas no Databricks
- Parquet foi utilizado na camada Bronze por ser eficiente para ingestão em larga escala.
- Delta Lake foi adotado nas camadas Silver e Gold para garantir consistência de schema, controle transacional e facilidade de evolução do pipeline
- O foco do projeto é Engenharia de Dados, não modelagem analítica

---
## 9. Fonte
Projeto desenvolvido a partir de um tutorial do canal **Brazil Data Guy**, com adaptações e implementações próprias.

[Projeto Engenharia de Dados End to End](https://www.youtube.com/watch?v=viKANCDhOqo&list=PLjofrX8yPdUQl_Z5w6gM0yet_3XGPSqjV)

---
## Referências
[^1]:Medallion Architecture: https://www.databricks.com/br/glossary/medallion-architecture
[^2]Key Vault: https://learn.microsoft.com/pt-br/azure/data-factory/store-credentials-in-key-vault
[^3]:Token Databricks: https://docs.databricks.com/aws/pt/dev-tools/auth/pat
[^4]:Secret Scope: https://learn.microsoft.com/en-us/azure/databricks/security/secrets/
[^5]:Sistema de Arquivos de Blobs do Azure(ABFSS): https://learn.microsoft.com/pt-br/azure/storage/blobs/data-lake-storage-abfs-driver
[^6]:Conexão Azure Data Lake: https://docs.databricks.com/aws/pt/connect/storage/azure-storage
