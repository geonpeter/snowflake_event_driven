# Snowflake - GCP Snowpipe Integration

This guide outlines the steps for setting up a Snowpipe integration between Snowflake and Google Cloud Storage (GCS) using Pub/Sub for event-driven data ingestion.

## Prerequisites

- **Snowflake Account** with `ACCOUNTADMIN` role.
- **Google Cloud Platform (GCP)**:
  - A GCS bucket with proper permissions.
  - Pub/Sub service enabled.
  - A service account with access to GCS and Pub/Sub.

---
# Snowflake - GCP Snowpipe Integration

This guide outlines the steps for setting up a Snowpipe integration between Snowflake and Google Cloud Storage (GCS).

## Architecture Diagram
![Diagram of Snowflake Integration](https://github.com/user-attachments/assets/1086c7f8-5218-41f4-942e-9404bcaac083)


## Steps to Set Up Snowpipe Integration

### 1. Use Snowflake Role

Switch to the `ACCOUNTADMIN` role to ensure you have the required privileges for creating integrations and pipes.

```sql
-- Use role
USE ROLE ACCOUNTADMIN;
```
### 2. Create Snowflake Database and Table

Create the database and table in Snowflake where the data from GCS will be ingested.

```sql
-- Create database
CREATE OR REPLACE DATABASE snowpipe_demo;

-- Create table 
CREATE OR REPLACE TABLE orders_data_lz(
    order_id INT,
    product VARCHAR(20),
    quantity INT,
    order_status VARCHAR(30),
    order_date DATE
);

```
### 3. Create Storage Integration with GCS

Create a storage integration that allows Snowflake to securely access the GCS bucket.

```sql
-- Create a Cloud Storage Integration in Snowflake
CREATE OR REPLACE STORAGE INTEGRATION gcs_bucket_read_int
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = GCS
  ENABLED = TRUE
  STORAGE_ALLOWED_LOCATIONS = ('gcs://snowpipe_raw_data/');
```

### 4. Retrieve Service Account for GCS Access
Describe the integration to get the service account details. This service account will be used to grant permissions in GCP.
```sql
-- Describe the integration to get service account details
DESC STORAGE INTEGRATION gcs_bucket_read_int;
```

### 5. Create External Stage in Snowflake
Create an external stage that references the GCS bucket where the data will be uploaded.
```sql
-- Create a stage in Snowflake pointing to the GCS bucket
CREATE OR REPLACE STAGE snowpipe_stage
  URL = 'gcs://snowpipe_raw_data/'
  STORAGE_INTEGRATION = gcs_bucket_read_int;
```
You can check the available stages with:
```sql
-- Show available stages
SHOW STAGES;

-- List the files in the stage
LIST @snowpipe_stage;
```

### 6. Create GCP Pub/Sub Topic and Subscription
In GCP, create a Pub/Sub topic and subscription to handle file upload notifications from the GCS bucket.
```bash
# Create a Pub/Sub topic in GCP
gsutil notification create -t snowpipe_pubsub_topic -f json gs://snowpipe_raw_data/
```

### 7. Create Notification Integration in Snowflake
Create a notification integration in Snowflake to listen to GCP Pub/Sub messages, triggering Snowpipe when new files are uploaded.
```sql
-- Create a notification integration to connect with Pub/Sub
CREATE OR REPLACE NOTIFICATION INTEGRATION notification_from_pubsub_int
  TYPE = QUEUE
  NOTIFICATION_PROVIDER = GCP_PUBSUB
  ENABLED = TRUE
  GCP_PUBSUB_SUBSCRIPTION_NAME = 'projects/your-project-id/subscriptions/snowpipe_pubsub_topic-sub';
```
Describe the notification integration to confirm the setup:
```sql
-- Describe the notification integration
DESC INTEGRATION notification_from_pubsub_int;
```

### 8. Create Snowpipe for Automated Data Ingestion
Create the Snowpipe that automatically ingests data from the GCS bucket into the Snowflake table upon receiving a Pub/Sub notification.
```sql
-- Create a Snowpipe to ingest data into the orders_data_lz table
CREATE OR REPLACE PIPE gcs_to_snowflake_pipe
  AUTO_INGEST = TRUE
  INTEGRATION = notification_from_pubsub_int
  AS
  COPY INTO orders_data_lz
  FROM @snowpipe_stage
  FILE_FORMAT = (TYPE = 'CSV');
```
Check the available pipes with:
```sql
-- Show all pipes
SHOW PIPES;
```

### 9. Monitor Snowpipe Activity
You can check the status of the pipe using the following query:
```sql
-- Check the status of the pipe
SELECT SYSTEM$PIPE_STATUS('gcs_to_snowflake_pipe');
```
You can also view the history of ingested data:
```sql
-- Check ingestion history
SELECT * 
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(TABLE_NAME => 'orders_data_lz', START_TIME => DATEADD(hours, -1, CURRENT_TIMESTAMP())));
```

### 10.  Drop the Snowpipe 
To stop the pipe, you can drop it:
```sql
-- Drop the Snowpipe
DROP PIPE gcs_to_snowflake_pipe;
```

