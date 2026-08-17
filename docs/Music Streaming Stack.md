# Music Streaming Stack (Gonic & Koito)

This guide covers the deployment of a self-hosted music streaming and scrobbling stack. **Gonic** serves as the lightweight, Subsonic-compatible music server, while **Koito** acts as a local ListenBrainz server to track your listening history privately.

---

## 1. LXC & Storage Preparation

Because i store my music on an SMB share, so LXC container needs access to it.

1. **Create the Container:** Deploy a **Privileged** Debian 13 LXC in Proxmox. *(A privileged container is required to properly map and handle external storage shares via the Proxmox host).*
2. **Mount SMB to Proxmox:** Ensure your SMB share is mounted at the datacenter/node level in the Proxmox GUI (e.g., at `/mnt/pve/<smb_id>`).
3. **Pass the Mount to the LXC:**
Power **down** the LXC container, then open the Proxmox node shell (not the container shell) and run the following command to map the directory:

```bash
pct set <CONTAINER_ID> -mp0 /mnt/pve/<smb_id>,mp=/shared/<name>

```

---

## 2. Docker Installation

Next, install the Docker engine on the Debian 13 LXC.

!!! tip "Automated Deployment via Ansible"
    Instead of installing Docker manually, I utilized my custom Ansible playbook to automate the deployment. You can view the exact playbook used for this in my GitHub repository:
    **[install-docker.yaml](https://github.com/xEska1337/ansible/blob/main/playbooks/docker/install-docker.yaml)**

---

## 3. Docker Compose Deployment

Create a directory for your music stack and define the `docker-compose.yml` file.

```bash
mkdir -p gonic-stack/{data,cache,koito}
cd gonic-stack
nano docker-compose.yml

```

```yaml title="docker-compose.yml"
services:
  gonic:
    image: sentriz/gonic:latest
    container_name: gonic
    environment:
      - TZ=Europe/Warsaw
      - GONIC_SCAN_WATCHER_ENABLED=true
    ports:
      - "4747:80"
    volumes:
      - ./data:/data
      - /shared/music:/music:ro
      - /shared/music/playlists:/playlists
      - ./cache:/cache
    restart: unless-stopped

  koito:
    image: gabehf/koito:latest
    container_name: koito
    ports:
      - "4110:4110"
    volumes:
      - ./koito:/etc/koito
    restart: unless-stopped

```


Start the stack:

```bash
docker compose up -d

```

---

## 4. Post-Install Configuration

With both containers running, we need to configure them and link Gonic to Koito for scrobbling.

### Step A: Configure Koito (Scrobbling)

1. Navigate to **`http://<YOUR_IP>:4110`**.
2. Log in with the default credentials:
* **Username:** `admin`
* **Password:** `changeme`


3. Immediately change your password in the settings.
4. Locate and **copy your API Key** from the Koito dashboard.

### Step B: Configure Gonic (Music Server)

1. Navigate to **`http://<YOUR_IP>:4747`**.
2. Log in with the default credentials:
* **Username:** `admin`
* **Password:** `admin`


3. Change your admin password.
4. Go to the **ListenBrainz** settings section.
5. Set the ListenBrainz URL to point to your local Koito container:
`http://<YOUR_IP>:4110/apis/listenbrainz`
6. Paste the **API Key** you copied from Koito.
7. Finally, trigger a manual **Scan** of your music folder so Gonic can build your library database.

---

## 5. Client Application

Since Gonic uses the standard Subsonic API, it is compatible with dozens of mobile and desktop clients.

I like **Feishin**. 

**[Download Feishin on GitHub](https://github.com/jeffvli/feishin)**

