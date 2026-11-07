# 🏬 Proyecto Lakehouse "Vendiqa" en AWS

[![Terraform](https://img.shields.io/badge/Infrastructure-Terraform-7B42BC?logo=terraform)](https://www.terraform.io/)
[![AWS](https://img.shields.io/badge/Cloud-AWS-FF9900?logo=amazonaws)](https://aws.amazon.com/)
[![Apache Iceberg](https://img.shields.io/badge/Format-Apache_Iceberg-E6522C?logo=apache)](https://iceberg.apache.org/)
[![Apache Spark](https://img.shields.io/badge/Engine-Apache_Spark-E25A1C?logo=apachespark)](https://spark.apache.org/)

**Curso:** Bases de Datos II (BDII-GC)  
**Docente:** Juan Alejandro Carrillo Jaimes  
**Integrantes:** Jose Leonardo Soler Duarte y [Nombre de tu Compañero/a]  
**Fecha de actualización:** 07 de noviembre de 2026  


---

## 📖 Descripción del Proyecto
Desarrollo de un Data Lakehouse escalable en AWS para "Vendiqa" (plataforma de retail/e-commerce). El proyecto transforma **7 millones de registros** transaccionales aplicando la arquitectura Medallion (`Landing` → `Bronze` → `Silver` → `Gold`). 

Toda la infraestructura se despliega como código (IaC) mediante **Terraform**. Se utiliza **Apache Spark en AWS Glue** para las transformaciones ETL, **Apache Iceberg** en las capas Silver y Gold para garantizar transacciones ACID, y **Amazon Athena** para consultas SQL sin servidor. La capa Gold se modela como un esquema estrella (`fact_sales` + 5 dimensiones), permitiendo responder consultas gerenciales en segundos y demostrando las ventajas de un Lakehouse (Time Travel, MERGE, GDPR Delete) sobre un Data Lake tradicional, con un presupuesto controlado (< 5 USD).

---

## 🏗️ Diagrama de Arquitectura

```mermaid
graph LR
    subgraph Origen
        Script[Script Python Faker<br/>7M registros CSV/Parquet]
    end

    subgraph AWS Cloud us-east-1
        subgraph S3[Lakehouse Vendiqa - Amazon S3]
            Landing[/landing: Crudo intacto/]
            Bronze[/bronze: Tablas crudas/]
            Silver[/"silver: Limpio y Validado<br/>(Apache Iceberg)"/]
            Gold[/gold: Modelo Estrella<br/>fact_sales + 5 dims (Iceberg)/]
        end

        Crawler[AWS Glue Crawler]
        Catalog[(AWS Glue Data Catalog)]
        
        Job1[AWS Glue Job 1: Spark<br/>Limpieza y Tipado]
        Job2[AWS Glue Job 2: Spark<br/>Transformación a Estrella]
        
        StepFunc[AWS Step Functions<br/>Orquestación .sync]
        
        Athena[Amazon Athena<br/>SQL sin servidor]
    end

    subgraph Gobernanza
        TF[Terraform: IaC<br/>apply / destroy]
        IAM[AWS IAM: Menor privilegio]
        Budget[AWS Budgets: Alerta < 5 USD]
    end

    Script -->|Carga inicial| Landing
    Landing -->|Detecta esquema| Crawler
    Crawler -->|Registra| Catalog
    
    StepFunc -->|Ejecuta| Job1
    Job1 -->|Espera .sync| StepFunc
    StepFunc -->|Ejecuta| Job2
    
    Job1 -.->|Lee/Escribe| Catalog
    Job2 -.->|Lee/Escribe| Catalog
    
    Catalog -.->|Metadatos Iceberg| Athena
    
    Athena -->|1. Consultas de Negocio| Gold
    Athena -->|2. Time Travel| Gold
    Athena -->|3. GDPR Delete| Gold

    TF -.- S3
    TF -.- StepFunc
    TF -.- Athena
    IAM -.- AWS Cloud
    Budget -.- AWS Cloud

    classDef aws fill:#FF9900,stroke:#333,stroke-width:2px,color:white;
    classDef iceberg fill:#E6522C,stroke:#333,stroke-width:2px,color:white;
    class S3,Job1,Job2,StepFunc,Athena,Catalog,Crawler aws;
    class Silver,Gold iceberg;
