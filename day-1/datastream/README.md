# Datastream - MySQL to BigQuery

![Datastream](datastream.png)

Datastream is a serverless and easy-to-use Change Data Capture (CDC) and replication service that allows you to synchronize data across heterogeneous databases, storage systems, and applications reliably and with minimal latency. In this lab you’ll learn how to replicate data changes from your OLTP workloads into BigQuery, in real time. 

In this hands-on lab you’ll deploy the below mentioned resources all at once via terrafrom or individually. Then, you will create and start a Datastream stream, Dataflow job for replication and CDC.

What you’ll do:

- Prepare a MySQL Cloud SQL instance
- Create a GCS bucket to be used in replication/CDC
- Create a Pub/Sub topic, subscription, and a GCS Pub/Sub notification policy
- Create a BigQuery dataset
- Import data into the Cloud SQL instance
- Create a Datastream connection profile referencing the MySQL DB
- Create a Datastream connection profile referencing the GCS destination
- Create a Datastream stream and start replication
- Deploy a Dataflow job to replicate data
- Write Inserts and Updates
- Verify updates in BigQuery


## Git clone repo 

```
git clone https://github.com/AmritRaj23/data-journey.git
cd data-journey/day-1/datastream
```

## Set-up Cloud Environment

### Initilize your account and project

If you are using the Google Cloud Shell you can skip this step.

```
gcloud init
```
### Set Google Cloud Project

```
export project_id=<your-project-id>
gcloud config set project $project_id
```

### Check Google Cloud Project config set correctly

```
gcloud config list
```

### Enable Google Cloud APIs

```
gcloud services enable compute.googleapis.com dataflow.googleapis.com
```

### Set compute zone

```
gcloud config set compute/zone us-central1-f
```
## Deploy using Terraform

Use Terraform to deploy the folllowing services defined in the `main.tf` file

- Cloud SQL
- Pub/Sub
- Google Cloud Storage
- BigQuery

### Install Terraform

If you are using the Google Cloud Shell Terraform is already installed.

