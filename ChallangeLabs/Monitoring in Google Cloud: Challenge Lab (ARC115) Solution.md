# Monitoring in Google Cloud: Challenge Lab (ARC115) Solution

### Task 1: Install the Cloud Logging and Monitoring agents using Ops Agent
1. In the Google Cloud Console, go to **Compute Engine > VM instances**.
2. Note the **External IP** of your VM instance (you will need it for Task 2).
3. Click the **SSH** button next to your VM instance to open a terminal connection.
4. Run the following commands in the SSH window:

```bash
curl -sSO [https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh](https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh)
sudo bash add-google-cloud-ops-agent-repo.sh --also-install

sudo cp /etc/google-cloud-ops-agent/config.yaml /etc/google-cloud-ops-agent/config.yaml.bak

sudo tee /etc/google-cloud-ops-agent/config.yaml > /dev/null << EOF
metrics:
  receivers:
    apache:
      type: apache
  service:
    pipelines:
      apache:
        receivers:
          - apache
logging:
  receivers:
    apache_access:
      type: apache_access
    apache_error:
      type: apache_error
  service:
    pipelines:
      apache:
        receivers:
          - apache_access
          - apache_error
EOF

sudo service google-cloud-ops-agent restart
sudo systemctl status google-cloud-ops-agent"*"
```
*(Press `q` to exit the status view, but keep the SSH window open for Task 3).*

### Task 2: Add an uptime check for Apache Web Server on the VM
1. In the Google Cloud Console, go to **Monitoring > Uptime checks**.
2. Click **+ Create Uptime Check**.
3. **Protocol:** Select `HTTP`.
4. **Resource Type:** Select `URL`.
5. **Hostname:** Enter the **External IP** of your VM.
6. **Path:** Leave as `/`
7. Click **Continue** through *Response Validation* and *Alert & Notification* (leave defaults).
8. **Title:** `Apache Uptime Check`.
9. Click **Test** to verify connection, then click **Create**.

### Task 3: Add an alert policy for Apache Web Server
1. Go to **Monitoring > Alerting** and click **+ Create Policy**.
2. Click **Select a Metric**.
   * Search for: `apache traffic`
   * Select **VM instance > Apache > Network traffic** (`workload/apache.traffic`) and click **Apply**.
3. In the **Transform data** section, set:
   * **Rolling window:** `1 min`
   * **Rolling window function:** `rate`
   * Click **Next**.
4. In the **Configure alert trigger** section, set:
   * **Condition type:** `Any time series violates`
   * **Threshold position:** `Above threshold`
   * **Threshold value:** `3072` *(Note: If the UI specifies KiB/s as the unit instead of B/s, type `3`)*.
   * Click **Next**.
5. In the **Notifications and name** section:
   * Click **Manage Notification Channels** (opens in a new tab).
   * Scroll to **Email**, click **ADD NEW**, enter your email/display name, and click **Save**.
   * Return to the Alerting tab, click the refresh icon ↻ next to Notification Channels, and select your email.
   * **Alert Policy Name:** `Apache High Traffic Alert`.
6. Click **Create Policy**.
7. **Trigger the Alert:** Return to your VM's SSH window and run the following command to spike the traffic:
```bash
timeout 120 bash -c -- 'while true; do curl localhost | grep -oP "<title>.*</title>"; sleep .1s;done '
```

### Task 4: Create a dashboard and charts for Apache Web Server on the VM
1. Go to **Monitoring > Dashboards** and click **+ Create Custom Dashboard**.
2. Name the dashboard `Apache Dashboard`.
3. **Chart 1 (CPU Load):**
   * Click **+ Add Widget** > Select **Line** chart.
   * In the *Metric* dropdown, search for `CPU load (1m)`.
   * Select **VM instance > Cpu > CPU load (1m)** and click **Apply**.
4. **Chart 2 (Apache Requests):**
   * Click **+ Add Widget** > Select **Line** chart.
   * In the *Metric* dropdown, search for `Requests`.
   * Select **VM instance > Apache > Requests** and click **Apply**.

### Task 5: Create a log-based metric
Open a standard **Cloud Shell** terminal in your Google Cloud Console (do *not* run this in the VM SSH window). Execute the following commands to create the metric programmatically:

```bash
PROJECT_ID=$(gcloud config get-value project)

gcloud logging metrics create test \
  --description="Count Apache 200 OK responses" \
  --log-filter='resource.type="gce_instance"
logName="projects/'"$PROJECT_ID"'/logs/apache-access"
textPayload:"200"'
```