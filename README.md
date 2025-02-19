# Real-time Streaming Analytics Accelerator

This accelerator provides a framework for processing real-time data streams using AWS services like Amazon Kinesis, AWS Lambda, and Databricks Structured Streaming. It's designed to handle high-volume data ingestion, transformation, and storage for various use cases.

## Realtime Streaming Analytics Architecture Flow

![Realtime_Streaming_Analytics_Architecture](docs/icons/Realtime_Streaming_Analytics.png)

## Purpose

The accelerator processes real-time logs, financial transactions, sensor data, and other streaming data to generate actionable insights.

## Implementation

1.  **Data Ingestion:** Amazon Kinesis (or Kafka) ingests the incoming data stream.
2.  **Real-time Processing:** Databricks Structured Streaming performs transformations and writes the processed data to Delta Lake.
3.  **Storage:** Processed insights are stored in Amazon Redshift or S3 for analysis and reporting.

## Use Cases

1.  **Log Analytics:** Continuously process logs from applications, servers, or security systems to detect anomalies, failures, or security threats in real time.
2.  **Financial Transactions Monitoring:** Monitor payment transactions for fraud detection, compliance checks, and anomaly detection.
3.  **IoT & Sensor Data Processing:** Analyze real-time sensor readings from industrial machines, vehicles, or smart devices to identify operational issues or optimize performance.
4.  **Clickstream Analysis:** Track user behavior on websites or mobile apps to personalize recommendations and detect potential bot traffic.
5.  **Stock Market & Trading Analytics:** Process financial market data streams to execute algorithmic trading or provide real-time insights to traders.

## Key Benefits

1.  **Low-latency insights:** Enables immediate decision-making based on real-time data.
2.  **Scalability:** Handles large volumes of streaming data efficiently.
3.  **Cost-effectiveness:** Leverages serverless processing with AWS Lambda for certain tasks.
4.  **Integration with BI tools:** Seamlessly integrates with BI tools like QuickSight, Tableau, or Power BI for data visualization and reporting.

## Solution Overview

The solution comprises the following components:

1.  **Infrastructure Provisioning:** Terraform manages the provisioning of AWS resources, including Kinesis, Redshift, S3, and Lambda.
2.  **Streaming Data Ingestion:** AWS Kinesis captures and buffers the incoming real-time events.
3.  **Real-time Processing:** Databricks Structured Streaming transforms the data and writes it to Delta Lake for efficient storage and querying.
4.  **Storage:** Processed data is stored in Amazon Redshift and S3, providing flexibility for different analytical needs.

## Getting Started


### Step 1: Clone the Repository
```sh
git clone https://github.kadellabs.com/digiclave/databricks-accelerators.git
cd databricks-accelerators\Accelerators\realtime_streaming_analytics
```

Follow these steps to deploy and run the accelerator:

## 1. Prerequisites

1.  An AWS account.
2.  Terraform installed.
3.  A Databricks workspace.
4.  Basic understanding of AWS services (Kinesis, S3, Redshift, Lambda, IAM).
5.  Python 3.8 or higher installed locally (for Lambda deployment).
6.  AWS CLI configured.

## 2. Infrastructure Provisioning (Terraform)

1.  **Create Terraform Files:** Create a directory (e.g., `infra`) and create the `main.tf` file within it. Copy and paste the Terraform code provided below into `main.tf`.  **Important:** Replace placeholder values like passwords, bucket names, and region with your actual values.  Also, review the IAM policies and ensure they are appropriate for your security requirements (avoid overly permissive policies like `AmazonS3FullAccess` in production).

    ```terraform
    # infra/main.tf

    provider "aws" {
      region = "us-east-1" # Replace with your region
    }

    # Kinesis Stream
    resource "aws_kinesis_stream" "data_stream" {
      name             = "real-time-stream" # Replace if needed
      shard_count      = 1
      retention_period = 24
    }

    # S3 Bucket for Processed Data
    resource "aws_s3_bucket" "processed_data" {
      bucket = "streaming-processed-data-bucket" # Replace with your bucket name
    }

    # Redshift Cluster
    resource "aws_redshift_cluster" "analytics_cluster" {
      cluster_identifier = "analytics-cluster" # Replace if needed
      database_name      = "analytics_db"
      master_username    = "admin"
      master_password    = "SuperSecurePass123!" # Replace with a strong password
      node_type          = "dc2.large"
      cluster_type       = "single-node"
      skip_final_snapshot = true
    }

    # IAM Role for Databricks
    resource "aws_iam_role" "databricks_role" {
      name = "databricks-streaming-role" # Replace if needed
      assume_role_policy = <<EOF
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Action": "sts:AssumeRole",
          "Principal": { "Service": "ec2.amazonaws.com" },
          "Effect": "Allow"
        }
      ]
    }
    EOF
    }

    # Attach policies to IAM Role (Example - Adjust permissions as needed)
    resource "aws_iam_policy_attachment" "databricks_s3_access" {
      name       = "databricks-s3-access"
      roles      = [aws_iam_role.databricks_role.name]
      policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess" # Consider more restrictive policy
    }

    #... (Other resources like Redshift IAM role, Lambda IAM role, etc.)
    ```

2.  **Initialize Terraform:**

    ```bash
    cd infra/
    terraform init
    ```

