
# README 

Use BigQuery Federated table with Partitioned data in Cloud Storage


## Step 1, copy sample data to Cloud Storage
```
gsutil cp data-20201031-102001.json gs://warm-actor-291222_cloudbuild/test-data/date=2020-10-31/
gsutil ls gs://warm-actor-291222_cloudbuild/test-data/date=2020-10-31/
# gs://warm-actor-291222_cloudbuild/test-data/date=2020-10-31/data-20201031-102001.json

bq mkdef --source_format NEWLINE_DELIMITED_JSON --hive_partitioning_mode=AUTO \
--hive_partitioning_source_uri_prefix=gs://warm-actor-291222_cloudbuild/test-data \
'gs://warm-actor-291222_cloudbuild/test-data/*'
```

This should print things like
```
Credential creation complete. Now we will select a default project.

List of projects:
  #       projectId         friendlyName    
 --- ------------------- ------------------ 
  1   warm-actor-291222   My First Project  
Found only one project, setting warm-actor-291222 as the default.

BigQuery configuration complete! Type "bq" to get started.

{
  "autodetect": true,
  "hivePartitioningOptions": {
    "mode": "AUTO",
    "sourceUriPrefix": "gs://warm-actor-291222_cloudbuild/test-data"
  },
  "sourceFormat": "NEWLINE_DELIMITED_JSON",
  "sourceUris": [
    "gs://warm-actor-291222_cloudbuild/test-data/*"
  ]
}
```
The JSON part is the schema.json we want to use

Run the command again to save it to schema.json
```
bq mkdef --source_format NEWLINE_DELIMITED_JSON --hive_partitioning_mode=AUTO \
--hive_partitioning_source_uri_prefix=gs://warm-actor-291222_cloudbuild/test-data \
'gs://warm-actor-291222_cloudbuild/test-data/*' > schema.json
```

Then run this to create table

```
bq mk --dataset data_test
# Dataset 'warm-actor-291222:data_test' successfully created.
bq mk --external_table_definition=schema.json data_test.users
# Table 'warm-actor-291222:data_test.users' successfully created.
```

Now we should see the new data_test dataset and `users` table in BigQuery

Run this to query the table
```
select *, _FILE_NAME file_name from data_test.users;
```

Try to copy another json file to a different partitioned folder
```
gsutil cp data-20201101-002001.json gs://warm-actor-291222_cloudbuild/test-data/date=2020-11-01/
```

Wait for a few seconds, run the query again
```
select *, _FILE_NAME file_name from data_test.users;
```
Should see 6 entries in result. (3 previous and 3 new ones)

The statement above doesn't use the partition column `date` so it's going to scan all the data.  
To avoid doing that, need to query like this:
```
select *, _FILE_NAME file_name from data_test.users
WHERE date >='2020-11-01';
```

If you use `bq` command CLI
```
bq query --use_legacy_sql=false 'select *, _FILE_NAME file_name from data_test.users;'
```


## Step 2, create view so we can extract data from the view easily

```
#standardSQL
CREATE OR REPLACE VIEW `data_test.view_users`
AS SELECT
  PARSE_TIMESTAMP('%Y%m%d', REGEXP_EXTRACT(_FILE_NAME, 'data-([0-9]+)-')) datehour
  , *
  , _FILE_NAME filename
FROM `data_test.users`;
```

This is to use the full _FILE_NAME to get the date time, if we only need the partition `date` 

We can just use `PARSE_TIMESTAMP(date)`, assuming those are UTC time. There are options for TimeZone if we need to convert

## Step 3, we might need to only allow user to query recent 7 days data so even we store 1 years of data, their queries won't have chance to scan all of our data and break our bank.

```
CREATE OR REPLACE VIEW `data_test.view_for_7days_report`
AS SELECT 
  *
FROM `data_test.users`
WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY );

```

Note, the `date` column is already using DATE format, so we don't need to use `FORMAT_DATE`  

```
WHERE date >= FORMAT_DATE('%Y-%m-%d', DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY ));
```

When we query the VIEW `data_test.view_for_7days_report`, it will only scan those 7 days data

We won't be able to scan the whole dataset by mistake.


## Step 4. We can create a few native BigQuery tables to store aggregated data

Some use-cases will be like a table to hold hourly errors status and another table to hold daily status.

The hourly one will be updated hourly or every a few hours, they only hold say 2 days of hourly data. Older than 2 days data will be removed from this table.

The daily one will be similar. It only holds cerntain days of data and will auto expire after a few days.

This will make the recent report very efficient, it only needs to query these native tables, they won't scan the files in Cloud Storage.

The benefits of the solution:
* We just need to dump data into the Cloud Storage, the data will be automatically availabe for BigQuery. There is no extra steps to process and load into BigQuery
* It can store years data without impacting performance and costs because we use `date` to partition them and our query will only scan the data in the range.
* We can setup Cloud Storage expiration rule, older than 3 month data will be moved to ColdLine to save costs. There is no extra steps to remove data from database tables.
* We can then process the data regularly to create report tables. Those tables will be much smaller because they only contain aggregated data for limited period. This makes our query reports much efficient and fast.
* This makes data reprocessing easier. If some raw data is corrupted, we don't need to clean up BigQuery database tables. We just need to remove that day's data on Cloud Storage and run collectors again, then the data is automatically availible for query.