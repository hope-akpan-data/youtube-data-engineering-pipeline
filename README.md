# YouTube Data Engineering Pipeline (AWS)

This raw YouTube data came in messy files that couldn’t be queried, shared, or automated. I built a cloud system that turns those files into clean, searchable tables analysts can use instantly with SQL.
Before this system:
- The data was in CSV and JSON files
- Hard to clean
- Hard to analyze
- Hard to automate
After this system:
- The data is clean and structured
- Easy to query using SQL
- Automatically updated
- Ready for visualizations and reports

# Tools Used (Only What I Actually Used)
- Amazon S3 : Cloud storage for raw and cleaned data
- AWS Lambda :  Serverless data processing
- AWS Glue : Data catalog and crawlers
- Amazon Athena : SQL querying directly on S3
- AWS IAM : Security and access control
- AWS SDK for Pandas (awswrangler) : Writing between Pandas, S3, and Glue
- Python(pandas)

# Real Business Problem
Companies often receive data in different formats with messy structures and mostly as large files. Resulting in analysts being unable to answer questions like:
   - Which category gets the most views?
   - How do trends change by region?
   - Which videos perform best?
This pipeline solves that by cleaning the data automatically and storing it in a fast, queryable format.

# Dataset
Source: Kaggle ; YouTube Trending Videos Dataset
Includes:
- Regional CSV files (views, likes, comments, etc.)
- JSON category metadata


# Step 1: Securing My AWS Account
Before building anything:
- I created an IAM user
- Logged out of the root account
- Gave only the permissions needed
- Used IAM roles for services instead of hardcoded credentials
This protects the account and follows professional cloud security standards.

# Step 2: Storing Raw Data in Amazon S3
I downloaded the YouTube Trending Videos Dataset from Kaggle.
The dataset included:
- CSV files → video statistics (views, likes, comments, etc.)
- JSON files → category metadata (Music, Gaming, Sports, etc.)

Keeping these files on my laptop was bad for:
- Automation
- Scaling
- Sharing
So I stored them in Amazon S3, which is cloud storage that works well with analytics tools. 

# These are the exact buckets I created for this project:
Bucket Name                                Region              Purpose

de-youtube-raw-us-east2-devenvironment     US East (Ohio)      Stores raw CSV and JSON files
de-youtube-raw-us-east2-athena-query1      US East (Ohio)     Stores Athena query results 
de-youtube-cleansed-us-east2-devenvironment US East (Ohio)    Stores cleaned Parquet files

 # S3 Folder Structure:
de-youtube-raw-us-east2-devenvironment/
└── youtube/
    ├── raw_statistics_reference_data/
    │   └── *.json
    └── raw_statistics/
        └── region=xx/
            └── *.csv

de-youtube-cleansed-us-east2-devenvironment/
└── youtube/
    └── *.parquet

N.B;This separation makes it easy to:
Know what data is raw vs clean and track pipeline stages

# Step 3: Creating IAM Roles for AWS Services
Before building automation, I created IAM roles for
AWS Lambda and AWS Glue

These roles allowed:
- Reading raw data from S3
- Writing cleaned data to S3
- Creating Glue tables
- Deleting old Parquet files when overwriting
I avoided adding credentials anywhere.

# Step 4: Automating Data Cleaning with AWS Lambda
The JSON category files were deeply nested and hard to analyze directly.
Instead of cleaning them manually, I built an AWS Lambda function, a cloud program that runs automatically when triggered.
This Lambda function:
- Reads a JSON file from S3
- Flattens nested structures using Pandas
- Converts the data into Parquet format
- Writes cleaned data back to S3
- Registers the table in AWS Glue

# Why I used AWS Lambda
Because:
- No servers to manage
- Runs only when needed
- Scales automatically

# Why I converted to Parquet
Because Parquet:
- Queries faster than CSV/JSON
- Uses less storage
- Works well with AWS Athena and Glue

# Lambda Core Logic (Actual Code Used)
import awswrangler as wr
import pandas as pd
import urllib.parse
import os

