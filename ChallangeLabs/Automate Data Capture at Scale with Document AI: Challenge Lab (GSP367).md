
# Automate Data Capture at Scale with Document AI: Challenge Lab (GSP367)

This repository contains the automated solution and deployment scripts for the **Automate Data Capture at Scale with Document AI: Challenge Lab (GSP367)**. 

The pipeline automatically processes raw invoices uploaded to Cloud Storage, extracts key form entities using a Google Cloud Document AI Form Parser processor, and writes the structured outputs directly into BigQuery.

---

## Architecture Overview

1. **Cloud Storage (Input Bucket):** Receives raw PDF/image invoices.
2. **Eventarc & Cloud Run Function:** Automatically triggers a Gen2 Python Cloud Run function upon file finalization (`google.storage.object.finalize`).
3. **Document AI API:** Parses the document layout and extracts fields using a General Form Parser processor.
4. **BigQuery:** Stores the extracted structured entities in a target table matching the provided schema.
5. **Cloud Storage (Output & Archive Buckets):** Retains processed results and archives source files.

---

## Automated Deployment Script

Instead of manually configuring each component in the console, you can run the following automated Bash script in your **Cloud Shell** to execute all tasks end-to-end.

### Prerequisites
* Active Google Cloud console session with Cloud Shell.
* Appropriate project permissions assigned to the lab student account.

### Execution Script

Save the following script as `deploy.sh` in your Cloud Shell environment, or execute it directly:

