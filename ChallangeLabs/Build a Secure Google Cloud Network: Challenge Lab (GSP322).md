# Build a Secure Google Cloud Network: Challenge Lab (GSP322)

This guide provides the complete solution for **GSP322**, configuring targeted VPC firewall rules, network tags, and secure administrative access via Identity-Aware Proxy (IAP).

---

## Lab Architecture & Requirements

* **Bastion Host:** Accessible externally via SSH over Identity-Aware Proxy (IAP) only; no external public IP.
* **Juice-Shop Server:** Accessible via HTTP (`tcp:80`) publicly, and via SSH (`tcp:22`) only from the management subnet (`acme-mgmt-subnet`) through the bastion.
* **Overly Permissive Rules:** All open/default broad access firewall rules must be removed.

---

## Task-by-Task Implementation

### Tasks 1–5: Automated CLI Configuration

Run the following block directly in **Google Cloud Shell** to clean up insecure rules, start the bastion, apply network tags, and provision least-privilege firewall rules.

```bash
# 1. Dynamically retrieve the provisioned zone and subnet CIDR
export ZONE=$(gcloud compute instances list --filter="name=bastion" --format="value(zone)")
export REGION=${ZONE%-*}
export MGMT_SUBNET_IP=$(gcloud compute networks subnets describe acme-mgmt-subnet --region=${REGION} --format="value(ipCidrRange)")

# 2. Extract network tags specified in the lab manual
# (Verify your exact tag suffix in the lab instructions, e.g., accept-ssh-iap-ingress-ql-302)
export IAP_TAG=$(gcloud compute instances describe bastion --zone=${ZONE} --format="value(tags.items)" 2>/dev/null | grep -o 'accept-ssh-iap-ingress-ql-[0-9]*' || echo "accept-ssh-iap-ingress-ql-302")
export HTTP_TAG=$(gcloud compute instances describe juice-shop --zone=${ZONE} --format="value(tags.items)" 2>/dev/null | grep -o 'accept-http-ingress-ql-[0-9]*' || echo "accept-http-ingress-ql-302")
export INTERNAL_SSH_TAG=$(gcloud compute instances describe juice-shop --zone=${ZONE} --format="value(tags.items)" 2>/dev/null | grep -o 'accept-ssh-internal-ingress-ql-[0-9]*' || echo "accept-ssh-internal-ingress-ql-302")

# Task 1: Delete overly permissive firewall rule
gcloud compute firewall-rules delete open-access --quiet

# Task 2: Start the bastion host instance
gcloud compute instances start bastion --zone=${ZONE}

# Task 3: Allow SSH (tcp/22) from Google Cloud IAP IP range (35.235.240.0/20) to bastion
gcloud compute firewall-rules create allow-ssh-iap-ingress \
  --network=acme-vpc \
  --allow=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --target-tags=${IAP_TAG}

gcloud compute instances add-tags bastion \
  --tags=${IAP_TAG} \
  --zone=${ZONE}

# Task 4: Allow HTTP (tcp/80) ingress to juice-shop
gcloud compute firewall-rules create allow-http-ingress \
  --network=acme-vpc \
  --allow=tcp:80 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=${HTTP_TAG}

gcloud compute instances add-tags juice-shop \
  --tags=${HTTP_TAG} \
  --zone=${ZONE}

# Task 5: Allow internal SSH from acme-mgmt-subnet to juice-shop
gcloud compute firewall-rules create allow-ssh-internal-ingress \
  --network=acme-vpc \
  --allow=tcp:22 \
  --source-ranges=${MGMT_SUBNET_IP} \
  --target-tags=${INTERNAL_SSH_TAG}

gcloud compute instances add-tags juice-shop \
  --tags=${INTERNAL_SSH_TAG} \
  --zone=${ZONE}

```

---

### Task 6: Connect to Bastion via IAP and SSH to Juice-Shop

1. In the Google Cloud Console, navigate to **Navigation Menu (☰)** > **Compute Engine** > **VM instances**.
2. Click the **SSH** button next to the **bastion** instance to open the browser-based terminal.
3. In the bastion SSH terminal, retrieve the internal IP of `juice-shop` (or use `192.168.11.2`) and connect:
```bash
ssh 192.168.11.2 #find ip of juice shop in vm instances too if you are confused

```


4. Type `yes` when prompted to verify the host key fingerprint.
5. Once authenticated (`student-...@juice-shop:~$`), type `exit` to disconnect.

---

## Verification Checklist

* [x] `open-access` firewall rule removed.
* [x] `bastion` status is **RUNNING**.
* [x] `allow-ssh-iap-ingress` configured with source `35.235.240.0/20`.
* [x] `allow-http-ingress` open on port `80` to target `juice-shop`.
* [x] `allow-ssh-internal-ingress` restricted to `acme-mgmt-subnet` CIDR.
* [x] Successful SSH session established from `bastion` to `juice-shop`.

