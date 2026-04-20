# Paperless-ngx with AI

This guide details the deployment of a fully automated, AI-powered document management system. 

!!! info "Architecture Overview"
    To maximize performance, this deployment is split across two machines:  
    **GPU Machine:** Hosts Ollama and Open WebUI to provide raw AI processing power.  
    **TrueNAS SCALE:** Hosts the Paperless-ngx stack (along with Paperless-AI and Paperless-GPT) as a Docker stack to ensure secure storage and redundancy.

---

## Phase 1: AI Backend Setup (GPU Machine)

First, we will set up the local LLM provider on the machine with GPU processing power.

### 1. Install & Configure Ollama
Install Ollama using the official script:
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

By default, Ollama only listens to `localhost`. We need to configure it to accept connections from the TrueNAS machine. Open the systemd service file:
```bash
sudo nano /etc/systemd/system/ollama.service
```
Add the following line under the `[Service]` section:
```text
Environment="OLLAMA_HOST=0.0.0.0"
```
Reload the daemon and restart the service:
```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

### 2. Install Docker
Install the Docker engine to host Open WebUI:
```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

### 3. Deploy Open WebUI
Create a directory and the `docker-compose.yaml` file for Open WebUI:
```bash
mkdir openwebui
cd openwebui
nano docker-compose.yaml
```

```yaml title="docker-compose.yaml"
services:
  open-webui:
    image: 'ghcr.io/open-webui/open-webui:main'
    restart: unless-stopped
    container_name: open-webui
    volumes:
      - ./open-webui:/app/backend/data
    extra_hosts:
      - 'host.docker.internal:host-gateway'
    ports:
      - '3000:8080'
```
Start the container:
```bash
docker compose up -d
```

### 4. Pull Required Models
Pull the specific models required for document tagging and vision OCR:
```bash
# Lightweight model for metadata/tagging suggestions
ollama pull llama3.2:3b 

# Vision model for improved OCR (Paperless-GPT relies on this)
ollama pull minicpm-v:8b
```

---

## Phase 2: Paperless Stack Setup (TrueNAS SCALE)

The core application stack is deployed on TrueNAS SCALE. 

### 1. Directory Structure
Create the following directory structure on your TrueNAS storage pool:
```text
paperless-stack/
├── paperless-consume 
├── paperless-data 
├── paperless-export
├── paperless-media 
├── paperless-ai-data 
├── paperless-gpt-prompts 
├── paperless-postgres-data 
└── paperless-redis-data 
```

### 2. Docker Compose File
Be sure to replace `<YOUR_PATH>` with the actual absolute path to your dataset.

```yaml title="docker-compose.yaml"
services:
  paperless:
    image: ghcr.io/paperless-ngx/paperless-ngx:latest
    container_name: paperless-ngx
    restart: unless-stopped
    env_file:
      - .env
    depends_on:
      - postgres
      - redis
      - gotenberg
      - tika
    ports:
      - 8777:8000
    volumes:
      - <YOUR_PATH>/paperless-stack/paperless-data:/usr/src/paperless/data
      - <YOUR_PATH>/paperless-stack/paperless-media:/usr/src/paperless/media
      - <YOUR_PATH>/paperless-stack/paperless-export:/usr/src/paperless/export
      - <YOUR_PATH>/paperless-stack/paperless-consume:/usr/src/paperless/consume

  postgres:
    image: postgres:16
    restart: unless-stopped
    container_name: postgres
    env_file:
      - .env
    volumes:
      - <YOUR_PATH>/paperless-stack/paperless-postgres-data:/var/lib/postgresql/data

  redis:
    image: docker.io/library/redis:8
    container_name: redis
    restart: unless-stopped
    env_file:
      - .env
    volumes:
      - <YOUR_PATH>/paperless-stack/paperless-redis-data:/data

  gotenberg:
    image: docker.io/gotenberg/gotenberg:8.25
    container_name: gotenberg
    env_file:
      - .env
    restart: unless-stopped
    command:
      - gotenberg
      - --chromium-disable-javascript=true
      - --chromium-allow-list=file:///tmp/.*

  tika:
    image: docker.io/apache/tika:latest
    container_name: tika
    restart: unless-stopped
    env_file: .env

  paperless-ai:
    image: clusterzx/paperless-ai:latest
    container_name: paperless-ai
    restart: unless-stopped
    depends_on:
      - paperless
    ports:
      - "8778:3000"
    env_file:
      - .env
    volumes:
      - <YOUR_PATH>/paperless-stack/paperless-ai-data:/app/data

  paperless-gpt:
    image: icereed/paperless-gpt:latest
    container_name: paperless-gpt
    restart: unless-stopped
    depends_on:
      - paperless
    ports:
      - "8779:8080"
    env_file:
      - .env
    volumes:
      - <YOUR_PATH>/paperless-stack/paperless-gpt-prompts:/app/prompts

networks: {}
```

### 3. Environment Variables (`.env`)
Prepare your environment variables. *Ensure you replace `<DB_PASSWORD>`, `<TOKEN>` (generated later), and `<OLLAMA_IP>` with your actual values.*

