# Create and Manage AlloyDB Instances: Challenge Lab (GSP395)

Run the following automated script in **Google Cloud Shell** to safely complete all tasks (Tasks 1 through 5) in a single execution without terminal paste corruption.

### Execution Script

1. Open **Google Cloud Shell**.
2. Copy the entire block below and paste it into the terminal.
3. Press **Enter**.

```bash
cat << 'EOF' > run_alloy.sh
#!/bin/bash
export PROJECT_ID=$(gcloud config get-value project)
export ZONE=$(gcloud compute instances list --filter="name=alloydb-client" --format="value(zone)")
export REGION="${ZONE%-*}"

echo "Detected Region: $REGION"

echo "Task 1: Creating AlloyDB Cluster..."
gcloud beta alloydb clusters create lab-cluster \
    --password=Change3Me \
    --network=peering-network \
    --region=$REGION \
    --project=$PROJECT_ID

echo "Task 1: Creating Primary Instance..."
gcloud beta alloydb instances create lab-instance \
    --instance-type=PRIMARY \
    --cpu-count=2 \
    --region=$REGION \
    --cluster=lab-cluster \
    --project=$PROJECT_ID

echo "Tasks 2 & 3: Creating Tables and Loading Data..."
export ALLOYDB=$(gcloud beta alloydb clusters describe lab-cluster --region=$REGION --format="value(ipAddress)")

gcloud compute ssh alloydb-client --zone=$ZONE --quiet --command="
export PGPASSWORD=Change3Me
psql -h $ALLOYDB -U postgres -c \"
CREATE TABLE regions (region_id bigint NOT NULL PRIMARY KEY, region_name varchar(25));
CREATE TABLE countries (country_id char(2) NOT NULL PRIMARY KEY, country_name varchar(40), region_id bigint);
CREATE TABLE departments (department_id smallint NOT NULL PRIMARY KEY, department_name varchar(30), manager_id integer, location_id smallint);

INSERT INTO regions VALUES (1, 'Europe'), (2, 'Americas'), (3, 'Asia'), (4, 'Middle East and Africa');
INSERT INTO countries VALUES ('IT', 'Italy', 1), ('JP', 'Japan', 3), ('US', 'United States of America', 2), ('CA', 'Canada', 2), ('CN', 'China', 3), ('IN', 'India', 3), ('AU', 'Australia', 3), ('ZW', 'Zimbabwe', 4), ('SG', 'Singapore', 3);
INSERT INTO departments VALUES (10, 'Administration', 200, 1700), (20, 'Marketing', 201, 1800), (30, 'Purchasing', 114, 1700), (40, 'Human Resources', 203, 2400), (50, 'Shipping', 121, 1500), (60, 'IT', 103, 1400);
\""

echo "Task 4: Creating Read Pool Instance..."
gcloud beta alloydb instances create lab-instance-rp1 \
    --instance-type=READ_POOL \
    --cpu-count=2 \
    --read-pool-node-count=2 \
    --region=$REGION \
    --cluster=lab-cluster \
    --project=$PROJECT_ID

echo "Task 5: Creating Backup..."
gcloud beta alloydb backups create lab-backup \
    --cluster=lab-cluster \
    --region=$REGION \
    --project=$PROJECT_ID

echo "All tasks complete!"
EOF

bash run_alloy.sh

```

**Note on Timing:** AlloyDB cluster and instance provisioning takes approximately **15 to 25 minutes**. Leave your Cloud Shell tab open and active until the script completes entirely. Once finished, click **Check my progress** for all tasks on the lab instruction page.
