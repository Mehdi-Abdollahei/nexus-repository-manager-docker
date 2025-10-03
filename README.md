# 🚀 Setup Nexus Repository 3.x as a Docker Registry (v3.84.1)

This guide explains how to set up **Nexus Repository OSS 3.84** with Docker Compose and configure other Docker hosts to use it as a private Docker Registry.

### Follow me

[![YouTube](https://img.shields.io/badge/YouTube-%23FF0000?style=for-the-badge&logo=youtube&logoColor=white)](https://www.youtube.com/@mehdi_devops_pro)
[![Instagram](https://img.shields.io/badge/Instagram-%23E1306C?style=for-the-badge&logo=instagram&logoColor=white)](https://www.instagram.com/mehdi.devops.pro/)

---

## 📦 1. Prepare the Nexus with docker compose

```bash
# Create data directory
sudo mkdir -p /srv/nexus-data
sudo chown -R 200:200 /srv/nexus-data

# Create compose directory
sudo mkdir -p /opt/docker-compose/nexus
cd /opt/docker-compose/nexus

# This file uploaded in this github project
touch /opt/docker-compose/nexus/docker-compose.yml 

```
## ⚙️ 2. Docker Compose File
**vim /opt/docker-compose/nexus/docker-compose.yml**
```yaml
version: "3.8"                                   # Compose file format; v3.8 is widely supported

services:
  nexus:                                         # Service name (your Nexus app)
    image: sonatype/nexus3:latest                # Pin a known-good version (avoid :latest drift)
    container_name: nexus                        # Stable name makes exec/logs easier
    restart: unless-stopped                      # Auto-restart on reboot/crash; not if you stopped it
    ports:
      - "8081:8081"                              # Expose UI/API on the host (http://HOST:8081)
      - "5000:5000"                              # (Optional) expose if you create a Docker repo on port 5000
    environment:
      TZ: "UTC"                                  # Consistent timestamps in logs
      INSTALL4J_ADD_VM_PARAMS: >-                # JVM settings for performance & stability
        -Xms2g -Xmx2g                            # Heap: adjust as your repos/metadata grow
        -XX:MaxDirectMemorySize=2g               # Off-heap buffers for I/O
        -Djava.util.prefs.userRoot=/nexus-data/javaprefs
    volumes:
      - /srv/nexus-data:/nexus-data:rw           # Persist EVERYTHING (configs/db/blobs/logs) on host
    ulimits:
      nofile:
        soft: 65536                              # Lots of files/sockets under load
        hard: 65536
      nproc: 65535                               # Allow enough JVM threads
    security_opt:
      - no-new-privileges:true                   # Prevent privilege escalation inside the container
    cap_drop:
      - ALL                                      # Drop Linux capabilities (Nexus doesn’t need extra caps)
    stop_grace_period: 120s                      # Give JVM time to shut down cleanly
    healthcheck:                                 # Let Docker know when Nexus is actually ready
      test: ["CMD-SHELL", "curl -fsS http://localhost:8081/service/rest/v1/status | grep -q AVAILABLE"]
      interval: 30s
      timeout: 5s
      retries: 20
      start_period: 90s
    logging:                                     # Prevent log files from filling the disk
      driver: json-file
      options:
        max-size: "50m"
        max-file: "3"
```

## 🛠️ 3. Configure Nexus Repositories (UI)

Login → Administration:

**Enable Realm**

Go to: Security → Realms

Move Docker Bearer Token Realm to Active.


**Create Blob Store (optional but recommended)**

Repository → Blob Stores → Create

Type: File → Name: docker-blob


**Create Repositories**

______________________________________

docker (hosted) → for your own images

______________________________________

docker (proxy) → proxy to Docker Hub 
- remote storage: https://registry-1.docker.io
- Find **use docker hub** then tick the check box
  
______________________________________

docker (group) → combines hosted + proxy

- Members: docker-hosted first, then docker-hub
- Find **Other Connectors**
- Set HTTP: **5000**


## 🖥️ 4. Verify Locally on Nexus Host
```bash
curl -i http://127.0.0.1:5000/v2/
# Expected: 200 OK or 401 Unauthorized (means registry is alive)
```

## 🔑 5. Configure Other Docker Hosts to use nexus
```bash
# On each client host:
touch /etc/docker/daemon.json

vim /etc/docker/daemon.json

{
  "insecure-registries": ["<NEXUS-IP>:5000"]
}

# Restart Docker:
sudo systemctl restart docker
docker logout
docker login <NEXUS-IP>:5000
```

## 🐳 6. Pull from Proxy (Docker Hub Mirror)
```bash
docker pull <NEXUS-IP>:5000/library/nginx:latest

```
## 🏗️ 7. Push Your Own Images (Hosted)
```bash
# For Hosted You Should Use Other Port like 5001 then Use Revers Proxy For Route Traffic To 5000 And 5001 And Your Client Login To 443
docker tag <NEXUS-IP>:5000/library/busybox:latest <NEXUS-IP>:5001/myteam/hello:1.0
docker push <NEXUS-IP>:5001/myteam/hello:1.0

```
## 🖼️ Architecture Diagram
```mermaid
flowchart TD
    subgraph DeveloperHost["Developer Host"]
        A["docker push myteam/hello:1.0"]
        B["docker pull myteam/hello:1.0"]
    end

    subgraph Nexus["Nexus Repository OSS"]
        H["Hosted Repo (push only)"]
        P["Proxy Repo (Docker Hub cache)"]
        G["Group Repo (pull only)"]
    end

    subgraph Hub["Docker Hub"]
        R["registry-1.docker.io"]
    end

    subgraph Cluster["Docker Cluster"]
        S1["Server 1"]
        S2["Server 2"]
        S3["Server 3"]
    end

    %% Connections
    A -->|Push| H
    B -->|Pull| G
    G -->|Check hosted first| H
    G -.->|If not found use proxy| P
    P --> R

    %% Cluster pulls from Nexus
    S1 -->|Pull image| G
    S2 -->|Pull image| G
    S3 -->|Pull image| G

    %% Styling
    style H fill:#e1f7d5,stroke:#333,stroke-width:1px
    style P fill:#cce5ff,stroke:#333,stroke-width:1px
    style G fill:#fff3cd,stroke:#333,stroke-width:1px
    style S1 fill:#f8d7da,stroke:#333,stroke-width:1px
    style S2 fill:#f8d7da,stroke:#333,stroke-width:1px
    style S3 fill:#f8d7da,stroke:#333,stroke-width:1px
