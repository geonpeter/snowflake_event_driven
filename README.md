# snowflake_event_driven (Coding)

## Event driven data ingestion in snowflake table using SnowPipe (Tech Stack Used : Google Storage Bucket, GCP Pub-Sub, Snowflake)
      -> Create storage integration in snowflake to read data from external Google Storage Bucket
      -> Create GCP Pub-Sub topic to send notification into it as soon as file is uploaded in bucket
      -> Setup notification subscription between Bucket & Pub-Sub
      -> Create notification integration in snowflake
      -> Create external stage in snowflake
      -> Create SnowPipe to load data from external stage to snowflake target table


