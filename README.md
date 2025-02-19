# Real-Time Data Flow: SQL Server to Databricks to Sigma

## Overview

This project demonstrates a real-time data flow pipeline between SQL Server, Databricks, and Sigma. The pipeline ensures that transactional data from SQL Server is processed in Databricks and visualized in Sigma, enabling real-time analytics.

## Components

- **SQL Server**: Source database containing transactional data.
- **Databricks**: Processes and transforms data for analytics.
- **Sigma**: Visualization tool for real-time data insights.
- **Azure Event Hub (Optional)**: Used for real-time data streaming.
- **Delta Lake**: Stores processed data for efficient querying.

## Workflow

1. **Data Ingestion**: SQL Server transactions are streamed using Change Data Capture (CDC) or an ETL pipeline.
2. **Databricks Processing**: Data is ingested into Databricks, cleaned, and stored in Delta Lake.
3. **Data Transformation**: Business logic is applied to process raw data.
4. **Sigma Integration**: The processed data is visualized using Sigma dashboards.

## Prerequisites

- SQL Server instance with transactional data.
- Databricks workspace setup.
- Sigma account for visualization.
- (Optional) Azure Event Hub for real-time streaming.

## SQL Server Setup

Ensure CDC is enabled for the relevant table:

```sql
EXEC sys.sp_cdc_enable_db;
GO
EXEC sys.sp_cdc_enable_table
  @source_schema = N'dbo',
  @source_name   = N'orders',
  @role_name     = NULL;
GO
```

## Databricks Code

### 1. Ingest Data from SQL Server

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *

spark = SparkSession.builder.appName("SQLServer_Ingestion").getOrCreate()

jdbc_url = "jdbc:sqlserver://<your-server>:1433;databaseName=<your-db>"
properties = {
    "user": "<your-username>",
    "password": "<your-password>",
    "driver": "com.microsoft.sqlserver.jdbc.SQLServerDriver"
}

orders_df = spark.read.jdbc(jdbc_url, "dbo.orders", properties=properties)
orders_df.write.format("delta").mode("overwrite").save("/mnt/delta/orders")
```

### 2. Transform Data

```python
transformed_df = orders_df.withColumn("order_date", to_date("order_timestamp"))
transformed_df.write.format("delta").mode("overwrite").save("/mnt/delta/transformed_orders")
```

### 3. Stream Data in Real Time

```python
streaming_df = spark.readStream.format("delta").load("/mnt/delta/orders")
query = streaming_df.writeStream.format("console").start()
query.awaitTermination()
```

## Sigma Integration

1. Connect to Databricks using JDBC.
2. Use SQL queries to extract insights from Delta tables.
3. Create real-time dashboards with the transformed data.

## Deployment

- Schedule a Databricks job to run the ETL pipeline at fixed intervals.
- Use Databricks Delta Live Tables (DLT) for continuous processing.
- Set up alerting and monitoring using Databricks or Sigma.

## Conclusion

This pipeline enables real-time data flow from SQL Server to Databricks and visualization in Sigma, ensuring up-to-date insights for decision-making.

## Contributions
Feel free to submit pull requests for improvements or additional features.

## License
This project is licensed under the **MIT License**.

## Contact
For issues or support, reach out via **GitHub Issues** or email the project maintainer.


