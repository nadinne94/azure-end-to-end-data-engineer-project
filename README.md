<div align='center'>

  # Projeto Engenharia de Dados End-to-End com Azure
  ![Azure](https://img.shields.io/badge/Azure-0089D6?logo=microsoft-azure&logoColor=white)
  ![Databricks](https://img.shields.io/badge/Databricks-FF3621?logo=databricks&logoColor=white)
  ![Delta Lake](https://img.shields.io/badge/Delta_Lake-003366?logo=apachespark&logoColor=white)
  ![Power BI](https://img.shields.io/badge/Power_BI-F2C811?logo=powerbi&logoColor=black)
</div>
  
---
 ## Visão Geral
 Este projeto demonstra a implementação de um pipeline de dados na Azure aplicando a arquitetura Medallion[^1]. O fluxo cobre desde a ingestão de dados transacionais (AdventureWorksLT)[^2] até a disponibilização para consumo analítico no Power BI.
  
 ---
 ## Arquitetura
 A arquitetura segue um fluxo clássico de engenharia de dados:
 - Fonte de dados: base de dados relacional (SalesLT)
 - Ingestão: Azure Data Factory
 - Armazenamento: Azure Data Lake Storage Gen2
 - Processamento: Azure Databricks (Spark + Delta Lake)
 - Consumo: Power BI
<div align='center'>
    
  ![](docs/arquitetura.png)
</div>
  
 ---
 ## Tecnologias Utilizadas:
<div align='center'>
  
  Tecnologia | Descrição
  :---|:---
  SQL Server | Banco de dados (on-premises)
  Azure Data Factory| Orquestração e ingestão de dados
  Azure Data Lake Storage Gen2 | Armazenamento em camadas
  Azure Databricks | Processamento e transformação de dados
  Delta Lake | Camada transacional e versionamento
  Databricks SQL | Processamento e transformação de dados
  Power BI | Consumo analítico
</div>

 ---
 ## Estrutura do Data Lake
  
 ```text
  ADLS/
    ├──Containers/
          ├──bronze/
          ├──silver/
          └──gold/ 
 ```
- Bronze
  - Dados brutos ingeridos do banco relacional
  - Formato _Parquet_
- Silver
  - Processamento inicial do dados
  - Formato _Delta_
- Gold
  - Dados prontos para consumo analítico
  - Estrutura otimizada para BI

---
## Como executar
1. Criar recursos no Azure
  - Storage account
  - Data Factory
  - Databricks
  - Databricks Access Conector
  - Key Vault
2. Configurar permissões de acesso (RBAC)
3. Vincular Serviços _(Estúdio Data Factory)_
  - SQL Server
  - Key Vault
  - Storage Acount
  - Databricks
4. Executar o pipeline de ingestão no ADF
```
Pipeline Principal:
  ├── Lookup: Lista tabelas do SalesLT
  ├── ForEach: Processa cada tabela
  └── Copy Data: SQL Server → Data Lake
      ├── Fonte: SELECT * FROM SalesLT.{tabela}
      └── Destino: /bronze/SalesLT/{tabela}.parquet
```
---  
## 5.Processamento no Databricks
1ª Etapa:
  - Conexão com o ADLS Gen2
  - Leitura de dados da camada Bronze
2ª Etapa:
Notebook|Entrada|Saída|Transformações Principais
:---|:---|:---|:---
bronze_to_silver|/bronze/|/silver/|•Tratamento inicial dos dados<br> •Padronização dos nomes das colunas (snake_case)<br> •Conversão de campos de data para formato `yyyy-MM-dd`
silver_to_gold|/silver/|/gold/|•Garantia de consistência de schemag<br> •Reorganização dos dados por entidade<br> •Formato delta
3ª Etapa:
  - Criar tabelas Delta registradas como tabelas SQL
  - Conectar o Power BI ao SQL Warehouse_(databricks)_
> Os notebooks[^3] utilizados foram disponibilizados no final do projeto

---
## 6. Automação do Pipeline
- Conexão entre o Databricks e o Data Factory
![](docs/pipelineexecutada.png)

---
## Consumo Analítico
- Integração do Databricks com o Power BI
- Importação das tabelas via SQL Waherouse

<img width="3484" height="644" alt="image" src="https://github.com/user-attachments/assets/ba18cf21-c8e3-4745-826f-f3669c71c309" />

> Por ser um projeto voltado para engenharia de dado, no momento, o projeto não conteplou a analisa dos relacionamentos entre as tabelas e a visualização dos dados.
---
## Considerações Técnicas
- Projeto desenvolvido a partir de um tutorial do canal [Brazil Data Guy](https://www.youtube.com/watch?v=viKANCDhOqo&list=PLjofrX8yPdUQl_Z5w6gM0yet_3XGPSqjV)
- Algumas configurações foram ajustadas em relação ao tutorial original
- Orquestração centralizada no Azure Data Factory
- Transformações concentradas no Databricks
- Parquet foi utilizado na camada Bronze por ser eficiente para ingestão em larga escala.
- Delta Lake foi adotado nas camadas Silver e Gold para garantir consistência de schema, controle transacional e facilidade de evolução do pipeline
- O foco do projeto é Engenharia de Dados, não modelagem analítica
- Possível continuação do projeto no Power Bi trazendo alguns insigts dos dados.
  
---
## Nota de Rodapé
[^1]: [Medallion Architecture](https://www.databricks.com/br/glossary/medallion-architecture)
[^2]: [AdventureWorks](https://learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure?view=sql-server-ver16&tabs=ssms)
[^3]: [Storage Access](storage_access.ipynb)<br>[Bronze to Silver](bronzetosilver.ipynb)<br>[Silver to Gold](silvertogold.ipynb)
  
## Referências
[Key Vault](https://learn.microsoft.com/pt-br/azure/data-factory/store-credentials-in-key-vault)<br>
[Token Databricks](https://docs.databricks.com/aws/pt/dev-tools/auth/pat)<br>
[Secret Scope](https://learn.microsoft.com/en-us/azure/databricks/security/secrets/)<br>
[Sistema de Arquivos de Blobs do Azure(ABFSS)](https://learn.microsoft.com/pt-br/azure/storage/blobs/data-lake-storage-abfs-driver)<br>
  [Conexão Azure Data Lake](https://docs.databricks.com/aws/pt/connect/storage/azure-storage)

</div>
