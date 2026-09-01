# Create a Secure Data Lake on Cloud Storage: Challenge Lab (ARC119)

Because this is a Challenge Lab, Qwiklabs randomly assigns one of **4 different forms** when you click *Start Lab*.

Review your specific tasks on the lab page, identify which form you have been assigned based on the task list below, and run the corresponding script in **Google Cloud Shell**.

---

### How to use this guide:

1. Open **Cloud Shell** in your Google Cloud Console.
2. Ensure the Dataplex API is enabled before running your form's script:
```bash
gcloud services enable dataplex.googleapis.com
export PROJECT_ID=$(gcloud config get-value project)
export REGION="us-west1"

```


3. Copy the script for your assigned form and paste it into Cloud Shell.

---

## 🗂️ Form 1

**How to identify:** Your tasks include *Create a Cloud Storage bucket, Create a lake and zone, Create an entry group, Create a tag template (Storage bucket).*

**Cloud Shell Script:**

```bash
# 1. Create a Cloud Storage bucket
gsutil mb -p $PROJECT_ID -l $REGION -b on gs://$PROJECT_ID-bucket/

# 2. Create a lake in Dataplex
gcloud alpha dataplex lakes create customer-lake \
    --display-name="Customer-Lake" \
    --location=$REGION \
    --labels="domain_type=source_data"

# 3. Add a zone to your lake
gcloud dataplex zones create public-zone \
    --lake=customer-lake \
    --location=$REGION \
    --type=RAW \
    --resource-location-type=SINGLE_REGION \
    --display-name="Public-Zone"

# 4. Create an entry group (Environment)
gcloud dataplex environments create dataplex-lake-env \
    --project=$PROJECT_ID \
    --location=$REGION \
    --lake=customer-lake \
    --os-image-version=1.0 \
    --compute-node-count 3 \
    --compute-max-node-count 3

# 5. Create a tag template
gcloud data-catalog tag-templates create customer_data_tag_template \
    --location=$REGION \
    --display-name="Customer Data Tag Template" \
    --field=id=data_owner,display-name="Data Owner",type=string,required=TRUE \
    --field=id=pii_data,display-name="PII Data",type="enum(Yes|No)",required=TRUE

```

*(Check your progress!)*

---

## 🗂️ Form 2

**How to identify:** Your tasks include *Create a lake and zone, Create an entry group, Attach an existing Cloud Storage bucket, Create a tag template.*

**Cloud Shell Script:**

```bash
# 1. Create a lake in Dataplex
gcloud alpha dataplex lakes create customer-lake \
    --display-name="Customer-Lake" \
    --location=$REGION \
    --labels="domain_type=source_data"

# 2. Add a zone to your lake
gcloud dataplex zones create public-zone \
    --lake=customer-lake \
    --location=$REGION \
    --type=RAW \
    --resource-location-type=SINGLE_REGION \
    --display-name="Public-Zone"

# 3. Create an entry group (Environment)
gcloud dataplex environments create dataplex-lake-env \
    --project=$PROJECT_ID \
    --location=$REGION \
    --lake=customer-lake \
    --os-image-version=1.0 \
    --compute-node-count 3 \
    --compute-max-node-count 3

# 4. Attach existing Cloud Storage bucket
gcloud dataplex assets create customer-raw-data \
    --location=$REGION \
    --lake=customer-lake \
    --zone=public-zone \
    --resource-type=STORAGE_BUCKET \
    --resource-name=projects/$PROJECT_ID/buckets/$PROJECT_ID-customer-bucket \
    --discovery-enabled \
    --display-name="Customer Raw Data"

# 5. Attach existing BigQuery Dataset
gcloud dataplex assets create customer-reference-data \
    --location=$REGION \
    --lake=customer-lake \
    --zone=public-zone \
    --resource-type=BIGQUERY_DATASET \
    --resource-name=projects/$PROJECT_ID/datasets/customer_reference_data \
    --display-name="Customer Reference Data"

# 6. Create a tag template
gcloud data-catalog tag-templates create customer_data_tag_template \
    --location=$REGION \
    --display-name="Customer Data Tag Template" \
    --field=id=data_owner,display-name="Data Owner",type=string,required=TRUE \
    --field=id=pii_data,display-name="PII Data",type='enum(Yes|No)',required=TRUE

```

