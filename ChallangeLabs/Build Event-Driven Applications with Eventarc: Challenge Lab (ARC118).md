
# Build Event-Driven Applications with Eventarc: Challenge Lab (ARC118)

This document provides the technical architecture overview, implementation details, and verification steps for Part 2 of the Eventarc Challenge Lab.

## Architecture & Workflow

1. **Pub/Sub Topic (`${PROJECT_ID}-topic`)**: Acts as the central event ingestion bus where publishers push messages.
2. **Pub/Sub Subscription (`${PROJECT_ID}-topic-sub`)**: Ensures message persistence and pull/push subscription bindings for the topic.
3. **Cloud Run Service (`pubsub-events`)**: A stateless container service deployed using the public Google-provided Hello World image (`gcr.io/cloudrun/hello`), serving as the event sink.
4. **Eventarc Trigger (`pubsub-events-trigger`)**: Intercepts `google.cloud.pubsub.topic.v1.messagePublished` events from the transport topic and securely routes them to the Cloud Run sink container service.

---

## Detailed Execution Breakdown

### 1. Environment & API Initialization
The lab setup requires defining the project scope and ensuring all dependent control planes (Cloud Run, Eventarc, Pub/Sub, Logging, and Cloud Build) are fully active.

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=$(gcloud config get-value compute/region)

gcloud services enable run.googleapis.com \
    eventarc.googleapis.com \
    pubsub.googleapis.com \
    logging.googleapis.com \
    cloudbuild.googleapis.com

```

### 2. Messaging Infrastructure (Pub/Sub)

Task 1 mandates a specific naming convention using your active project ID prefix.

* **Topic Creation:**
```bash
gcloud pubsub topics create ${PROJECT_ID}-topic

```


* **Subscription Creation:**
```bash
gcloud pubsub subscriptions create --topic ${PROJECT_ID}-topic${PROJECT_ID}-topic-sub

```



### 3. Containerized Event Sink (Cloud Run)

Task 2 deploys a managed serverless container service to ingest incoming event payloads.

```bash
gcloud run deploy pubsub-events \
  --image=gcr.io/cloudrun/hello \
  --platform=managed \
  --region=${REGION} \
  --allow-unauthenticated

```

### 4. Event Routing & Testing (Eventarc)

Task 3 couples the Pub/Sub transport layer with the Cloud Run destination using an Eventarc trigger. A retry loop is implemented to gracefully handle asynchronous IAM service agent provisioning delays.

```bash
while ! gcloud eventarc triggers create pubsub-events-trigger \
  --location=${REGION} \
  --destination-run-service=pubsub-events \
  --destination-run-region=${REGION} \
  --transport-topic=${PROJECT_ID}-topic \
  --event-filters="type=google.cloud.pubsub.topic.v1.messagePublished"; do
    echo "Waiting for Eventarc service agent provisioning. Retrying in 15 seconds..."
    sleep 15
done

```

* **Publishing Validation Message:**
```bash
gcloud pubsub topics publish ${PROJECT_ID}-topic \
  --message="Eventarc trigger validation test message"

```



---

## Verification & Troubleshooting

* **Verify Cloud Run Service Status:**
```bash
gcloud run services describe pubsub-events --region=${REGION} --format="value(status.url)"

```


* **Inspect Eventarc Trigger Configuration:**
```bash
gcloud eventarc triggers describe pubsub-events-trigger --location=${REGION}

```


* **View Cloud Run Container Execution Logs:**
```bash
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=pubsub-events" --limit=20 --format="table(timestamp, textPayload)"

```



```

```
