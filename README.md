<div align="justify">

# Azure End-to-End Data Engineering Project

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
- **Azure Key Vault**: segurança e governança dos dados
- **Azure Databricks**: processamento e transformação de dados
- **Power BI**: consumo analítico

---

## 5. Pipeline de Dados

### Como Executar
1. Criar recursos no Azure (Data Factory, Storage Account, Databricks, Databricks Conector, Key Vault)
2. Configurar Microsoft Integration Runtime
3. Conectar o Data Factory ao banco de dados SQL Server On-Premises
4. [Ingestão dos dados brutos no Data Lake Storage Gen2](#ingest-anchor-point)
5. [Autenticar e acessar aos dados armazendaos, além da aplicação de transformações de dados no Databricks](#treatment-anchor-point)
6. [Executar pipeline no Data Factory](#pipeline-anchor-point)

<a name="ingest-anchor-point">

 ### Ingestão de Dados (Bronze)
 - Fonte: SQL Server local (AdventureWorksLT)
 - Utilização de **Self-hosted Integration Runtime**
 - Extração dinâmica das tabelas do schema `SalesLT`
 - Dados salvos em formato **Parquet**
</a>

<a name="treatment-anchor-point">

 ### Processamento dos Dados
 #### Acesso ao armazenamento[^2]
 - Armazenamento das chaves cliente_id e cliente_secret no Secret Scope[^3]
 - Definição dos caminhos de acesso ao Azure Data Lake Gen2 utilizando o protocolo ABFSS[^4][^5]
 #### Bronze to Silver[^6]
 - Padronização dos nomes das colunas (snake_case)
 - Conversão de campos de data para formato `yyyy-MM-dd`
 - Dados salvos no container silver após tratamento inicial.
#### Silver to Gold[^7]
- Reorganização dos dados por entidade
- Garantia de consistência de schema

> A validação de schema garante consistência entre execuções e facilita a evolução do pipeline, mesmo quando o consumo Delta não é direto.

</a>
<a name="pipeline-anchor-point">

### Visualização do Pipeline
![](docs/pipelineexecutada.png)
</a>

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
- Conexão direta do Power BI ao Azure Data Lake Storage Gen2
- Leitura dos dados da camada Gold
- Dados preparados para análise sem necessidade de movimentação adicional
  
> Por limitações de SKU, o consumo Delta no Power BI foi feito via Parquet pois não foi possível usar Databricks SQL Warehouse.

![](docs/relaçãotabelasbi.png)

---

## 8. Considerações Técnicas
- O projeto foi adaptado para o ambiente **Azure Free Trial**
- Algumas configurações foram ajustadas em relação ao tutorial original
- Orquestração centralizada no Azure Data Factory
- Transformações concentradas no Databricks
- Uso de Parquet no consumo final devido a limitações de SKU
- O foco do projeto é Engenharia de Dados, não modelagem analítica

---

## 9. Fonte
Projeto desenvolvido a partir de um tutorial do canal **Brazil Data Guy**, com adaptações e implementações próprias.

[Projeto Engenharia de Dados End to End](https://www.youtube.com/watch?v=viKANCDhOqo&list=PLjofrX8yPdUQl_Z5w6gM0yet_3XGPSqjV)

---

## Referências
[^1]:Medallion Architecture: https://www.databricks.com/br/glossary/medallion-architecture
[^2]:Acesso via ABFSS: https://github.com/nadinne94/azure-end-to-end-data-engineer-project/blob/main/storage_access.ipynb
[^3]:Secret Scope: https://learn.microsoft.com/en-us/azure/databricks/security/secrets/
[^4]:Sistema de Arquivos de Blobs do Azure(ABFSS): https://learn.microsoft.com/pt-br/azure/storage/blobs/data-lake-storage-abfs-driver
[^5]:Conexão Azure Data Lake: https://docs.databricks.com/aws/pt/connect/storage/azure-storage
[^6]:Bronze to Silver: https://github.com/nadinne94/azure-end-to-end-data-engineer-project/blob/main/bronzetosilver.ipynb
[^7]:Silver to Gold: https://github.com/nadinne94/azure-end-to-end-data-engineer-project/blob/main/silvertogold.ipynb
