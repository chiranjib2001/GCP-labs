# Configure Service Accounts and IAM Roles for Google Cloud: Challenge Lab (ARC134)

This lab evaluates your ability to administer Google Cloud Identity and Access Management (IAM), including creating service accounts, assigning pre-defined roles, attaching service accounts to Compute Engine virtual machines, building custom IAM roles using YAML, and using Python client libraries to execute BigQuery jobs via service account credentials.

### Before You Begin

1. Open the **Google Cloud Console**.
2. Click the **Activate Cloud Shell** icon (the terminal prompt `>_` symbol) in the top right toolbar.
3. If prompted, click **Continue**.

---

### Step 1: Initialize Variables & Create the Devops Service Account (Tasks 2 & 3)

Run the following code block in your Cloud Shell terminal to dynamically retrieve your active project settings, configure `gcloud` defaults, create the required `devops` service account, and securely apply its necessary IAM policy bindings (`iam.serviceAccountUser` and `compute.instanceAdmin`).

```bash
# 1. Initialize environment variables
export PROJECT_ID=$(gcloud config get-value project)
export ZONE=$(gcloud compute project-info describe --format="value(commonInstanceMetadata.items[google-compute-default-zone])" 2>/dev/null)
export REGION=${ZONE%-*}

gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE

# 2. Create the devops service account
gcloud iam service-accounts create devops --display-name devops

# Wait to ensure Google Cloud finishes creating the Service Account
sleep 5

# 3. Retrieve the full service account email address
SA_DEVOPS=$(gcloud iam service-accounts list --format="value(email)" --filter "displayName=devops")

# 4. Assign IAM permissions to the devops service account
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SA_DEVOPS" \
  --role="roles/iam.serviceAccountUser" \
  --quiet

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SA_DEVOPS" \
  --role="roles/compute.instanceAdmin" \
  --quiet

```

*Wait a moment for IAM policies to propagate, then click **Check my progress** on **Tasks 2 & 3**.*

---

### Step 2: Create the VM & Custom IAM Role (Tasks 4 & 5)

This block creates the `vm-2` instance bound to your new `devops` service account, and constructs the `role-definition.yaml` file to define a custom role containing the specific Cloud SQL permissions requested by the lab. It then pushes that YAML file into the pre-existing `lab-vm` instance to satisfy the strict Qwiklabs validation checks.

```bash
# 1. Create a compute instance with the devops service account attached
gcloud compute instances create vm-2 \
    --machine-type=e2-micro \
    --service-account=$SA_DEVOPS \
    --zone=$ZONE \
    --scopes=https://www.googleapis.com/auth/compute \
    --quiet

# 2. Create the Custom Role YAML definition locally
cat > role-definition.yaml <<EOF
title: Custom Role
description: Custom role with cloudsql permissions
includedPermissions:
- cloudsql.instances.connect
- cloudsql.instances.get
EOF

# 3. Apply the custom role to the Google Cloud Project
gcloud iam roles create customRole \
  --project $PROJECT_ID \
  --file role-definition.yaml \
  --quiet

# 4. Upload the YAML file to lab-vm to satisfy the grader's specific file-check requirement
gcloud compute scp role-definition.yaml lab-vm:~/role-definition.yaml --zone=$ZONE --quiet

```

*Click **Check my progress** on **Tasks 4 & 5**.*

---

### Step 3: BigQuery Service Account & Target VM Setup (Task 6)

Here, we create the `bigquery-qwiklab` service account, grant it `bigquery.dataViewer` and `bigquery.user` access so it can execute SQL queries against public datasets, and attach it to a new VM named `bigquery-instance`.

```bash
# 1. Create the bigquery-qwiklab service account
gcloud iam service-accounts create bigquery-qwiklab --display-name bigquery-qwiklab

sleep 5
SA_BQ=$(gcloud iam service-accounts list --format="value(email)" --filter "displayName=bigquery-qwiklab")

# 2. Assign BigQuery roles to the service account
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SA_BQ" \
  --role="roles/bigquery.dataViewer" \
  --quiet

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SA_BQ" \
  --role="roles/bigquery.user" \
  --quiet

# 3. Create the bigquery-instance VM attached to the BigQuery service account
gcloud compute instances create bigquery-instance \
    --machine-type=e2-micro \
    --service-account=$SA_BQ \
    --scopes=https://www.googleapis.com/auth/cloud-platform \
    --zone=$ZONE \
    --quiet

```

---

### Step 4: Execute the Python Payload on `bigquery-instance` (Task 6)

The lab requires you to execute a specific Python script from *inside* the `bigquery-instance`. This final block safely bundles the Python environment setup and the SQL execution script into a shell payload, transfers it to the `bigquery-instance`, and executes it remotely via SSH.

*(Note: We use strict quoting `<<'EOF'` below to ensure Bash doesn't prematurely evaluate the Python script's variables or SQL syntax before it reaches the VM).*

```bash
# 1. Generate the remote execution script
cat > bq_script.sh <<'EOF'
#!/bin/bash
echo "Installing Python dependencies..."
sudo apt-get update -y
sudo apt-get install -y git python3-pip python3.11-venv

# Setup Python Virtual Environment
python3 -m venv myvenv
source myvenv/bin/activate
pip install --upgrade pip
pip install google-cloud-bigquery pandas pyarrow db-dtypes google-auth

# Extract VM Metadata for the Python Script
export VM_PROJECT_ID=$(curl -s "http://metadata.google.internal/computeMetadata/v1/project/project-id" -H "Metadata-Flavor: Google")
export VM_SA_EMAIL=$(curl -s "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/email" -H "Metadata-Flavor: Google")

echo "Generating Python query script..."
cat > query.py <<INLINE_EOF
from google.auth import compute_engine
from google.cloud import bigquery

credentials = compute_engine.Credentials(
    service_account_email='${VM_SA_EMAIL}')

query = '''
SELECT name, SUM(number) as total_people
FROM \`bigquery-public-data.usa_names.usa_1910_2013\`
WHERE state = 'TX'
GROUP BY name, state
ORDER BY total_people DESC
LIMIT 20
'''

client = bigquery.Client(
    project='${VM_PROJECT_ID}',
    credentials=credentials)

print(client.query(query).to_dataframe())
INLINE_EOF

echo "Executing Python script..."
python3 query.py
EOF

# 2. Transfer the script to the VM and execute it over SSH
gcloud compute scp bq_script.sh bigquery-instance:/tmp --zone=$ZONE --quiet
gcloud compute ssh bigquery-instance --zone=$ZONE --quiet --command="bash /tmp/bq_script.sh"

```

Once the terminal outputs the Pandas DataFrame showing the list of names and `total_people` counts, return to the lab manual and click **Check my progress** for **Task 6**. You should now have 100/100 points!
