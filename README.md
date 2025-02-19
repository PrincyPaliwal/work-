# Automated Data Ingestion Pipeline

## Overview

This accelerator automates data ingestion from AWS services into Databricks Delta Lake with schema evolution. It leverages AWS Glue, Amazon S3, Kinesis, and EventBridge to build a robust and scalable data pipeline.

## Automated Data Ingestion Pipeline Architecture Flow

![Automated Data Ingestion Pipeline](docs/icons/Automated_Data_Ingestion_Pipeline.png)


## Features

- **End-to-End Data Pipeline:** Automates data ingestion from AWS to Databricks Delta Lake.
- **AWS Glue Crawler:** Detects schema changes in S3 and updates metadata.
- **Amazon Kinesis:** Streams real-time data for continuous processing.
- **AWS EventBridge:** Triggers Databricks transformations based on Glue Crawler events.
- **Databricks Integration:** Executes transformation notebooks and monitors execution.
- **Delta Lake Schema Evolution:** Automatically adapts to changing data structures.
- **AWS CloudFormation:** Automates infrastructure provisioning (Kinesis, Glue, S3).
- **Scalability:** Designed to handle large-scale real-time and batch processing.
- **Serverless Execution:** Uses AWS Lambda for event-driven triggers.
- **Monitoring & Logging:** Tracks job execution and alerts on failures.

## Prerequisites

Before using this accelerator, ensure the following:

- An AWS account with necessary permissions (Glue, Kinesis, EventBridge, S3, CloudFormation, Lambda).
- A Databricks workspace.
- Basic understanding of AWS services (Glue, Kinesis, EventBridge, S3, CloudFormation, Lambda).

## Setup Instructions (All Steps Merged)