3.  **Plan and Apply:**

    ```bash
    terraform plan
    terraform apply -auto-approve # Use -auto-approve cautiously in production
    ```

    Note the output values, especially the Kinesis stream name, S3 bucket name, and Redshift endpoint.

## 3. Databricks Structured Streaming Code

1.  **Create Databricks Notebook:** Create a new notebook (Python).


    ```python
    # Databricks Notebook (Python)

    from pyspark.sql import SparkSession
    from pyspark.sql.functions import col, from_json, expr
    from pyspark.sql.types import StructType, StringType, IntegerType

    # Initialize Spark Session
    spark = SparkSession.builder \
      .appName("RealTimeStreamingAnalytics") \
      .config("spark.jars.packages", "com.databricks:spark-redshift_2.12:2.0.1") \
      .getOrCreate()

    # Define Schema for Incoming Data (Example)
    schema = StructType() \
      .add("event_id", StringType()) \
      .add("user_id", StringType()) \
      .add("transaction_amount", IntegerType()) \
      .add("event_timestamp", StringType())

    # Read Streaming Data from Kinesis
    kinesis_stream = spark.readStream \
      .format("kinesis") \
      .option("streamName", "real-time-stream") # Replace with your stream name
      .option("region", "us-east-1") # Replace with your region
      .option("initialPosition", "LATEST") \
      .load()

    # Parse JSON Data
    parsed_stream = kinesis_stream \
      .selectExpr("CAST(data AS STRING) as jsonData") \
      .select(from_json(col("jsonData"), schema).alias("data")) \
      .select("data.*")

    # Write to Delta Lake
    delta_path = "s3://streaming-processed-data-bucket/delta-lake/" # Replace with your S3 path
    query = parsed_stream.writeStream \
      .format("delta") \
      .outputMode("append") \
      .option("checkpointLocation", delta_path + "/checkpoint/") \
      .start(delta_path)

    # Write to Redshift
    parsed_stream.write \
      .format("com.databricks.spark.redshift") \
      .option("url", "jdbc:redshift://[analytics-cluster.xxxxxxxx.region.redshift.amazonaws.com:5439/analytics_db](https://www.google.com/search?q=https://analytics-cluster.xxxxxxxx.region.redshift.amazonaws.com:5439/analytics_db)") # Replace with your Redshift URL
      .option("dbtable", "real_time_events") \
      .option("tempdir", "s3://streaming

## Extracted Python Code from Notebooks

### Realtime Streaming Analytics Code

```python
# Use %run to import and access variables from another Databricks notebook
```python
%run "./Realtime Streaming Variables"

from pyspark.sql import SparkSession 
from pyspark.sql.functions import col, from_json, expr 
from pyspark.sql.types import StructType, StringType, IntegerType 

spark = SparkSession.builder \ 
    .appName("RealTimeStreamingAnalytics") \ 
    .config("spark.jars.packages", "com.databricks:spark-redshift_2.12:2.0.1") \ 
    .getOrCreate() 

schema = StructType() \ 
    .add("event_id", StringType()) \ 
    .add("user_id", StringType()) \ 
    .add("transaction_amount", IntegerType()) \ 
    .add("event_timestamp", StringType()) 

kinesis_stream = spark.readStream \ 
    .format("kinesis") \ 
    .option("streamName", "real-time-stream") \ 
    .option("region", "us-east-1") \ 
    .option("initialPosition", "LATEST") \ 
    .load() 

parsed_stream = kinesis_stream \ 

    .selectExpr("CAST(data AS STRING) as jsonData") \ 
    .select(from_json(col("jsonData"), schema).alias("data")) \ 
    .select("data.*") 



 delta_path

query = parsed_stream.writeStream \ 
    .format("delta") \ 
    .outputMode("append") \ 
    .option("checkpointLocation", delta_path + "/checkpoint/") \ 
    .start(delta_path) 

 

parsed_stream.write \ 
    .format("com.databricks.spark.redshift") \ 
    .option("url", "jdbc:redshift://analytics-cluster.xxxxxxxx.region.redshift.amazonaws.com:5439/analytics_db") \ 
    .option("dbtable", "real_time_events") \ 
    .option("tempdir", "s3://streaming-processed-data-bucket/temp/") \ 
    .mode("append") \ 
    .save() 

query.awaitTermination() 

import boto3 

import json 
s3_client = boto3.client("s3") 

sns_client = boto3.client("sns") 

 TOPIC_ARN 

def lambda_handler(event, context): 

    for record in event["Records"]: 

        bucket = record["s3"]["bucket"]["name"] 

        key = record["s3"]["object"]["key"] 

        message = f"New processed data file in {bucket}: {key}" 

    sns_client.publish( 

            TopicArn=TOPIC_ARN, 

            Message=message, 

            Subject

        ) 

     

    return {"statusCode": 200, "body": json.dumps("Notification Sent")} 
```

### Realtime Streaming Variables Code

```python

    delta_path =  "s3://streaming-processed-data-bucket/delta-lake/",
    TOPIC_ARN =  "arn:aws:sns:us-east-1:123456789012:DataProcessed",
    Subject="New Processed Data Available" 
```

 ## Contributions
Feel free to submit pull requests for improvements or additional features.
 
## License
This project is licensed under the **MIT License**.
 
## Contact
For issues or support, reach out via **GitHub Issues** or email the project maintainer