*(Check your progress!)*

---

## 🗂️ Form 3

**How to identify:** Your tasks include *Create a BigQuery dataset, Add a zone to your lake, Attach an existing BigQuery Dataset, Create a tag template.*

**Cloud Shell Script:**

```bash
# 1. Create a BigQuery Dataset & Load Data
bq mk --location=US Raw_data
bq load --source_format=AVRO Raw_data.public-data gs://spls/gsp1145/users.avro

# 2. Add a zone to your lake
gcloud dataplex zones create temperature-raw-data \
    --lake=public-lake \
    --location=$REGION \
    --type=RAW \
    --resource-location-type=SINGLE_REGION \
    --display-name="temperature-raw-data"

# 3. Attach an existing BigQuery Dataset
gcloud dataplex assets create customer-details-dataset \
    --location=$REGION \
    --lake=public-lake \
    --zone=temperature-raw-data \
    --resource-type=BIGQUERY_DATASET \
    --resource-name=projects/$PROJECT_ID/datasets/customer_reference_data \
    --display-name="Customer Details Dataset" \
    --discovery-enabled

# 4. Create a tag template
gcloud data-catalog tag-templates create protected_data_template \
    --location=$REGION \
    --display-name="Protected Data Template" \
    --field=id=protected_data_flag,display-name="Protected Data Flag",type='enum(Yes|No)',required=TRUE

```

*(Check your progress!)*

---

## 🗂️ Form 4

**How to identify:** Your tasks include *Create a lake and zone, Attach an existing Cloud Storage bucket, Attach an existing BigQuery Dataset, Create Entities.*

**Part 1: Cloud Shell Script (Tasks 1, 2, and 3)**

```bash
# 1. Create a lake in Dataplex
gcloud alpha dataplex lakes create customer-lake \
    --display-name="Customer-Lake" \
    --location=$REGION \
    --labels="domain_type=source_data"

# 2. Add a zone to your lake
gcloud dataplex zones create public-zone \
    --lake=customer-lake \
    --location=$REGION \
    --type=RAW \
    --resource-location-type=SINGLE_REGION \
    --display-name="Public-Zone" \
    --discovery-enabled

# 3. Attach existing Cloud Storage bucket (with Discovery parsing configs)
gcloud dataplex assets create customer-raw-data \
    --location=$REGION \
    --lake=customer-lake \
    --zone=public-zone \
    --resource-type=STORAGE_BUCKET \
    --resource-name=projects/$PROJECT_ID/buckets/$PROJECT_ID-customer-bucket \
    --discovery-enabled \
    --display-name="Customer Raw Data" \
    --csv-delimiter="," \
    --csv-encoding="UTF-8" \
    --csv-header-rows=1

# 4. Attach existing BigQuery Dataset
gcloud dataplex assets create customer-details-dataset \
    --location=$REGION \
    --lake=customer-lake \
    --zone=public-zone \
    --resource-type=BIGQUERY_DATASET \
    --resource-name=projects/$PROJECT_ID/datasets/customer_reference_data \
    --display-name="Customer Details Dataset"

```

**Part 2: UI Instructions (Task 4 - Create Entities)**
Since Entity creation requires manual schema binding, complete Task 4 via the Web UI:

1. Paste this URL into your browser to go directly to the Entity Creation page:
`[https://console.cloud.google.com/dataplex/lakes/customer-lake/zones/public-zone/create-entity;location=us-west1](https://console.cloud.google.com/dataplex/lakes/customer-lake/zones/public-zone/create-entity;location=us-west1)`
2. Fill out the **Add Entity** form with the following details:
* **Display Name:** `My Entity`
* **Table Name:** `public_table`
* **Table Source:** `Google Cloud Storage`
* **Asset:** `Customer Raw Data`
* **File Format:** `CSV`
* **Schema:** *(Leave all as default)*


3. Click **Create** at the bottom of the page.

*(Check your progress!)*