os_input_s3_cleansed_layer = os.environ['s3_cleansed_layer']
os_input_glue_catalog_db_name = os.environ['glue_catalog_db_name']
os_input_glue_catalog_table_name = os.environ['glue_catalog_table_name']
os_input_write_data_operation = os.environ['write_data_operation']

def lambda_handler(event, context):
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'], encoding='utf-8')

    try:
        df_raw = wr.s3.read_json(f"s3://{bucket}/{key}")
        df_step_1 = pd.json_normalize(df_raw["items"])

        wr_response = wr.s3.to_parquet(
            df=df_step_1,
            path=os_input_s3_cleansed_layer,
            dataset=True,
            database=os_input_glue_catalog_db_name,
            table=os_input_glue_catalog_table_name,
            mode=os_input_write_data_operation
        )

        return wr_response
    except Exception as e:
        print(e)
        raise e

# Step 5: Organizing Data with AWS Glue
After cleaning the data, I needed a way to know what tables exist, view their columns and understand data types.

So I used AWS Glue, which works like a dictionary for data.
I created two Glue databases:
- de_youtube_raw
- de_youtube_cleaned
Then I ran Glue Crawlers to scan my S3 folders, detect schemas and create tables automatically

# Step 6: Querying the Data with Amazon Athena
Once the data was clean and cataloged, I used Amazon Athena to run SQL directly on S3 without creating any database servers.
Example query:
SELECT *
FROM "AwsDataCatalog"."de_youtube_cleaned"."cleaned_statistics_data"
LIMIT 10;

Filtering example:
SELECT *
FROM "AwsDataCatalog"."de_youtube_cleaned"."cleaned_statistics_data"
WHERE snippet_title = 'Gaming'
LIMIT 10;
Athena returned structured results instantly.

# Why I used Athena
Because:
- No database setup
- No servers
- Works directly on S3
- Fast and pay-per-query
- Perfect for analysis

# IAM & Security Setup
I created IAM roles for:
- AWS Lambda
- AWS Glue

# Permissions included:
- Reading raw data from S3
- Writing cleaned data to S3
- Creating Glue tables
- Deleting old Parquet files during overwrite operations
I avoided using the root account and worked entirely from IAM users and roles.

# Debugging Challenges I faced and Solved

❌ Issue 1 : Lambda couldn’t import awswrangler
Error: No module named 'awswrangler'
  Fix: Added AWS SDK for Pandas as a Lambda layer

❌ Issue 2 : Lambda timing out
Error: Task timed out after 183 seconds
  Fix: Increased Lambda memory and timeout

❌ Issue 3 : S3 Access Denied (403)
Error: An error occurred (403) when calling the HeadObject operation
  Fix: Added correct S3 permissions to the Lambda execution role

❌ Issue 4 : DeleteObject AccessDenied
Error: Not authorized to perform: s3:DeleteObject
  Fix: Added delete permissions since awswrangler overwrites Parquet files

❌ Issue 5 : Glue database not found
Error: Database de_youtube_cleaned not found
  Fix: Created the Glue database manually before running Lambda
  
❌ Issue 6 : Lambda ran but no output appeared
Cause: Incorrect S3 output path in environment variables
  Fix: Corrected the S3 URI format

# Lambda Environment Variables Used
Configured in AWS: 
Variable                  Purpose        
s3_cleansed_layer         Output S3 path for Parquet files
glue_catalog_db_name      Glue database name
glue_catalog_table_name   Glue table name
write_data_operation      Write mode (append/overwrite)


 # Example Output
Cleaned Parquet files stored in:
s3://de-youtube-cleansed-us-east2-devenvironment/youtube/

Glue table:
de_youtube_cleaned.cleaned_statistics_data

Athena successfully returned structured YouTube category data.

 # Business Value
This system:
✅ Removes manual data cleaning
 ✅ Makes data easy to search
 ✅ Speeds up reporting
 ✅ Works automatically when new data arrives
 ✅ Scales as data grows
 ✅ Stores data cheaply and queries it fast
 ✅ Turns raw files into analytics-ready datasets



