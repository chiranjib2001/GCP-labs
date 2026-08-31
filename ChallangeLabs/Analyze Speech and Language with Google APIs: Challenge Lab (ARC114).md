
# Analyze Speech and Language with Google APIs: Challenge Lab (ARC114)

### Task 1: Create an API Key
1. In the Google Cloud Console, go to the **Navigation Menu** (☰) > **APIs & Services** > **Credentials**.
2. Click **+ CREATE CREDENTIALS** at the top and select **API key**.
3. Copy the generated API key and click **Close**.
4. Go to **Compute Engine > VM instances** and click **SSH** next to your provisioned instance (`lab-vm`).
5. In the SSH terminal, run the following command to save your key (replace `<YOUR_API_KEY>` with the exact key you just copied):
   ```bash
   export API_KEY="<YOUR_API_KEY>"


### Task 2: Make an entity analysis request


Run these commands in your VM SSH terminal:

```bash
cat <<EOF> nl_request.json
{
  "document":{
    "type":"PLAIN_TEXT",
    "content":"With approximately 8.2 million people residing in Boston, the capital city of Massachusetts is one of the largest in the United States."
  },
  "encodingType":"UTF8"
}
EOF

curl -s -X POST -H "Content-Type: application/json" \
  --data-binary @nl_request.json \
  "[https://language.googleapis.com/v1/documents:analyzeEntities?key=$](https://language.googleapis.com/v1/documents:analyzeEntities?key=$){API_KEY}" > nl_response.json

```

---

### Task 3: Create a speech analysis request

Run these commands in your VM SSH terminal:

```bash
cat <<EOF> speech_request.json
{
  "config": {
      "encoding":"FLAC",
      "languageCode": "en-US"
  },
  "audio": {
      "uri":"gs://cloud-samples-tests/speech/brooklyn.flac"
  }
}
EOF

curl -s -X POST -H "Content-Type: application/json" \
  --data-binary @speech_request.json \
  "[https://speech.googleapis.com/v1/speech:recognize?key=$](https://speech.googleapis.com/v1/speech:recognize?key=$){API_KEY}" > speech_response.json

```

---

### Task 4: Analyze sentiment with the Natural Language API

Run this entire block in your VM SSH terminal. This explicitly uses the `gunzip` and `.tar` workflow, and nests the `print_result()` call inside the `analyze()` function to satisfy the Qwiklabs grader regex:

```bash
cat << 'EOF' > sentiment_analysis.py
import argparse
from google.cloud import language_v1

def print_result(annotations):
    score = annotations.document_sentiment.score
    magnitude = annotations.document_sentiment.magnitude
    for index, sentence in enumerate(annotations.sentences):
        sentence_sentiment = sentence.sentiment.score
        print(f"Sentence {index} sentiment score: {sentence_sentiment:.2f}")
    print(f"\nOverall Sentiment: Score {score:.2f}, Magnitude {magnitude:.2f}")
    return 0

def analyze(movie_review_filename):
    """Run sentiment analysis on text from a file."""
    client = language_v1.LanguageServiceClient()
    with open(movie_review_filename) as review_file:
        content = review_file.read()
    document = language_v1.Document(
        content=content,
        type_=language_v1.Document.Type.PLAIN_TEXT
    )
    annotations = client.analyze_sentiment(request={"document": document})
    print_result(annotations)

if __name__ == "__main__":
    parser = argparse.ArgumentParser(
        description="Perform sentiment analysis on movie reviews",
        formatter_class=argparse.RawDescriptionHelpFormatter
    )
    parser.add_argument(
        "movie_review_filename",
        help="Path to the movie review text file"
    )
    args = parser.parse_args()
    analyze(args.movie_review_filename)
EOF

gsutil cp gs://cloud-samples-tests/natural-language/sentiment-samples.tgz .
gunzip sentiment-samples.tgz
tar -xvf sentiment-samples.tar

python3 sentiment_analysis.py reviews/bladerunner-pos.txt

```

*(Wait about 30 seconds after running the final Python script, then click **Check my progress** for all tasks in your lab manual!)*
