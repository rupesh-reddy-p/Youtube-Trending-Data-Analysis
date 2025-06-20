# YouTube Data Analytics AWS Pipeline

This repository contains the code and configurations for an AWS-based data analytics pipeline designed to process, store, and analyze YouTube trending video statistics. The pipeline leverages various AWS services including S3, Lambda, Glue, Athena, and QuickSight to provide an end-to-end solution for data ingestion, transformation, and visualization.

## Table of Contents

  * [Architecture](https://www.google.com/search?q=%23architecture)
  * [Data Sources](https://www.google.com/search?q=%23data-sources)
  * [AWS Services Used](https://www.google.com/search?q=%23aws-services-used)
  * [Setup and Deployment](https://www.google.com/search?q=%23setup-and-deployment)
      * [IAM Roles and Users](https://www.google.com/search?q=%23iam-roles-and-users)
      * [S3 Buckets](https://www.google.com/search?q=%23s3-buckets)
      * [Data Upload (Raw Data)](https://www.google.com/search?q=%23data-upload-raw-data)
      * [Lambda Function (JSON to Parquet Conversion)](https://www.google.com/search?q=%23lambda-function-json-to-parquet-conversion)
      * [AWS Glue Data Catalog (Databases and Tables)](https://www.google.com/search?q=%23aws-glue-data-catalog-databases-and-tables)
      * [AWS Glue Crawlers](https://www.google.com/search?q=%23aws-glue-crawlers)
      * [AWS Glue ETL Jobs](https://www.google.com/search?q=%23aws-glue-etl-jobs)
      * [Athena for Querying](https://www.google.com/search?q=%23athena-for-querying)
      * [QuickSight for Visualization](https://www.google.com/search?q=%23quicksight-for-visualization)
  * [Code Explanation](https://www.google.com/search?q=%23code-explanation)
      * [`lambda_function.py`](https://www.google.com/search?q=%23lambda_functionpy)
      * [`pyspark_code.py`](https://www.google.com/search?q=%23pyspark_codepy)
  * [Monitoring and Troubleshooting](https://www.google.com/search?q=%23monitoring-and-troubleshooting)

## Architecture

The data pipeline follows a multi-stage approach:

1.  **Raw Data Ingestion**: Raw YouTube trending video data (CSV) and reference data (JSON) are uploaded to dedicated S3 buckets.
2.  **Cleansing/Standardization (Lambda)**: An S3-triggered Lambda function processes the raw JSON data, normalizes it, and converts it into Parquet format, storing it in a "cleansed" S3 layer.
3.  **ETL Transformation (Glue)**: AWS Glue ETL jobs read from the raw data layer (or cleansed layer), apply transformations (e.g., schema mapping, joins, filtering by region), and write the refined data to a "curated" or "analytics" S3 layer in Parquet format, often partitioned.
4.  **Data Cataloging (Glue Data Catalog & Crawlers)**: AWS Glue Crawlers automatically discover schemas and partitions from the data in S3 buckets and update the Glue Data Catalog. This catalog serves as a central metadata repository for all data in the pipeline, defining databases and tables.
5.  **Ad-hoc Querying (Athena)**: AWS Athena is used for interactive querying of the data directly in S3, leveraging the Glue Data Catalog for schema information.
6.  **Business Intelligence (QuickSight)**: AWS QuickSight connects to the data cataloged in Glue (via Athena or direct S3 connection) to create interactive dashboards and visualizations for business analysis.

## Data Sources

The pipeline processes YouTube trending video data, which typically includes:

  * **Raw Statistics Data**: CSV files containing video metadata, views, likes, dislikes, comment counts, etc., often partitioned by `region` (e.g., `CAvideos.csv` for Canada).
  * **Reference Data**: JSON files containing category ID mappings for different regions (e.g., `CA_category_id.json`).

## AWS Services Used

  * **Amazon S3**: For scalable and durable storage of raw, cleansed, and processed data.
  * **AWS Lambda**: For serverless, event-driven processing of raw JSON data into Parquet.
  * **AWS Glue**:
      * **Data Catalog**: A persistent metadata store for all data.
      * **Crawlers**: To automatically discover and update schema information in the Data Catalog.
      * **ETL Jobs**: For serverless data transformation using PySpark.
  * **Amazon Athena**: For interactive query analysis directly on data in S3 using standard SQL.
  * **Amazon QuickSight**: For building interactive dashboards and performing business intelligence on the processed data.
  * **AWS IAM**: For managing access and permissions for AWS services and users.

## Setup and Deployment

### IAM Roles and Users

Ensure you have the necessary IAM roles and users configured for the services to interact with each other and S3.

  * **`rupesh-test-account1`**: An IAM user for console access and managing resources.
  * **`youtube-on-de-glue-s3-role`**: An IAM role for AWS Glue jobs to access S3 buckets.
  * **`youtube-on-de-raw-useast1-s3-lambda-role`**: An IAM role for the Lambda function to access S3 buckets and perform operations.

### S3 Buckets

Create the following S3 buckets in the `us-east-1` region:

  * `youtube-on-de-raw-useast1-dev`: For raw input data.
      * `youtube-on-de-raw-useast1-dev/youtube/raw_statistics_reference_data/`: To store JSON reference files.
      * `youtube-on-de-raw-useast1-dev/youtube/raw_statistics/`: To store raw CSV video statistics, partitioned by region (e.g., `region=ca/CAvideos.csv`).
  * `youtube-on-de-cleansed-useast1-dev`: For cleansed data in Parquet format.
  * `youtube-on-de-analytics-useast1-dev`: For final analytical data, partitioned by region and category ID.
  * `youtube-on-de-raw-useast1-athena-job`: For Athena query results and temporary data.
  * `aws-glue-assets-XXXXXXXXXXXX-useast1`: Default Glue assets bucket (auto-created).

### Data Upload (Raw Data)

Upload your raw CSV video data and JSON reference data to the `youtube-on-de-raw-useast1-dev` bucket using the AWS CLI or console. The following `aws s3 cp` commands demonstrate the typical structure:

```bash
# Upload reference JSON files
aws s3 cp CA_category_id.json s3://youtube-on-de-raw-useast1-dev/youtube/raw_statistics_reference_data/CA_category_id.json --exclude "*" --include "*.json"
# Repeat for other region JSON files (DE, FR, GB, IN, JP, KR, MX, RU, US)

# Upload raw CSV video files, partitioned by region
aws s3 cp CAvideos.csv s3://youtube-on-de-raw-useast1-dev/youtube/raw_statistics/region=ca/CAvideos.csv
# Repeat for other region CSV files (DE, FR, GB, IN, JP, KR, MX, RU, US)
```

### Lambda Function (JSON to Parquet Conversion)

1.  **Create Lambda Function**: Create a new Lambda function named `youtube-on-de-raw-useast1-lambda-json-parquet`.
2.  **Runtime**: Python 3.x.
3.  **Execution Role**: Use the `youtube-on-de-raw-useast1-s3-lambda-role`.
4.  **Code**: Upload the `lambda_function.py` script.
5.  **Environment Variables**: Configure the following environment variables:
      * `s3_cleansed_layer`: `s3://youtube-on-de-cleansed-useast1-dev/youtube/raw_statistics/`
      * `glue_catalog_db_name`: `db_youtube_cleaned`
      * `glue_catalog_table_name`: `raw_statistics`
      * `write_data_operation`: `append` (or `overwrite` as needed)
6.  **Trigger**: Add an S3 trigger to the `youtube-on-de-raw-useast1-dev` bucket, specifically for the `youtube/raw_statistics_reference_data/` prefix, for `PUT` events. This will automatically convert new JSON reference data to Parquet.

### AWS Glue Data Catalog (Databases and Tables)

The Glue Data Catalog will store metadata for your data in S3.

1.  **Databases**: Verify the existence or create the following databases:
      * `de_youtube_raw`
      * `db_youtube_cleaned`
      * `db_youtube_analytics`
2.  **Tables**: Tables will be created by Glue Crawlers or Glue ETL jobs. Expected tables include:
      * `raw_statistics` (in `de_youtube_raw` for CSV, and `db_youtube_cleaned` for Parquet)
      * `raw_statistics_reference_data` (in `de_youtube_raw`)
      * `final_analytics` (in `db_youtube_analytics`)

### AWS Glue Crawlers

Set up Glue Crawlers to discover schema and partitions for your S3 data and populate the Glue Data Catalog.

1.  **Create Crawlers**:
      * **Crawler 1 (Raw CSV)**: Point to `s3://youtube-on-de-raw-useast1-dev/youtube/raw_statistics/`.
          * Output database: `de_youtube_raw`.
          * Table name: `raw_statistics` (CSV).
      * **Crawler 2 (Raw JSON Reference)**: Point to `s3://youtube-on-de-raw-useast1-dev/youtube/raw_statistics_reference_data/`.
          * Output database: `de_youtube_raw`.
          * Table name: `raw_statistics_reference_data`.
      * **Crawler 3 (Cleansed Parquet)**: Point to `s3://youtube-on-de-cleansed-useast1-dev/youtube/raw_statistics/`.
          * Output database: `db_youtube_cleaned`.
          * Table name: `raw_statistics` (Parquet).
2.  **Run Crawlers**: Execute the crawlers to populate your Glue Data Catalog.

### AWS Glue ETL Jobs

Create and configure Glue ETL jobs for data transformations.

1.  **`youtube-on-de-cleansed-csv-to-parquet`**:
      * This job is expected to take CSV data from the raw layer and convert it to Parquet in the cleansed layer. Although its visual representation is not fully provided, it is listed as a Glue ETL job.
2.  **`youtube-on-de-parquet-analytics-version`**:
      * **Type**: Spark ETL Job.
      * **Script**: Upload `pyspark_code.py`.
      * **Visual ETL (Diagram)**:
          * **Data Source**: `de_youtube_raw.raw_statistics` (filtered by `region in ('ca','gb','us')`).
          * **Transformation**: ApplyMapping to ensure correct data types.
          * **Data Target (S3)**: `s3://youtube-on-de-cleansed-useast1-dev/youtube/raw_statistics/`.
          * **Format**: Glue Parquet.
          * **Compression**: Snappy.
          * **Partition Keys**: `region`.
          * **Data Catalog Update Options**: Create a table in the Data Catalog and on subsequent runs, update the schema and add new partitions.
          * **Database**: `db_youtube_analytics`.
      * **Execution Role**: Assign `youtube-on-de-glue-s3-role`.
      * **Run the job** as needed to process the data.

### Athena for Querying

After your data is cataloged in Glue, you can query it using Athena.

1.  **Open Athena**: Navigate to the Athena console.

2.  **Select Database**: Choose `db_youtube_analytics` or `db_youtube_cleaned`.

3.  **Run Queries**: Execute SQL queries against your tables. For example:

    ```sql
    SELECT * FROM "db_youtube_analytics"."final_analytics" limit 10;
    ```

### QuickSight for Visualization

1.  **Access QuickSight**: Log in to your Amazon QuickSight account.
2.  **Create New Analysis**:
3.  **Add Data Source**: Select "Athena" and choose your Glue Data Catalog database (e.g., `db_youtube_analytics`).
4.  **Choose Table**: Select the `final_analytics` table.
5.  **Visualize Data**: Drag and drop fields to create visualizations. Examples include:
      * **Count of Likes by Region**: Bar chart (e.g., `ca`, `gb`, `us` regions).
      * **Count of Views by Snippet\_title**: Pie chart.
      * **Count of Views by Region**: Map visualization.
      * **Count of Comment\_count by Snippet\_title**: Line chart.

## Code Explanation

### `lambda_function.py`

This Python script is designed to be deployed as an AWS Lambda function, triggered by S3 object creation events.

```python
import awswrangler as wr
import pandas as pd
import urllib.parse
import os

# Environment variables for configuration
os_input_s3_cleansed_layer = os.environ['s3_cleansed_layer']
os_input_glue_catalog_db_name = os.environ['glue_catalog_db_name']
os_input_glue_catalog_table_name = os.environ['glue_catalog_table_name']
os_input_write_data_operation = os.environ['write_data_operation']

def lambda_handler(event, context):
    # Extract bucket and key from the S3 event
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'], encoding='utf-8')
    try:
        # Read JSON data from the S3 object using awswrangler
        df_raw = wr.s3.read_json('s3://{}/{}'.format(bucket,key))
        # Normalize the 'items' array in the JSON to flatten the structure into a DataFrame
        df_step_1 = pd.json_normalize(df_raw['items'])
        # Write the DataFrame to S3 as Parquet, registering it in Glue Data Catalog
        wr_response = wr.s3.to_parquet(
            df= df_step_1,
            path=os_input_s3_cleansed_layer, # S3 path for cleansed data
            dataset=True, # Treat as a dataset for partitioning/schema evolution
            database=os_input_glue_catalog_db_name, # Glue database name
            table=os_input_glue_catalog_table_name, # Glue table name
            mode=os_input_write_data_operation # Write mode (e.g., 'append', 'overwrite')
            )
        return wr_response
    except Exception as e:
        print(e)
        print('Error getting object {} from bucket {}. Make sure they exist and your bucket is in the same region as this function.'.format(key,bucket))
        raise e

```

**Functionality**:

  * **Triggered by S3**: This Lambda function is invoked when a new object (presumably a JSON file) is uploaded to a specified S3 location.
  * **Reads JSON**: It uses `awswrangler` to read the JSON file directly from S3.
  * **Normalizes Data**: `pd.json_normalize(df_raw['items'])` is crucial for flattening nested JSON structures, specifically extracting data from the 'items' array.
  * **Writes Parquet to S3**: The flattened data is then written back to S3 in Parquet format, a columnar storage format optimized for analytical queries.
  * **Registers with Glue Catalog**: It automatically registers the new Parquet data in the AWS Glue Data Catalog, defining the schema and making it queryable by services like Athena.
  * **Error Handling**: Includes basic error handling for S3 object access.

### `pyspark_code.py`

This PySpark script is designed to run as an AWS Glue ETL job.

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame

args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Predicate pushdown for filtering data at the source
predicate_pushdown = "region in ('ca','gb','us')"

# Script generated for node AWS Glue Data Catalog
# Reads raw_statistics from de_youtube_raw database, applying predicate pushdown
AWSGlueDataCatalog_node1721164954747 = glueContext.create_dynamic_frame.from_catalog(database="de_youtube_raw", table_name="raw_statistics", transformation_ctx="AWSGlueDataCatalog_node1721164954747", push_down_predicate = predicate_pushdown)

# Script generated for node Change Schema
# Applies schema mapping to ensure correct data types
ChangeSchema_node1721165002070 = ApplyMapping.apply(frame=AWSGlueDataCatalog_node1721164954747, mappings=[("video_id", "string", "video_id", "string"), ("trending_date", "string", "trending_date", "string"), ("title", "string", "title", "string"), ("channel_title", "string", "channel_title", "string"), ("category_id", "long", "category_id", "bigint"), ("publish_time", "string", "publish_time", "string"), ("tags", "string", "tags", "string"), ("views", "long", "views", "bigint"), ("likes", "long", "likes", "bigint"), ("dislikes", "long", "dislikes", "bigint"), ("comment_count", "long", "comment_count", "bigint"), ("thumbnail_link", "string", "thumbnail_link", "string"), ("comments_disabled", "boolean", "comments_disabled", "boolean"), ("ratings_disabled", "boolean", "ratings_disabled", "boolean"), ("video_error_or_removed", "boolean", "video_error_or_removed", "boolean"), ("description", "string", "description", "string"), ("region", "string", "region", "string")], transformation_ctx="ChangeSchema_node1721165002070")

# Coalesce to 1 file (optional, often done for smaller datasets or specific target system requirements)
datasink1 = ChangeSchema_node1721165002070.toDF().coalesce(1)
df_final_output = DynamicFrame.fromDF(datasink1, glueContext, "df_final_output")

# Script generated for node Amazon S3
# Writes the transformed data to S3 in Glue Parquet format, partitioned by region
AmazonS3_node1721165208226 = glueContext.write_dynamic_frame.from_options(frame=ChangeSchema_node1721165002070, connection_type="s3", format="glueparquet", connection_options={"path": "s3://youtube-on-de-cleansed-useast1-dev/youtube/raw_statistics/", "partitionKeys": ["region"]}, format_options={"compression": "snappy"}, transformation_ctx="AmazonS3_node1721165208226")

job.commit()
```

**Functionality**:

  * **Initializes Glue Job**: Standard Glue job setup.
  * **Reads from Glue Catalog**: Reads data from the `raw_statistics` table within the `de_youtube_raw` database.
  * **Predicate Pushdown**: Optimizes data retrieval by filtering `region` at the source to include only 'ca', 'gb', and 'us'.
  * **Schema Mapping**: Applies a schema transformation to ensure correct data types for each column, which is essential for consistent data quality and downstream consumption.
  * **Writes to S3 (Cleansed Layer)**: Writes the transformed data to the `youtube-on-de-cleansed-useast1-dev` S3 bucket in `glueparquet` format.
  * **Partitioning**: The output data is partitioned by `region`, which improves query performance in Athena and QuickSight.
  * **Compression**: Uses Snappy compression for efficient storage and faster reads.
  * **Coalesce**: The `coalesce(1)` operation attempts to reduce the number of output files to one, which can be useful for smaller datasets to avoid small file problems in data lakes.

## Monitoring and Troubleshooting

  * **AWS CloudWatch Logs**: Monitor Lambda function logs and Glue job logs in CloudWatch for errors and performance issues.
  * **AWS Glue Job Runs**: Check the "Runs" tab for your Glue ETL jobs to see status, duration, and logs.
  * **S3 Bucket Contents**: Periodically inspect S3 buckets to ensure data is being written as expected at each stage (raw, cleansed, analytics).
  * **Athena Query History**: Review Athena query history for failed queries and their error messages.
  * **Glue Data Catalog**: Verify that crawlers are successfully updating table schemas and partitions in the Glue Data Catalog.
