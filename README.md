<div align="justify">

# Azure End-to-End Data Engineering Project

## 1. Visão Geral
Este projeto demonstra a implementação de um pipeline de Engenharia de Dados de ponta a ponta *(end to end)* utilizando serviços da Microsoft Azure. O fluxo cobre desde a ingestão de dados transacionais até a disponibilização dos dados para consumo analítico no Power BI.

O cenário utiliza o banco de dados **AdventureWorksLT**, simulando um ambiente real de dados operacionais.

---

## 2. Arquitetura

 A solução foi construída seguindo o padrão **Medallion Architecture[^1]**:

- **Bronze**: dados brutos ingeridos do SQL Server
- **Silver**: dados tratados e padronizados
- **Gold**: dados prontos para consumo analítico

<div align="center">

  ![](docs/arquitetura.png)

Fluxo dos dados: SQL Server → ADF → ADLS (Bronze/Silver/Gold) → Databricks → Power BI
</div>

### Tecnologias Utilizadas:
- **SQL Server:** banco de dados (on-prem)
- **Azure Data Factory**: orquestração e ingestão de dados
- **Azure Data Lake Storage Gen2**: armazenamento em camadas
- **Azure Databricks**: processamento e transformação de dados
- **Power BI**: consumo analítico
---

## 3. Ingestão de Dados (Bronze)
- Fonte: SQL Server local (AdventureWorksLT)
- Ingestão realizada via Azure Data Factory
- Utilização de **Self-hosted Integration Runtime**
- Extração dinâmica das tabelas do schema `SalesLT`
- Dados salvos em formato **Parquet**

---

## 4. Processamento de Dados (Silver)
As transformações foram realizadas no Azure Databricks utilizando PySpark:

1º Passo: Configurar o acesso ao Azure Data Lake[^2][^3]
Definição dos caminhos de acesso ao Azure Data Lake Gen2 utilizando o protocolo ABFSS, garantindo padronização e separação de responsabilidades entre configuração e transformação de dados.
![](storage_access.py)


- Padronização dos nomes das colunas (snake_case)
- Conversão de campos de data para formato `yyyy-MM-dd`
- Organização dos dados por tabela
- Escrita dos dados tratados na camada Silver

---

## 5. Disponibilização para Consumos (Gold)
Na camada Gold, os dados foram organizados para facilitar o consumo analítico:

- Estruturação dos dados por domínio

> A criação de tabelas no metastore e a modelagem dimensional não fazem parte do escopo do projeto.

---

## 6. Consumo Analítico (Power BI)
- Conexão direta do Power BI ao Azure Data Lake Storage Gen2
- Leitura dos dados da camada Gold
- Dados preparados para análise sem necessidade de movimentação adicional

---

## 7. Estrutura do Data Lake
bronze/

└── SalesLT/

silver/

└── SalesLT/

gold/

└── SalesLT/


---

## 8. Considerações Técnicas
- O projeto foi adaptado para o ambiente **Azure Free Trial**
- Algumas configurações foram ajustadas em relação ao tutorial original
- O foco do projeto é Engenharia de Dados, não modelagem analítica

---

## 9. Fonte
Projeto desenvolvido a partir de um tutorial do canal **Brazil Data Guy**, com adaptações e implementações próprias.


## Referências
[^1]:Medallion Architecture: https://www.databricks.com/br/glossary/medallion-architecture
[^2]:Sistema de Arquivos de Blobs do Azure(ABFSS): https://learn.microsoft.com/pt-br/azure/storage/blobs/data-lake-storage-abfs-driver
[^3]:Conexão Azure Data Lake: https://docs.databricks.com/aws/pt/connect/storage/azure-storage
Secret Scopehttps://learn.microsoft.com/en-us/azure/databricks/security/secrets/
