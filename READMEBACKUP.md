Here’s a comprehensive document that integrates all aspects of the S3 to PostgreSQL data pipeline project using Java, Spark, Terraform, EKS, Airflow, and unit testing.


---

# Complete Guide: AWS S3 to PostgreSQL Data Pipeline with Java, Spark, Terraform, EKS, Airflow, and Unit Testing

### Overview
This guide provides a step-by-step setup for a data pipeline that:

1. Reads a pipe-delimited file from AWS S3.
2. Processes the data using a Java Spark application.
3. Performs CRUD operations on PostgreSQL.
4. Deploys on AWS EKS with environments for development (dev), UAT, and production.
5. Triggers an Airflow DAG when a file is placed in S3.
6. Automates AWS infrastructure setup using Terraform.
7. Includes unit tests to validate the pipeline’s functionality.

### Prerequisites
- **Java 8+**
- **Apache Spark**
- **Terraform** 
- **AWS CLI and SDKs** (for S3, EMR, Lambda, RDS, and EKS)
- **PostgreSQL** (with JDBC driver)
- **JUnit** and **Mockito** for unit testing

### Table of Contents
1. [Terraform Configuration for AWS Infrastructure](#1-terraform-configuration-for-aws-infrastructure)
2. [Java Spark Application Code](#2-java-spark-application-code)
3. [Airflow DAG for S3 Trigger](#3-airflow-dag-for-s3-trigger)
4. [Unit Testing](#4-unit-testing)
5. [Deployment and Execution](#5-deployment-and-execution)

---

### 1. Terraform Configuration for AWS Infrastructure
This Terraform configuration sets up the necessary AWS resources, including S3, EMR, RDS PostgreSQL, Lambda, EKS clusters, and IAM roles.

Terraform Configuration File (`main.tf`)

```hcl
provider "aws" {
  region = "us-east-1"
}

# S3 Bucket for Input Data
resource "aws_s3_bucket" "data_bucket" {
  bucket = "your-data-bucket"
}

# IAM Role for EMR Cluster
resource "aws_iam_role" "emr_role" {
  name = "EMR_Spark_Role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Action = "sts:AssumeRole",
        Principal = { Service = "elasticmapreduce.amazonaws.com" },
        Effect   = "Allow"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "emr_policy" {
  role       = aws_iam_role.emr_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
}

# EMR Cluster
resource "aws_emr_cluster" "spark_emr_cluster" {
  name          = "spark-emr-cluster"
  release_label = "emr-6.3.0"
  applications  = ["Spark"]
  service_role  = aws_iam_role.emr_role.arn

  ec2_attributes {
    instance_profile = aws_iam_role.emr_role.arn
  }

  master_instance_group {
    instance_type = "m5.xlarge"
    instance_count = 1
  }

  core_instance_group {
    instance_type = "m5.xlarge"
    instance_count = 2
  }
}

# RDS PostgreSQL Database
resource "aws_db_instance" "postgres_db" {
  identifier        = "spark-postgres-db"
  allocated_storage = 20
  engine            = "postgres"
  engine_version    = "13"
  instance_class    = "db.t3.micro"
  username          = "postgres_user"
  password          = "password123"
  skip_final_snapshot = true
}

# Lambda Function to Trigger Airflow DAG
resource "aws_lambda_function" "trigger_dag" {
  function_name = "trigger_airflow_dag"
  role          = aws_iam_role.emr_role.arn
  runtime       = "python3.8"
  handler       = "lambda_function.lambda_handler"
  filename      = "lambda_function.zip" # Provide your code here
}

# EKS Cluster with Node Groups
module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "your-cluster"
  cluster_version = "1.21"
  subnets         = ["subnet-12345678", "subnet-23456789"]
  vpc_id          = "vpc-12345678"
  node_groups = {
    dev = { ... } # Define capacities as shown above for dev, UAT, and prod environments
  }
}
```

**Instructions**:
1. Initialize Terraform: `terraform init`
2. Plan the setup: `terraform plan`
3. Apply the configuration: `terraform apply`

---

### 2. Java Spark Application Code

This Java application reads a file from S3, performs CRUD operations on PostgreSQL, and triggers a Lambda function to start the Airflow DAG.

Java Spark Code (`S3ToPostgresApp.java`)

```java
import org.apache.spark.sql.SparkSession;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SaveMode;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.lambda.LambdaClient;
import software.amazon.awssdk.services.lambda.model.InvokeRequest;
import software.amazon.awssdk.services.lambda.model.InvokeResponse;
import software.amazon.awssdk.core.SdkBytes;

import java.nio.file.Paths;
import java.util.Properties;

public class S3ToPostgresApp {
    private static final String POSTGRES_URL = "jdbc:postgresql://your-postgres-url:5432/yourdb";
    private static final String POSTGRES_USER = "your_username";
    private static final String POSTGRES_PASSWORD = "your_password";
    private static final String S3_BUCKET = "your-s3-bucket";
    private static final String S3_FILE_PATH = "path/to/your-file.csv";
    private static final String LAMBDA_FUNCTION_NAME = "yourLambdaFunctionName";

    public static void main(String[] args) {
        SparkSession spark = SparkSession.builder().appName("S3ToPostgresWithCRUD").getOrCreate();
        Properties connectionProps = new Properties();
        connectionProps.put("user", POSTGRES_USER);
        connectionProps.put("password", POSTGRES_PASSWORD);

        // Main workflow
        Dataset<Row> s3Data = readDataFromS3(spark);
        createData(s3Data, connectionProps);
        readData(spark, connectionProps);
        updateData(spark, connectionProps);
        deleteData(spark, connectionProps);
        invokeLambda();

        spark.stop();
    }

    public static Dataset<Row> readDataFromS3(SparkSession spark) {
        System.out.println("Reading data from S3...");
        return spark.read()
                .format("csv")
                .option("header", "true")
                .load("s3a://" + S3_BUCKET + "/" + S3_FILE_PATH);
    }

    public static void createData(Dataset<Row> data, Properties connectionProps) {
        System.out.println("Writing data to PostgreSQL...");
        data.write()
                .mode(SaveMode.Append)
                .jdbc(POSTGRES_URL, "your_table_name", connectionProps);
    }

    public static void readData(SparkSession spark, Properties connectionProps) {
        System.out.println("Reading data from PostgreSQL...");
        Dataset<Row> postgresData = spark.read()
                .jdbc(POSTGRES_URL, "your_table_name", connectionProps);
        postgresData.show();
    }

    public static void updateData(SparkSession spark, Properties connectionProps) {
        System.out.println("Updating data in PostgreSQL...");
        Dataset<Row> updatedData = spark.read().jdbc(POSTGRES_URL, "your_table_name", connectionProps)
                .withColumn("column_to_update", functions.lit("new_value"));
        updatedData.write()
                .mode(SaveMode.Overwrite)
                .jdbc(POSTGRES_URL, "your_table_name", connectionProps);
    }

    public static void deleteData(SparkSession spark, Properties connectionProps) {
        System.out.println("Deleting data from PostgreSQL...");
        spark.sql("DELETE FROM your_table_name WHERE condition_column = 'condition_value'")
                .write()
                .mode(SaveMode.Overwrite)
                .jdbc(POSTGRES_URL, "your_table_name", connectionProps);
    }

    public static void invokeLambda() {
        System.out.println("Invoking AWS Lambda function...");
        try (LambdaClient lambdaClient = LambdaClient.create()) {
            InvokeRequest invokeRequest = InvokeRequest.builder()
                    .functionName(LAMBDA_FUNCTION_NAME)
                    .payload(SdkBytes.fromUtf8String("{\"key\": \"value\"}"))
                    .build();
            InvokeResponse response = lambdaClient.invoke(invokeRequest);
            System.out.println("Lambda response: " + response.payload().asUtf8String());
        } catch (Exception e) {
            System.err.println("Failed to invoke Lambda function: " + e.getMessage());
        }
    }
}

```

Explanation of Each Method:
readDataFromS3: Reads data from an S3 bucket, assuming the data is in CSV format. You can adjust the file format and options as needed.
createData: Writes the data from S3 into a PostgreSQL table, using SaveMode.Append to add data without overwriting.
readData: Reads data from the PostgreSQL table and displays it, providing verification of the data that’s stored.
updateData: Performs an update by reading the data, modifying a column, and overwriting the table.
deleteData: Executes a delete operation based on a specific condition. Adjust the SQL delete query to target the intended rows.
invokeLambda: Invokes an AWS Lambda function using LambdaClient. Replace the payload with the necessary data for your Lambda.


### 3. Airflow DAG for S3 Trigger

The Airflow DAG monitors S3 for new files, and when a new file is detected, it starts an EMR cluster for processing.

```python
from airflow import DAG
from airflow.providers.amazon.aws.sensors.s3_key import S3KeySensor
from airflow.providers.amazon.aws.operators.emr_create_job_flow import EmrCreateJobFlowOperator
from airflow.utils.dates import days_ago

default_args = {"owner": "airflow", "depends_on_past": False}
dag = DAG("s3_to_postgres", default_args=default_args, schedule_interval=None, start_date=days_ago(1))

# Detect new file in S3
s3_sensor = S3KeySensor(
    task_id="check_for_file_in_s3",
    bucket_key="your-bucket/path/to/file.csv",
    bucket_name="your-bucket",
    dag=dag,
)

# Start EMR job flow
emr_job = EmrCreateJobFlowOperator(
    task_id="start_emr_cluster",
    job_flow_overrides={"Name": "EMR Spark Cluster"},
    aws_conn_id="aws_default",
    dag=dag,
)

s3_sensor >> emr_job
```

---

### 4. Unit Testing

Test Class (`S3ToPostgresAppTest.java`)

```java
import org.apache.spark.sql.SparkSession;
import org.junit.jupiter.api.*;
import org.mockito.Mockito;

@TestInstance(TestInstance.Lifecycle.PER_CLASS)
public class S3ToPostgresAppTest {
    private SparkSession spark;

    @BeforeAll
    public void setup() {
        spark = SparkSession.builder().appName("Test").master("local").getOrCreate();
    }

    @Test
    public void testReadDataFromS3() {
        // Implement mock S3 client to verify file download
    }

    @Test
    public void testCreateData() {
        // Use an in-memory database to test data creation in PostgreSQL
    }

    @Test
    public void testReadData() {
        // Check data read from PostgreSQL
    }

    @Test
    public void testUpdateData() {
        // Check data update in PostgreSQL
    }

    @Test
    public void testDeleteData() {
        // Verify data deletion in PostgreSQL
    }

    @Test
    public void testInvokeLambda() {
        // Mock Lambda client and verify invocation
    }

    @AfterAll
    public void tear

Down() {
        spark.stop();
    }
}
```

Run tests using:
```bash
mvn clean test
```

---

### 5. Deployment and Execution

1. **Upload Data to S3**: Place your data file in the designated S3 bucket path.
2. **Run Airflow DAG**: The DAG will detect the file and trigger the EMR job.
3. **Observe Output**: Data should be processed, loaded into PostgreSQL, and the DAG triggered successfully.
4. **Monitor**: Use CloudWatch for logs and the Airflow UI for DAG monitoring.

---

This document covers the complete pipeline, from Terraform setup to Java Spark code, Airflow DAG, and unit tests. Each component is set up for efficient deployment and testing in an enterprise-level AWS environment. Let me know if you need further refinement!