```python
import boto3
import json
import base64
import requests
from pyspark.sql import SparkSession
from pyspark.sql.functions import col
from delta.tables import DeltaTable

# --- CONFIGURATION (Replace with actual values) ---
KINESIS_STREAM_NAME = "your-kinesis-stream"
KINESIS_REGION = "us-east-1"
DELTA_LAKE_BUCKET = "your-delta-lake-bucket"
DELTA_TABLE_PATH = f"s3://{DELTA_LAKE_BUCKET}/delta-table"
GLUE_DATABASE_NAME = "my-database"
GLUE_CRAWLER_NAME = "my-glue-crawler"
S3_DATA_BUCKET = "your-data-bucket" # Bucket where your raw data resides
DATABRICKS_HOST = "https://your-databricks-instance"
DATABRICKS_TOKEN = "your-databricks-token" # Use a PAT
DATABRICKS_NOTEBOOK_PATH = "/Shared/your_notebook"
CLOUDFORMATION_STACK_NAME = "AWSIngestionPipeline"
AWS_REGION = "us-east-1"

# --- AWS Clients ---
glue_client = boto3.client('glue', region_name=AWS_REGION)
events_client = boto3.client('events', region_name=AWS_REGION)
cloudformation_client = boto3.client('cloudformation', region_name=AWS_REGION)

# --- Combined Function Definitions and Execution ---

# Step 1: AWS Infrastructure Setup using CloudFormation
def create_cloudformation_stack():
    template_body = f"""
    AWSTemplateFormatVersion: '2010-09-09'
    Resources:
      KinesisStream:
        Type: AWS::Kinesis::Stream
        Properties:
          Name: {KINESIS_STREAM_NAME}
          ShardCount: 1
      GlueDatabase:
        Type: AWS::Glue::Database
        Properties:
          CatalogId: !Ref AWS::AccountId
          DatabaseInput:
            Name: {GLUE_DATABASE_NAME}
      S3Bucket:
        Type: AWS::S3::Bucket
        Properties:
          BucketName: {DELTA_LAKE_BUCKET}
    """
    response = cloudformation_client.create_stack(
        StackName=CLOUDFORMATION_STACK_NAME,
        TemplateBody=template_body,
        Capabilities=['CAPABILITY_NAMED_IAM']
    )
    return response

# Step 2: AWS Glue Crawler Setup
def create_and_start_glue_crawler():
    response = glue_client.create_crawler(
        Name=GLUE_CRAWLER_NAME,
        Role='AWSGlueServiceRole',  # Replace with appropriate role
        DatabaseName=GLUE_DATABASE_NAME,
        Targets={'S3Targets': [{'Path': f's3://{S3_DATA_BUCKET}/'}]} # Point to your data bucket
    )
    glue_client.start_crawler(Name=GLUE_CRAWLER_NAME)
    return response

# Step 3: AWS EventBridge Rule Setup
def setup_eventbridge_rule():
    response = events_client.put_rule(
        Name='GlueCrawlerCompleteRule',
        EventPattern=json.dumps({"source": ["aws.glue"], "detail-type": ["Glue Crawler State Change"]}),
        State='ENABLED'
    )
    # Target the Databricks notebook execution
    target_response = events_client.put_targets(
        Rule='GlueCrawlerCompleteRule',
        Targets=[
            {
                'Id': 'DatabricksNotebookTarget',  # Unique ID for the target
                'Arn': f'arn:aws:lambda:{AWS_REGION}:{AWS_ACCOUNT_ID}:function:trigger-databricks-notebook' # Lambda to trigger Databricks
            }
        ]
    )
    return response, target_response

# Step 4: Databricks Notebook Execution and Monitoring (Lambda Function)
def trigger_databricks_notebook(event, context):
    headers = {"Authorization": f"Bearer {DATABRICKS_TOKEN}"}
    payload = {"notebook_path": DATABRICKS_NOTEBOOK_PATH, "params": {}}

    response = requests.post(f"{DATABRICKS_HOST}/api/2.0/jobs/run-now", headers=headers, json=payload)

    job_id = response.json().get("run_id")

    if job_id:
        print(f"Databricks job {job_id} triggered successfully. Monitoring execution...")
        while True:
            status_response = requests.get(
                f"{DATABRICKS_HOST}/api/2.0/jobs/runs/get?run_id={job_id}", headers=headers)
            status = status_response.json().get("state", {}).get("life_cycle_state", "UNKNOWN")
            if status in ["TERMINATED", "SUCCESS", "FAILED"]:
                print(f"Job completed with status: {status}")
                break
        return {"statusCode": 200, "body": f"Databricks job {job_id} completed with status: {status}"}
    else:
      return {"statusCode": 500, "body": f"Failed to trigger Databricks notebook. Response: {response.text}"}


# Step 5: Kinesis Stream Processing (Databricks - run in a notebook)
def process_kinesis_stream():
    spark = SparkSession.builder.appName("KinesisToDelta").getOrCreate() # Create Spark Session inside the notebook
    df = spark.readStream.format("kinesis") \
        .option("streamName", KINESIS_STREAM_NAME) \
        .option("region", KINESIS_REGION) \
        .option("startingPosition", "LATEST") \
        .load()

    query = df.selectExpr("CAST(data AS STRING) as jsonData") \
        .writeStream \
        .format("delta") \
        .option("checkpointLocation", f"s3://{DELTA_LAKE_BUCKET}/_checkpoints/") \
        .option("mergeSchema", "true") \
        .outputMode("append") \
        .start(DELTA_TABLE_PATH)

    query.awaitTermination()


# --- Main Execution (AWS Lambda - triggered by EventBridge) ---
def lambda_handler(event, context):
    # This Lambda function is triggered by EventBridge when the Glue Crawler completes.
    # It then triggers the Databricks notebook.
    response = trigger_databricks_notebook(event, context)
    return response

# --- Local Testing (Optional) ---
# if __name__ == "__main__":
#     create_cloudformation_stack()
#     create_and_start_glue_crawler()
#     setup_eventbridge_rule()
#     # process_kinesis_stream() # Run this in Databricks notebook


## Contributions
Feel free to submit pull requests for improvements or additional features.
 
## License
This project is licensed under the **MIT License**.
 
## Contact
For issues or support, reach out via **GitHub Issues** or email the project maintainer
