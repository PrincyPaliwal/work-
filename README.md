# Serverless Databricks Job Orchestrator

This solution eliminates the need for managing a separate orchestration tool like Airflow by leveraging AWS-native services such as Step Functions and Lambda to orchestrate Databricks notebook execution.

## Serverless Databricks Job Orchestrator Architecture Flow

![Serverless Databricks Job Orchestrator Architecture](docs/icons/Serverless_Databricks_Job_Orchestrator.png)


## Key Benefits

1.  **Eliminates Airflow Dependency:**
    *   **Reduces Infrastructure Overhead:** No need to manage an Airflow instance, reducing maintenance, scaling, and uptime concerns.
    *   **Simplifies Workflow Management:** AWS Step Functions provide a fully managed orchestration service with built-in retry mechanisms.

2.  **Cost Efficiency:**
    *   **Pay-as-you-go Model:** Serverless execution means you only pay for what you use, eliminating the need for persistent EC2 instances required by an Airflow setup.
    *   **No Dedicated Orchestrator:** Removes the cost associated with running and maintaining an Airflow cluster.

3.  **Native AWS Integration:**
    *   **Step Functions for Orchestration:** Provides stateful execution of Databricks jobs with visual workflow tracking.
    *   **Lambda for Triggering Jobs:** Serverless functions efficiently handle job execution, eliminating the need for a constantly running scheduler.
    *   **CloudWatch Logging:** Centralized logging ensures easier debugging and monitoring.

4.  **Simplified Security and Access Control:**
    *   **IAM-based Security:** Uses AWS IAM roles and policies to control access, making security management simpler than maintaining Airflow's role-based access.
    *   **Integration with AWS Secrets Manager:** Securely manages Databricks API credentials (recommended).

5.  **Scalability and Fault Tolerance:**
    *   **Automatic Scaling:** AWS services scale automatically without provisioning resources.
    *   **Built-in Error Handling:** Step Functions support error handling, retries, and fallback states, reducing manual intervention.

## When to Use This Approach?

*   If you don't want to manage Airflow and prefer AWS-native orchestration.
*   If you need serverless and cost-efficient orchestration.
*   If you want tighter integration with AWS services for logging, security, and monitoring.
*   If you have simple to moderately complex workflows without Airflow-specific features like DAG dependencies.

## Implementation

1.  **Orchestration:** Step Functions orchestrate Databricks notebook execution.
2.  **Job Triggering:** Lambda functions trigger jobs via the Databricks API.
3.  **Logging:** Logs are stored in CloudWatch.

## Getting Started


### Step 1: Clone the Repository
```sh
git clone https://github.kadellabs.com/digiclave/databricks-accelerators.git
cd databricks-accelerators\Accelerators\serverless_databricks_job_orchestrator
```

Follow these steps to set up the Serverless Databricks Job Orchestrator:

## 1. Create an IAM Role for Lambda

1.  **Create IAM Role:** Create an IAM Role (e.g., `AWSLambdaBasicExecutionRole`) with permissions to:
    *   Invoke Step Functions
    *   Write logs to CloudWatch
    *   Call Databricks API

2.  **Attach IAM Policy:** Attach the following IAM Policy (example - adjust as needed):

    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "logs:CreateLogGroup",
            "logs:CreateLogStream",
            "logs:PutLogEvents"
          ],
          "Resource": "*"
        },
        {
          "Effect": "Allow",
          "Action": "states:StartExecution",
          "Resource": "*"
        },
        {
          "Effect": "Allow",
          "Action": [
            "databricks:*" // Grant Databricks API permissions - refine as needed
          ],
          "Resource": "*"
        }
      ]
    }
    ```

## 2. Create a Lambda Function to Trigger Databricks Jobs

1.  **Create Lambda Function:** Go to AWS Lambda → Create a new function.
2.  **Runtime:** Python 3.9 (or higher)
3.  **Handler:** `lambda_function.lambda_handler`
4.  **Execution Role:** Attach the IAM role created in Step 1.

5.  **Lambda Code (Python):**

    ```python
    import json
    import os
    import requests

    # Databricks Configuration (Use Secrets Manager in production!)
    DATABRICKS_INSTANCE = "https://<your-databricks-instance>" # Replace
    DATABRICKS_TOKEN = "dapi-xxxxxxxxxxxxxxxxxxxx" # Replace or use Secrets Manager
    DATABRICKS_JOB_ID = "<your-databricks-job-id>" # Replace

    def lambda_handler(event, context):
        url = f"{DATABRICKS_INSTANCE}/api/2.1/jobs/run-now"
        headers = {
            "Authorization": f"Bearer {DATABRICKS_TOKEN}",
            "Content-Type": "application/json"
        }
        payload = {"job_id": DATABRICKS_JOB_ID}

        try:
            response = requests.post(url, headers=headers, json=payload)
            response_data = response.json()

            if response.status_code == 200:
                return {
                    "statusCode": 200,
                    "body": json.dumps({"message": "Databricks job triggered", "run_id": response_data.get("run_id")})
                }
            else:
                return {
                    "statusCode": response.status_code,
                    "body": json.dumps({"error": response_data})
                }
        except Exception as e:
            return {
                "statusCode": 500,
                "body": json.dumps({"error": str(e)})
            }
    ```

6.  **Deploy Lambda Function:** Upload the code as a ZIP or paste it directly into the inline editor. Save and Deploy.

## 3. Create an AWS Step Functions State Machine

1.  **Open Step Functions:** Open Step Functions in AWS.
2.  **Create State Machine:** Click Create State Machine → Choose Standard Workflow.
3.  **State Machine Definition (JSON):** Use the following JSON:

    ```json
    {
      "Comment": "Step Function to orchestrate Databricks Jobs",
      "StartAt": "TriggerDatabricksJob",
      "States": {
        "TriggerDatabricksJob": {
          "Type": "Task",
          "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:YourLambdaFunctionName",
          "End": true,
          "Retry": [
            {
              "ErrorEquals": ["States.TaskFailed"],
              "IntervalSeconds": 5,
              "MaxAttempts": 3,
              "BackoffRate": 2.0
            }
          ],
          "Catch": [
            {
              "ErrorEquals": ["States.ALL"],
              "Next": "JobFailed"
            }
          ]
        },
        "JobFailed": {
          "Type": "Fail",
          "Cause": "Databricks Job Execution Failed"
        }
      }
    }
    ```

4.  **Replace:**
    *   `REGION`: Your AWS region (e.g., `us-east-1`)
    *   `ACCOUNT_ID`: Your AWS account ID
    *   `YourLambdaFunctionName`: The Lambda function name

5.  **Save and Deploy:** Save and Deploy the Step Function.

## 4. Test the Workflow

1.  **Trigger Step Function:** Manually trigger the Step Function.

2.  **Verify Execution:** It should:
    *   Call the Lambda function.
    *   The Lambda function will invoke the Databricks Jobs API.
    *   If successful, the job will run in Databricks.
    *   If it fails, Step Functions will retry 3 times before marking as failed.

## 5. Monitoring & Debugging

1.  **CloudWatch Logs:** Lambda execution logs are in AWS CloudWatch. Find them under `/aws/lambda/YourLambdaFunctionName`.

2.  **Step Functions Execution History:** Open AWS Step Functions → Select your State Machine. Navigate to Execution History for logs.

## Contributions
Feel free to submit pull requests for improvements or additional features.
 
## License
This project is licensed under the **MIT License**.
 
## Contact
For issues or support, reach out via **GitHub Issues** or email the project maintainer
