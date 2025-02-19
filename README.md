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


## Databricks SQL Queries

### A2RawData Query
```sql
%sql
SELECT
  c.Ticker,
  c.Issuer,
  c.ResourceProvider,
  c.StartDate,
  c.EndDate,
  c.AdjustedValue,
  c.Location,
  c.Region,
  c.Format,
  c.Value,
  c.Currency,
  c.Title,
  c.AggregatedStatus,
  c.Attendee,
  c.Status,
  c.Note,
  c.Team,
  c.CreatedAt,
  c.LastUpdate,
  c.CreatedBy,
  c.LastUpdateBy,
  c.InOffice,
  c.AllDay,
  c.CreatedByIngestor,
  c.BuysideStatus,
  c.IsReviewRequired,
  c.Id,
  c.SplitTeam AS research_splits,
  CASE
    WHEN bc.broker IS NOT NULL THEN bc.broker
    ELSE 'Other'
  END AS ParentBroker
FROM
  (
    SELECT
      MAX(resource_provider) AS resource_provider,
      broker
    FROM
      freerider.sigma_input.broker_mapping
    GROUP BY
      broker
  ) bc
RIGHT JOIN (
  SELECT
    r.*,
    r.Value * CAST(rs.weighting AS DOUBLE) AS AdjustedValue,
    rs.SplitTeam
  FROM
    risk_sql_prod.dbo.a2arecords r
  INNER JOIN (
    SELECT DISTINCT
      fullname,
      SplitTeam,
      weighting
    FROM (
      SELECT
        fullname,
        team AS SplitTeam,
        weighting,
        ROW_NUMBER() OVER (
          PARTITION BY fullname, team
          ORDER BY fullname, team DESC
        ) AS rn
      FROM
        freerider.sigma_input.research_split
    ) rs
    WHERE rn = 1
  ) rs
  ON r.Attendee = rs.fullname
) c
ON c.ResourceProvider = bc.resource_provider;
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

