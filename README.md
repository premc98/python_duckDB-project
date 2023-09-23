# python_duckDB-project
# Problem Statement

- The tracker folders in perform-{env}-atomicduck/{site}/{tracker} has multiple duckdb files named *.duckdb and *.new.duckdb
- The *.duckdb files are compiled with the duckdb release 0.6.1 and the *.new.duckdb files are compiled with the 0.7.1 release.
- We need to migrate to the latest duckdb release which is 0.8.1

## Solution

- Use the existing script created by Fisher and  modify it to use the latest duckdb release 0.8.1
- Test if the new duckdb file has the indexes in them after running the EXPORT/IMPORT
- Come up with a concurrent strategy using docker to run the tasks
- This way we can save time when we have to migrate thousands of trackers


## Design:

### Publisher
- Get all the tracked ID for the trackers that need to be migrated from Postgres goal table
- Depending on the size of each .duckdb file, queue them into different queues on Rabbit MQ, namely (migrate_duckdb_release for file_size < 1.5GB, migrate_duckdb_release_big for file_size > 1.5GB )
- Using the tracker ids from postgres, we create the following message based on the requirement

**S3 to S3** -> migrate_duck_db_s3-s3_publisher.py
```json
{
  "source_datastore": "s3",
  "destination_datastore": "s3",
  "source_bucket": "perform-dev-raw",
  "destination_bucket": "perform-dev-atomicduck",
  "tracker-id": "9324670152232788",
  "source_key": "tracker-migration/9324670152232788/VolumeCases-9324670152232788.duckdb",
  "dest_key": "56/9324670152232788/VolumeCases-9324670152232788.duckdb"
}
```

**EFS to EFS** -> migrate_duck_db_efs-efs_publisher.py

```json
{
    "source_datastore": "efs",
    "destination_datastore": "efs",
    "key": "325929995912186/DistributionNewPODs-325929995912186.duckdb"
}
```

**S3 to EFS**

```json
{
  "source_datastore": "s3",
  "destination_datastore": "efs",
  "source_bucket": "perform-dev-raw",
  "tracker-id": "9324670152232788",
  "source_key": "tracker-migration/9324670152232788/VolumeCases-9324670152232788.duckdb",
  "key": "9324670152232788/VolumeCases-9324670152232788.duckdb"
}
```

***EFS to S3**

```json
{
  "source_datastore": "efs",
  "destination_datastore": "s3",
  "key": "325929995912186/DistributionNewPODs-325929995912186.duckdb",
  "destination_bucket": "perform-dev-raw",
  "dest_key": "325929995912186/DistributionNewPODs-325929995912186.duckdb"
}
```

- Publisher environment variables are:

```py
source_bucket = os.environ.get("SOURCE_BUCKET", "perform-dev-raw")
destination_bucket = os.environ.get("DESTINATION_BUCKET", "perform-dev-raw")
db_secret = json.loads(cache.get_secret_string(os.environ.get("DB_SECRET", "perform_database_dev")))
secret = json.loads(cache.get_secret_string(os.environ.get("RABBIT_SECRET", "perform_rabbitmq_dev")))
```

### Consumer
- The consumer will function as a containerized task in ECS using Fargate
- Since there are two queues, there are two types of consumers, basically one with smaller capcity and the other one with higher capacity to migrate bigger files.
- So basically two services in ECS called migrate-duck-small and migrate-duck-big respectively.
- The consumer on startup will poll the queue for new messages, as soon as it receives a message, the consumer will start doing the work on a new thread.
- Every consumer has the
```py
channel.basic_qos(prefetch_count=1)
```
option set, this enables each consumer to only handle one process at a time (the queue will only send a next message to the consumer after it receives an ack for the previous message).
- The work:
    - extract values from the message
    - based on the values of "source_datastore" and "destination_datastore", the consumer can decide where to download the file from and where to upload the file to.
    - Download the file to its local directory called /batch_download from source bucket
    - Check the duckdb version of the downloaded file, if the version is already 0.8.1 and the env is 0.8.1, proceed without migrating, if it is an unsupported version then skip file.
    - Migrate from determined duckdb release to duckdb release 0.8.1 (can be set as env variable)
        - The migration process is simple, EXPORT DATABASE to exportfolder
        - IMPORT DATABASE to /batch_upload from exportfolder
    - Upload the migrated file to destination bucket (staging area)
    - Clean up the files that were created during the process to save container space.
    - Send ack to rabbit mq.

- Consumer environment variables are:

```py
duckdb_env = os.environ.get("DUCKDB_MIGRATE_VERSION", "duckdb0.8.1")
secret = json.loads(cache.get_secret_string(os.environ.get("RABBIT_SECRET", "perform_rabbitmq_dev")))
os.environ.get('MIGRATE_SRC', "/mnt/efs/perform-dev-goal/")
os.environ.get('MIGRATE_DST', "/mnt/efs/perform-dev-goal-dst/")
os.environ.get("QUEUE_NAME", "migrate_duckdb_release")
```