```bash
#!/bin/bash

# ==============================================================================
# Document AI Challenge Lab - Automated Deployment Script
# ==============================================================================

echo "Starting deployment for GSP367 Challenge Lab..."

# Fetch active project and compute zone/region metadata
PROJECT_ID=$(gcloud config get-value project)
PROJECT_NUMBER=$(gcloud projects describe$PROJECT_ID --format='value(projectNumber)')
REGION=$(gcloud compute project-info describe --format="value(commonInstanceMetadata.items[google-compute-default-region])")

if [ -z "$REGION" ]; then
  REGION="us-central1"
  echo "Defaulting region to: $REGION"
else
  echo "Detected region: $REGION"
fi

# ==============================================================================
# TASK 1: Enable APIs and Download Starter Files
# ==============================================================================
echo "=== TASK 1: Enabling APIs and downloading starter files ==="

gcloud services enable documentai.googleapis.com \
                        cloudfunctions.googleapis.com \
                        cloudbuild.googleapis.com \
                        eventarc.googleapis.com \
                        run.googleapis.com

mkdir -p ./document-ai-challenge
gsutil -m cp -r gs://spls/gsp367/* ~/document-ai-challenge/

# ==============================================================================
# TASK 2: Create Document AI Form Parser Processor
# ==============================================================================
echo "=== TASK 2: Creating Document AI Form Parser Processor ==="

ACCESS_TOKEN=$(gcloud auth application-default print-access-token)

# Create the form processor via REST API
curl -s -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "display_name": "form-parser-processor",
    "type": "FORM_PARSER_PROCESSOR"
  }' \
  "[https://documentai.googleapis.com/v1/projects/$PROJECT_ID/locations/us/processors](https://documentai.googleapis.com/v1/projects/$PROJECT_ID/locations/us/processors)"

# ==============================================================================
# TASK 3: Create Cloud Storage Buckets and BigQuery Dataset/Tables
# ==============================================================================
echo "=== TASK 3: Creating Storage Buckets and BigQuery Resources ==="

# Create Input, Output, and Archive Buckets with Uniform Bucket-Level Access
gcloud storage buckets create gs://${PROJECT_ID}-input-invoices --location=${REGION} --default-storage-class=STANDARD --uniform-bucket-level-access
gcloud storage buckets create gs://${PROJECT_ID}-output-invoices --location=${REGION} --default-storage-class=STANDARD --uniform-bucket-level-access
gcloud storage buckets create gs://${PROJECT_ID}-archived-invoices --location=${REGION} --default-storage-class=STANDARD --uniform-bucket-level-access

# Create BigQuery Dataset
bq --location="US" mk -d \
    --description "Form Parser Results" \
    ${PROJECT_ID}:invoice_parser_results

# Navigate to schema directory and create BigQuery tables
cd ~/document-ai-challenge/scripts/table-schema/

bq mk --table \
    invoice_parser_results.doc_ai_extracted_entities \
    doc_ai_extracted_entities.json

bq mk --table \
    invoice_parser_results.geocode_details \
    geocode_details.json

# ==============================================================================
# TASK 4: Deploy Cloud Run Function with Eventarc Trigger and IAM Bindings
# ==============================================================================
echo "=== TASK 4: Deploying Cloud Run Function ==="

cd ~/document-ai-challenge/scripts

# Grant necessary IAM roles to Compute Engine Service Account and Storage Service Agent
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"

SERVICE_ACCOUNT=$(gcloud storage service-agent --project=$PROJECT_ID)

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member serviceAccount:$SERVICE_ACCOUNT \
  --role roles/pubsub.publisher

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:service-$PROJECT_NUMBER@gs-project-accounts.iam.gserviceaccount.com" \
  --role="roles/pubsub.publisher"

echo "Waiting 20 seconds for IAM permission propagation..."
sleep 20

# Robust deployment function with retry logic for background IAM propagation
deploy_function() {
  gcloud functions deploy process-invoices \
    --gen2 \
    --region=${REGION} \
    --entry-point=process_invoice \
    --runtime=python313 \
    --source=cloud-functions/process-invoices \
    --timeout=400 \
    --env-vars-file=cloud-functions/process-invoices/.env.yaml \
    --trigger-resource=gs://${PROJECT_ID}-input-invoices \
    --trigger-event=google.storage.object.finalize \
    --service-account=$PROJECT_NUMBER-compute@developer.gserviceaccount.com \
    --allow-unauthenticated
}

deploy_success=false
while [ "$deploy_success" = false ]; do
  if deploy_function; then
    echo "✅ Function deployed successfully!"
    deploy_success=true
  else
    echo "❌ Deployment failed due to pending IAM propagation. Retrying in 10 seconds..."
    sleep 10
  fi
done

# Extract dynamically generated Processor ID
PROCESSOR_ID=$(curl -s -X GET \
  -H "Authorization: Bearer $(gcloud auth application-default print-access-token)" \
  -H "Content-Type: application/json" \
  "[https://documentai.googleapis.com/v1/projects/$PROJECT_ID/locations/us/processors](https://documentai.googleapis.com/v1/projects/$PROJECT_ID/locations/us/processors)" | \
  grep '"name":' | \
  sed -E 's/.*"name": "projects\/[0-9]+\/locations\/us\/processors\/([^"]+)".*/\1/' | head -1)

echo "Retrieved Processor ID: $PROCESSOR_ID"

# Update function with explicit environment variables (Processor ID and Parser Location)
gcloud functions deploy process-invoices \
  --gen2 \
  --region=${REGION} \
  --entry-point=process_invoice \
  --runtime=python313 \
  --source=cloud-functions/process-invoices \
  --timeout=400 \
  --trigger-resource=gs://${PROJECT_ID}-input-invoices \
  --trigger-event=google.storage.object.finalize \
  --update-env-vars=PROCESSOR_ID=${PROCESSOR_ID},PARSER_LOCATION=us,PROJECT_ID=${PROJECT_ID} \
  --service-account=$PROJECT_NUMBER-compute@developer.gserviceaccount.com

# ==============================================================================
# TASK 5: Test and Validate End-to-End Pipeline
# ==============================================================================
echo "=== TASK 5: Testing and Validating Pipeline ==="

# Upload test invoices to trigger the Cloud Run function pipeline
gsutil -m cp -r ~/document-ai-challenge/invoices/* \
  gs://${PROJECT_ID}-input-invoices/

echo "Pipeline execution triggered. Invoices uploaded successfully!"

```

---

## Verification & Troubleshooting

* **Check Cloud Run Function Logs:**
```bash
gcloud functions logs read process-invoices --gen2 --region=<YOUR_REGION> --limit=50

```


* **Verify BigQuery Data Ingestion:**
```bash
bq query --nouse_legacy_sql 'SELECT * FROM `invoice_parser_results.doc_ai_extracted_entities` LIMIT 10'

```



```
credit: https://www.youtube.com/watch?v=EF6Yzf_lE_k this is the original source.
```
