# youtube-data-analysis

### Main goal

The main goal was to launch data-driven campaign.

### Subgoals

1. Design and build a new Data Lake architecture
2. Create ETL design and data pipelines for efficient data processing
3. Create a BI tier and build a dashboard
4. Build entire infrastructure in AWS



### Main advertising channel

YouTube was selected as main advertiseing channel.

#### Why Youtube?

YouTube is second most visited website in the entire world.&#x20;

<figure><img src=".gitbook/assets/image.png" alt=""><figcaption><p><a href="https://www.visualcapitalist.com/the-50-most-visited-websites-in-the-world/">https://www.visualcapitalist.com/the-50-most-visited-websites-in-the-world/</a></p></figcaption></figure>

### Initial questions to answer:&#x20;

* How to categorise videos, based on their comments and statistics.
* What factors affect how popular a YouTube video will be.

### Used dataset

<img src=".gitbook/assets/image (3).png" alt="" data-size="line"> Dataset was taken from here:&#x20;

[https://www.kaggle.com/datasets/datasnaek/youtube-new](https://www.kaggle.com/datasets/datasnaek/youtube-new)&#x20;

For future development there is an option to take this one that updates daily:

[https://www.kaggle.com/datasets/rsrishav/youtube-trending-video-dataset](https://www.kaggle.com/datasets/rsrishav/youtube-trending-video-dataset)

### Architecture of the project

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption><p>Data Flow diagram</p></figcaption></figure>





### Data Lake

Project required to create tree s3 buckets.

#### Landing area

Landing area - is a bucket for raw data from dataset.

Landing bucket has next structure:

<pre><code>&#x3C;!---
<strong>Here will be a tree of s3 bucket
</strong>-->
</code></pre>



#### Cleansed/Enriched&#x20;

Cleansed/Enriched - bucket for cleansed data after processing through "Data processing" layer.

Cleansed/Enriched bucket has next structure:

<pre><code>&#x3C;!---
<strong>Here will be a tree of s3 bucket
</strong>-->
</code></pre>

#### Analytics/Reporting

Analytics/Reporting  - is bucket for data that is ready for BI proces

Analytics/Reporting bucket has next structure:

<pre><code>&#x3C;!---
<strong>Here will be a tree of s3 bucket
</strong>-->
</code></pre>



### My development proces:

First of all an IAM user with the required permissions has been created. All further actions was made from this user account.

```
<!---
Here will be a configuration screen 
-->
```

Then "Landing Area" s3 bucket need to be created. Landing area is just a bucket for raw data. \
The dataset was previously downloaded and saved in the project folder. This data then had to be copied to the s3 bucket. To do this, I ran the following commands:

```shell
cd /.../data_directory

# Copy all .json files in directory to the selected path of the bucket
aws s3 cp . s3://<bucket_name>/youtube/raw_statistics_reference_data/ \
    --recursive --exclude "*" --include "*.json" 

# Copy each .csv file in separate folder
aws s3 cp CAvideos.csv s3://<bucket_name>/youtube/raw_statistics/region=ca/
aws s3 cp DEvideos.csv s3://<bucket_name>/youtube/raw_statistics/region=de/
aws s3 cp FRvideos.csv s3://<bucket_name>/youtube/raw_statistics/region=fr/
aws s3 cp GBvideos.csv s3://<bucket_name>/youtube/raw_statistics/region=gb/
aws s3 cp INvideos.csv s3://<bucket_name>/youtube/raw_statistics/region=in/
aws s3 cp JPvideos.csv s3://<bucket_name>/youtube/raw_statistics/region=jp/
aws s3 cp KRvideos.csv s3://<bucket_name>/youtube/raw_statistics/region=kr/
aws s3 cp MXvideos.csv s3://<bucket_name>/youtube/raw_statistics/region=mx/
aws s3 cp RUvideos.csv s3://<bucket_name>/youtube/raw_statistics/region=ru/
aws s3 cp USvideos.csv s3://<bucket_name>/youtube/raw_statistics/region=us/
```



After this step I needed to set up AWS Glue.

1. A database "youtube\_data\_analysis\_raw" in Glue was created
2. A crawler "...-youtube-data-analysis-raw-glue-1" was created with next settings:
   1. As data sourse i choosed s3 bucket path to .json files: s3://\<bucket\_name>/youtube/raw\_statistics\_reference\_data/
   2. A new security group for crawler with required permissions was created and selected
   3. A new Glue db was selected for output
3.  &#x20;Crawler created a new table with next schema:

    ```json
    [ 
        { "Name": "kind", "Type": "string" }, 
        { "Name": "etag", "Type": "string" }, 
        { 
            "Name": "items", 
            "Type": "array<struct<kind:string,etag:string,id:string,snippet:struct<channelId:string,title:string,assignable:boolean>>>" 
        } 
    ]
    ```
4.  After that I noticed that files .json in dataset have this structure:

    ```json
    {
     "kind": "string",
     "etag": "string",
     "items": [
      {
       "kind": "string",
       "etag": "string",
       "id": "int",
       "snippet": {
        "channelId": "string",
        "title": "string",
        "assignable": "Boolean"
       }
      },
      {
      ...
       },
       {
       ...
       }
      },
    ```
5. So i decided to process data through AWS lambda to get .parquet files instead of json

I needed to build a pipeline that will cleansing data from semi-structured to structured form.

<figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption><p>Data pipeline diagram</p></figcaption></figure>

To do this a Lambda function was created with next properties:

1. Name - youtube-data-analysis-lambda-json-to-parquet
2. Langueage - Python 3.9
3. Architecture - x86\_64
4. Also new role with required permissions was created

After this, new enviroment variables were added:

<figure><img src=".gitbook/assets/image (1).png" alt=""><figcaption><p>Enviroment variables (s3_cleansed_layer value was changed in web developer tools for this screenshot)</p></figcaption></figure>

Next step was to write a python script:

```python
import awswrangler as wr
import pandas as pd
import urllib.parse
import os

# Temporary hard-coded AWS Settings; i.e. to be set as OS variable in Lambda
os_input_s3_cleansed_layer = os.environ['s3_cleansed_layer']
os_input_glue_catalog_db_name = os.environ['glue_catalog_db_name']
os_input_glue_catalog_table_name = os.environ['glue_catalog_table_name']
os_input_write_data_operation = os.environ['write_data_operation']


def lambda_handler(event, context):
    # Get the object from the event and show its content type
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'], encoding='utf-8')
    try:

        # Creating DF from content
        df_raw = wr.s3.read_json('s3://{}/{}'.format(bucket, key))

        # Extract required columns:
        df_step_1 = pd.json_normalize(df_raw['items'])

        # Write to S3
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
        print('Error getting object {} from bucket {}. Make sure they exist and your bucket is in the same region as this function.'.format(key, bucket))
        raise e
```
