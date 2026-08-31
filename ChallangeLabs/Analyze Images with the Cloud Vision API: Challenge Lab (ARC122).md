
# Analyze Images with the Cloud Vision API: Challenge Lab (ARC122)

## Task 1: Verify your resources

Run the following block in your Cloud Shell to set up variables, enable APIs, generate your API Key, and make the image inside your pre-provisioned bucket publicly accessible.

```bash
export PROJECT_ID=$(gcloud config get-value project)
export BUCKET_NAME="${PROJECT_ID}-bucket"

# Enable the necessary APIs
gcloud services enable vision.googleapis.com apikeys.googleapis.com

# Create an API Key programmatically
gcloud services api-keys create --display-name="Vision API Key"

# Extract the API Key and save it to the environment variable
KEY_NAME=$(gcloud services api-keys list --filter="displayName='Vision API Key'" --format="value(name)")
export API_KEY=$(gcloud services api-keys get-key-string$KEY_NAME --format="value(keyString)")

# Get the exact image URI from the bucket
export IMAGE_URI=$(gcloud storage ls gs://$BUCKET_NAME | head -n 1)

# Make the object publicly accessible 
gsutil iam ch allUsers:objectViewer gs://$BUCKET_NAME

```

*Wait a few seconds for IAM policies to apply, then click **Check my progress** for Task 1.*

---

## Task 2 & 3 (Part 1): Text Detection

This block creates the `request.json` file for `TEXT_DETECTION`, calls the Vision API, saves the response, and uploads it back to your Cloud Storage bucket.

```bash
# Create the request.json file for TEXT_DETECTION
cat <<EOF> request.json
{
  "requests": [
      {
        "image": {
          "source": {
              "gcsImageUri": "${IMAGE_URI}"
          }
        },
        "features": [
          {
            "type": "TEXT_DETECTION",
            "maxResults": 10
          }
        ]
      }
  ]
}
EOF

# Call the Vision API and save the response
curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json "[https://vision.googleapis.com/v1/images:annotate?key=$](https://vision.googleapis.com/v1/images:annotate?key=$){API_KEY}" -o text-response.json

# Upload output to Cloud Storage
gcloud storage cp text-response.json gs://$BUCKET_NAME/

```

*Click **Check my progress** for the first part of Task 3.*

---

## Task 3 (Part 2): Landmark Detection

Overwrite the `request.json` file to use `LANDMARK_DETECTION`, call the API again, and upload the new response to the bucket.

```bash
# Update the request.json file for LANDMARK_DETECTION
cat <<EOF> request.json
{
  "requests": [
      {
        "image": {
          "source": {
              "gcsImageUri": "${IMAGE_URI}"
          }
        },
        "features": [
          {
            "type": "LANDMARK_DETECTION",
            "maxResults": 10
          }
        ]
      }
  ]
}
EOF

# Call the Vision API and save the response
curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json "[https://vision.googleapis.com/v1/images:annotate?key=$](https://vision.googleapis.com/v1/images:annotate?key=$){API_KEY}" -o landmark-response.json

# Upload output to Cloud Storage
gcloud storage cp landmark-response.json gs://$BUCKET_NAME/

```

*Click **Check my progress** for the final task to secure your 100/100 points!*

