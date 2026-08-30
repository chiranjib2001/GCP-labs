```markdown
# Organize and Govern Data with Knowledge Catalog: Challenge Lab (ARC117)

## Setup: Define Variables and Enable APIs
Open the **Cloud Shell** (the terminal icon in the top right of the Google Cloud console) and run the following commands. 

**Important:** Replace `<YOUR_REGION>` with the exact region provided to you on your Qwiklabs lab instruction panel (e.g., `us-central1`).

```bash
export REGION="<YOUR_REGION>"
export PROJECT_ID=$(gcloud config get-value project)

# Enable the Knowledge Catalog (Dataplex) API
gcloud services enable dataplex.googleapis.com

```

---

## Task 1: Create a lake with a raw zone

Run the following commands in your Cloud Shell to create the lake and the raw zone:

```bash
# Create the Lake
gcloud dataplex lakes create customer-engagements \
  --location=$REGION \
  --display-name="Customer Engagements"

# Create the Raw Zone
gcloud dataplex zones create raw-event-data \
  --location=$REGION \
  --lake=customer-engagements \
  --display-name="Raw Event Data" \
  --resource-location-type=SINGLE_REGION \
  --type=RAW

```

*Note: It may take 1-2 minutes for these commands to finish provisioning the resources. Once they complete, you can click **Check my progress** for Task 1.*

---

## Task 2: Create and attach a Cloud Storage bucket to the zone

Run the following commands in your Cloud Shell to create the Cloud Storage bucket and attach it as an asset to your new zone:

```bash
# Create the Cloud Storage bucket using your Project ID
gcloud storage buckets create gs://$PROJECT_ID --location=$REGION

# Attach the bucket to the zone as an asset
gcloud dataplex assets create raw-event-files \
  --location=$REGION \
  --lake=customer-engagements \
  --zone=raw-event-data \
  --display-name="Raw Event Files" \
  --resource-type=STORAGE_BUCKET \
  --resource-name=projects/$PROJECT_ID/buckets/$PROJECT_ID \
  --discovery-enabled

```

*Note: Wait a minute or two for the asset attachment to finalize, then click **Check my progress** for Task 2.*

---

## Task 3: Create an aspect type and add the aspect to an asset

Because custom enum templates require complex schema payloads in the CLI, it is highly recommended to do this task manually in the Google Cloud Console UI.

### 1. Create the Aspect Type

1. In the Google Cloud Console navigation menu (☰), go to **Analytics > Knowledge Catalog** (or **Dataplex**).
2. On the left menu, under **Manage Metadata**, click **Metadata Types**.
3. Select the **Aspect types** tab and click **+ Create**.
4. Set the following properties:
* **Display Name:** `Protected Raw Data Aspect`
* **Location:** Select your assigned `<REGION>`


5. Under the **Template** section, click **Add field**:
* **Field Display Name:** `Protected Raw Data Flag`
* **Type:** `Enum`
* Click **Add an enum value**, enter `Y`, and click **Done**.
* Click **Add an enum value** again, enter `N`, and click **Done**.


6. Click **Save** at the bottom of the page. *(Wait about a minute for the Aspect Type to be fully created).*

### 2. Add the Aspect to the Zone

1. On the left menu, under **Discover**, click **Search**.
2. In the Search bar, type `Raw Event Data` and hit Enter.
3. Click on the **Raw Event Data** zone from the search results.
4. Scroll down to the **Aspects** section.
5. Next to *Optional aspects*, click the **Add** button.
6. In the Filter box, type `Protected Raw Data Aspect` and select it from the list.
7. For the **Protected Raw Data Flag** dropdown, select either `Y` or `N`.
8. Click **Save**.

Once the save completes, you can click **Check my progress** for Task 3!