```env title=".env"
# PAPERLESS-NGX
#=================
TZ=Europe/Warsaw
PAPERLESS_REDIS=redis://redis:6379
PAPERLESS_DBHOST=postgres
PAPERLESS_DBNAME=paperless
PAPERLESS_DBUSER=paperless
PAPERLESS_DBPASS=<DB_PASSWORD>
PAPERLESS_TIKA_ENABLED=1
PAPERLESS_TIKA_GOTENBERG_ENDPOINT=http://gotenberg:3000
PAPERLESS_TIKA_ENDPOINT=http://tika:9998

# POSTGRES
#=================
POSTGRES_DB=paperless
POSTGRES_USER=paperless
POSTGRES_PASSWORD=<DB_PASSWORD>

# PAPERLESS-AI
#=================
PAPERLESS_API_URL=http://paperless:8000/api
PAPERLESS_API_TOKEN=<TOKEN>
PAPERLESS_USERNAME=admin
AI_PROVIDER=ollama
OLLAMA_API_URL=http://<OLLAMA_IP>:11434
OLLAMA_MODEL=llama3.2:3b
RAG_SERVICE_URL=http://localhost:8000
RAG_SERVICE_ENABLED=true
SCAN_INTERVAL=*/30 * * * *
PAPERLESS_URL=http://paperless:8000

# PAPERLESS-GPT
#=================
PAPERLESS_BASE_URL="http://paperless:8000"
PAPERLESS_API_TOKEN=<TOKEN>

LLM_PROVIDER="ollama"
LLM_MODEL="llama3.2:3b"
OLLAMA_HOST="http://<OLLAMA_IP>:11434"
OLLAMA_CONTEXT_LENGTH="8192" # Sets Ollama NumCtx (context window)
TOKEN_LIMIT=1000 # Recommended for smaller models

LLM_LANGUAGE="the original language of the document"
OCR_PROVIDER="llm"
VISION_LLM_PROVIDER="ollama"
VISION_LLM_MODEL="minicpm-v:8b"

AUTO_OCR_TAG="paperless-gpt-ocr-auto"
AUTO_TAG="paperless-gpt-auto"
MANUAL_TAG="paperless-gpt-manual"
PDF_OCR_TAGGING="true"
PDF_OCR_COMPLETE_TAG="paperless-gpt-ocr-complete"
PDF_UPLOAD="false"
LOG_LEVEL=debug
```

### 4. Fix TrueNAS Permissions
Before starting the containers, TrueNAS requires specific ownership permissions for the mapped datasets. Open the TrueNAS shell and run:

```bash
sudo chown -R 999:999 <YOUR_PATH>/paperless-stack/paperless-postgres-data
sudo chown -R 999:999 <YOUR_PATH>/paperless-stack/paperless-redis-data
sudo chown -R 1000:1000 <YOUR_PATH>/paperless-stack/paperless-data
sudo chown -R 1000:1000 <YOUR_PATH>/paperless-stack/paperless-media
sudo chown -R 1000:1000 <YOUR_PATH>/paperless-stack/paperless-export
sudo chown -R 1000:1000 <YOUR_PATH>/paperless-stack/paperless-consume
sudo chown -R 1000:1000 <YOUR_PATH>/paperless-stack/paperless-ai-data
sudo chown -R 1000:1000 <YOUR_PATH>/paperless-stack/paperless-gpt-prompts
```

### 5. Deployment via Dockge & API Token Generation

!!! tip "Stack Management via Dockge"
    This entire Paperless-ngx stack is deployed and managed using **Dockge** on TrueNAS SCALE. Instead of using the command line, simply create a new stack in your Dockge web interface, paste the `docker-compose.yaml` and `.env` configurations into the editor, and deploy from there.

!!! danger "Required: API Token Configuration"
    The AI containers cannot communicate with Paperless without an API token. 

1. **Deploy the stack** for the first time via the Dockge interface.
2. Open the Paperless-ngx web UI.
3. Navigate to **Your Profile** and generate an **API Auth Token**. Copy this token.
4. Go back to Dockge, edit the Paperless stack, and paste the token into the `<TOKEN>` placeholders inside the `.env` file editor.
5. **Update/Deploy** the stack again in Dockge so the AI containers restart and register the new token.

---

## Phase 3: Post-Deployment AI Configuration

### Paperless-AI Setup
1. Navigate to the Paperless-AI web interface.
2. Create your initial admin user.
3. Most settings are already configured via the `.env` file, but navigate to the **Advanced** tab.
4. I highly recommend enabling the following AI functions:
    * **Tags Assignment**
    * **Correspondent Detection**
    * **Document Type Classification**
    * **Title Generation**
5. You can also create custom tags and prompts here. The default prompts are excellent, but you can tailor them to your specific homelab needs.
6. *Feature Note:* Paperless-AI allows you to chat directly with the LLM regarding specific documents and view processing statistics.

### Automating OCR with Paperless-GPT & Workflows
Paperless-GPT actively monitors Paperless for specific tags. Once a tag is applied, the document is sent to the GPU machine for OCR and processing.

To fully automate this:  
1. Inside Paperless-ngx, create two new tags: `paperless-gpt-auto` and `paperless-gpt-ocr-auto`.  
2. Navigate to **Workflows** and create a new workflow.  
3. **Trigger:** On document added.  
4. **Action:** Add tags -> select `paperless-gpt-auto` and `paperless-gpt-ocr-auto`.  

Once applied, Paperless-GPT will automatically intercept new documents on upload, process the visual OCR via the `minicpm-v` model, and evaluate the content.