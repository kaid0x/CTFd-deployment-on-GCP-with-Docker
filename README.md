# CTFd on GCP — WSH'26 (intraschool) CTF Infrastructure

> How I deployed a production-ready CTF platform for **Westminster School Hackathon 2026 (WSH'26)** — a 30-participant intraschool CTF & Hackathon event — using CTFd, Docker, and Google Cloud Platform.

---

## Architecture Overview

```
                        Internet
                           │
                    ┌──────▼──────┐
                    │    Nginx    │  (Reverse Proxy)
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼─────┐ ┌────▼────┐ ┌────▼────┐
       │    CTFd    │ │ MariaDB │ │  Redis  │
       │  (Python)  │ │  (DB)   │ │ (Cache) │
       └────────────┘ └─────────┘ └─────────┘
              │
     ┌────────┴────────┐
     │                 │
┌────▼─────┐    ┌──────▼──────┐
│ Gobuster │    │  BurpSuite  │
│Challenge │    │  Challenge  │
│ :8080    │    │  :8081      │
└──────────┘    └─────────────┘

Hosted on: GCP e2-standard-4 (4 vCPU, 16GB RAM)
OS: Ubuntu 22.04 LTS
Region: asia-south1 (Mumbai)
```

---

## Stack

| Component | Technology |
|-----------|-----------|
| CTF Platform | CTFd |
| Reverse Proxy | Nginx |
| Database | MariaDB 10.11 |
| Cache / Sessions | Redis 4 |
| Challenge Apps | Python / Flask |
| Containerization | Docker + Docker Compose |
| Cloud Provider | Google Cloud Platform (GCP) |
| VM | e2-standard-4 (4 vCPU, 16GB RAM) |
| OS | Ubuntu 22.04 LTS |

---

## Files in this repo

| File | What it is |
|---|---|
| `docker-compose.yml` | The compose file used for the event: CTFd 3.8.5's default stack (CTFd, Nginx, MariaDB, Redis) running the prebuilt `ctfd/ctfd:latest` image. Database passwords come from `.env` |
| `.env.example` | Copy to `.env` and set real database passwords |
| `conf/nginx/http.conf` | CTFd's stock Nginx reverse-proxy config, unchanged |

`docker-compose.yml` and `conf/nginx/http.conf` come from [CTFd](https://github.com/CTFd/CTFd) (Apache-2.0). To run it, clone CTFd, then copy these files over the ones in the CTFd folder (the compose file mounts CTFd's source read-only).

---

## Prerequisites

- Google Cloud account with billing enabled
- `gcloud` CLI installed locally
- Basic knowledge of Linux and Docker

---

## Deployment Steps

### 1. Provision the GCP VM

In the GCP Console:

- **Machine type:** e2-standard-4 (4 vCPU, 16GB RAM)
- **OS:** Ubuntu 22.04 LTS
- **Disk:** 20GB
- **Firewall:** Allow HTTP and HTTPS traffic

Or via gcloud CLI:

```bash
gcloud compute instances create ctfd-server \
  --machine-type=e2-standard-4 \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=20GB \
  --zone=asia-south1-c \
  --tags=http-server,https-server
```

### 2. SSH into the VM

```bash
gcloud compute ssh ctfd-server --zone=asia-south1-c
```

### 3. Install Docker

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo usermod -aG docker $USER
newgrp docker
```

Verify:

```bash
docker --version
docker-compose --version
```

### 4. Clone CTFd

```bash
git clone https://github.com/CTFd/CTFd.git
cd CTFd
```

### 5. Configure docker-compose.yml

The default `docker-compose.yml` spins up CTFd, Nginx, MariaDB, Redis, and a permissions container. Key environment variables in the CTFd service:

```yaml
environment:
  - UPLOAD_FOLDER=/var/uploads
  - DATABASE_URL=mysql+pymysql://ctfd:ctfd@db/ctfd
  - REDIS_URL=redis://cache:6379
  - WORKERS=1
```

### 6. Start the Platform

```bash
docker-compose up -d
```

CTFd will be accessible at `http://YOUR_VM_IP`.

### 7. Open Firewall Rules for Challenge Containers

In GCP Console → VPC Network → Firewall → Create Firewall Rule:

- **Name:** `allow-challenges`
- **Direction:** Ingress
- **Source ranges:** `0.0.0.0/0`
- **Protocols/ports:** TCP `8080,8081`

Or via gcloud CLI:

```bash
gcloud compute firewall-rules create allow-challenges \
  --allow=tcp:8080,tcp:8081 \
  --source-ranges=0.0.0.0/0
```

---

## Deploying Challenge Containers

Each challenge is a standalone Flask app containerized with Docker.

### Upload challenge files to VM

From your local machine:

```bash
gcloud compute scp ./gobuster-challenge.zip ctfd-server:~/challenges/ --zone=asia-south1-c
gcloud compute scp ./burpsuite-challenge.zip ctfd-server:~/challenges/ --zone=asia-south1-c
```

### Build and run on the VM

```bash
cd ~/challenges
unzip gobuster-challenge.zip
unzip burpsuite-challenge.zip

cd gobuster-challenge
docker build -t gobuster-challenge .
docker run -d -p 8080:8080 gobuster-challenge

cd ../burpsuite-challenge
docker build -t burpsuite-challenge .
docker run -d -p 8081:8081 burpsuite-challenge
```

Challenge URLs:
- Gobuster: `http://YOUR_IP:8080`
- BurpSuite: `http://YOUR_IP:8081`

---

## Networking & Security

- All services run inside Docker's internal network (`ctfd_internal`)
- Only Nginx is exposed on port 80 to the public
- Challenge containers exposed only on specific ports (8080, 8081)
- GCP firewall rules restrict inbound traffic to necessary ports only
- Static external IP reserved to ensure consistent access URL

---

## Event Stats

| Metric | Value |
|--------|-------|
| Participants | 30 |
| CTF Challenges | 30+ |
| Categories | 7 (Web, RE, Crypto, Stego, Forensics, OSINT, Misc) |
| Challenge Containers | 2 |
| Uptime | 100% during event |

---

## Lessons Learned

- Debian package mirrors can have hash mismatch issues during Docker builds — using a pre-built image (`ctfd/ctfd:latest`) or a different mirror resolves this
- Reserve a static IP on GCP before the event to avoid the IP changing on VM restart
- School/corporate networks behind FortiGuard firewalls will block unrated IPs — get the IP whitelisted by IT in advance
- Running challenges on non-standard ports (8080, 8081) may require additional firewall whitelisting at the network level

---

## Tech Used

`Docker` `Docker Compose` `Google Cloud Platform` `Ubuntu 22.04` `Nginx` `MariaDB` `Redis` `CTFd` `Python` `Flask` `gcloud CLI` `SSH`

---

*Deployed for WSH'26 — Westminster School Hackathon, Dubai 2026*