Follow the instructions to [install the Terraform cli](https://learn.hashicorp.com/tutorials/terraform/install-cli?in=terraform/gcp-get-started).

This repo has been tested on Terraform version `1.2.6` and the Google provider version  `4.31.0`

### Update Project ID in terraform.tfvars

Rename the `terraform.tfvars.example` file to `terraform.tfvars` and update the default project ID in the file to match your project ID.

Check that the file has been saved with the updated project ID value

```
cat terraform.tfvars
```

### Initialize Terraform

```
terraform init
```

### Create resources in Google Cloud

Run the plan cmd to see what resources will be greated in your project.

**Important: Make sure you have updated the Project ID in terraform.tfvars before running this**

```
terraform plan
```

Run the apply cmd and point to your `.tfvars` file to deploy all the resources in your project.

```
terraform apply -var-file terraform.tfvars
```

This will show you a plan of everything that will be created and then the following notification where you should enter `yes` to proceed:

```
Plan: 26 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: 
```

### Terraform output

Once everything has succesfully run you should see the following output:

```
google_compute_network.vpc_network: Creating...
.
.
.
Apply complete! Resources: 26 added, 0 changed, 0 destroyed.
```

## Import a SQL file into MySQL

Open a file named create_mysql.sql in vim or your favorite editor, then copy the text below into your file:

```
CREATE DATABASE IF NOT EXISTS database_datajourney;
USE database_datajourney;

CREATE TABLE IF NOT EXISTS database_datajourney.example_table (
id INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
text_col VARCHAR(50),
int_col INT,
created_at TIMESTAMP
);

INSERT INTO database_datajourney.example_table (text_col, int_col, created_at) VALUES
('hello', 0, '2020-01-01 00:00:00'),
('goodbye', 1, NULL),
('name', -987, NOW()),
('other', 2786, '2021-01-01 00:00:00');
```

Next, you will copy this file into the Cloud Storage bucket you created above (make sure you do not load the file into the `data/` directory), make the file accessible to your Cloud SQL service account, and import the SQL command into your database.

```
SERVICE_ACCOUNT=$(gcloud sql instances describe mysql-instance | grep serviceAccountEmailAddress | awk '{print $2;}')

gsutil cp create_mysql.sql gs://${project_id}/resources/create_mysql.sql
gsutil iam ch serviceAccount:${SERVICE_ACCOUNT}:objectViewer gs://${project_id}

gcloud sql import sql mysql-instance gs://${project_id}/resources/create_mysql.sql --quiet
```

## Create Datastream resources

In the Cloud Console UO, navigate to Datastream then click Enable to enable the Datastream AP.

Create two connection profiles, one for the MySQL source, and another for the Cloud Storage destination.

My SQL connection profile:
- The IP and port of the Cloud SQL for MySQL instance created earlier
- username: `root`, password: `password123`
- encryption: none
- connectivity method: IP allowlisting

Cloud Storage connection profile:
- connection profile path prefix: /data

Create stream by selecting MyQL and cloud storage connection profiles, and make sure to mark the tables you want to replicate (we will only replicate the datastram-datajourney database), and finally run validation, and create and start the stream.

## Deploy Dataflow job

```
gcloud dataflow flex-template run datastream-replication
--template-file-gcs-location gs://dataflow-templates-us-central1/latest/flex/Cloud_Datastream_to_BigQuery
--region us-central1
--worker-region us-central1
--network terraform-network
--parameters
  inputFilePattern=gs://${project_id}/data,gcsPubSubSubscription=projects/final-test-360907/subscriptions/datastream-subscription,
  inputFileFormat=avro,
  outputStagingDatasetTemplate=dataset,
  outputDatasetTemplate=dataset,
  deadLetterQueueDirectory=gs://${project_id}/dlq/,
  serviceAccount=datastreampipeline@${project_id}.iam.gserviceaccount.com
```

## View the Data in BigQuery

View these tables in the BigQuery UI.

## Write Inserts and Updates

Open a file named update_mysql.sql in vim or your editor, then copy the text below into your file:

```
CREATE DATABASE IF NOT EXISTS database_datajourney;
USE database_datajourney;

CREATE TABLE IF NOT EXISTS database_datajourney.example_table (
id INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
text_col VARCHAR(50),
int_col INT,
created_at TIMESTAMP
);

INSERT INTO database_datajourney.example_table (text_col, int_col, created_at) VALUES
('abc', 0, '2021-05-01 00:00:00'),
('def', 1, NULL),
('ghi', -987, NOW()),
('jkl', 2786, '2021-05-01 00:00:00'),
('abc', 0, '2021-05-01 00:00:00'),
('def', 1, NULL),
('ghi', -987, NOW()),
('jkl', 2786, '2021-05-01 00:00:00'),
('abc', 0, '2021-05-01 00:00:00'),
('def', 1, NULL),
('ghi', -987, NOW()),
('jkl', 2786, '2021-05-01 00:00:00'),
('abc', 0, '2021-05-01 00:00:00'),
('def', 1, NULL),
('ghi', -987, NOW()),
('jkl', 2786, '2021-05-01 00:00:00'),
('abc', 0, '2021-05-01 00:00:00'),
('def', 1, NULL),
('ghi', -987, NOW()),
('jkl', 2786, '2021-05-01 00:00:00');

UPDATE database_datajourney.example_table SET int_col=int_col*2;
```

Next, you will copy this file into the Cloud Storage bucket you created above (make sure you do not load the file into the `data/` directory), make the file accessible to your Cloud SQL service account, and import the SQL command into your database.

```
SQL_FILE=update_mysql.sql
SERVICE_ACCOUNT=$(gcloud sql instances describe mysql-instance | grep serviceAccountEmailAddress | awk '{print $2;}')

gsutil cp ${SQL_FILE} gs://${project_id}/resources/${SQL_FILE}
gsutil iam ch serviceAccount:${SERVICE_ACCOUNT}:objectViewer gs://${project_id}

gcloud sql import sql mysql-instance gs://${project_id}/resources/${SQL_FILE} --quiet
```

## Verify updates in BigQuery

Run the query below to verify data changes in BiqQuery:

```
SELECT
 *
FROM
 `<project_id>.dataset.example_table`
LIMIT
 100
```

## Terraform Destroy

Use Terraform to destroy all resources

```
terraform destroy
```