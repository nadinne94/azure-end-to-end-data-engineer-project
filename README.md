# Azure End-to-End Data Engineering Project

## 1. Visão Geral
Este projeto demonstra a implementação de um pipeline de Engenharia de Dados de ponta a ponta *(end to end)* utilizando serviços da Microsoft Azure. O fluxo cobre desde a ingestão de dados transacionais até a disponibilização dos dados para consumo analítico no Power BI.

O cenário utiliza o banco de dados **AdventureWorksLT**, simulando um ambiente real de dados operacionais.

---

## 2. Arquitetura
A solução foi construída seguindo o padrão **Medallion Architecture[^1]**, promovendo organização, escalabilidade e separação de responsabilidades.
- **Bronze**: dados brutos ingeridos do SQL Server
- **Silver**: dados tratados e padronizados
- **Gold**: dados prontos para consumo analítico

### Tecnologias Utilizadas:
- **SQL Server:** banco de dados (on-prem)
- **Azure Data Factory**: orquestração e ingestão de dados
- **Azure Data Lake Storage Gen2**: armazenamento em camadas
- **Azure Databricks**: processamento e transformação de dados
- **Power BI**: consumo analítico

**Visual – Diagrama de Arquitetura**
- Fluxo ponta a ponta: SQL Server → ADF → ADLS (Bronze/Silver/Gold) → Databricks → Power BI

Visual: docs/arquitetura.png

---

## Referências
[^1]:Medallion Architecture: https://www.databricks.com/br/glossary/medallion-architecture
