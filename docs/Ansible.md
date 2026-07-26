# Ansible & Semaphore UI Deployment

This guide covers the setup of a centralized automation server using **Ansible** and **Semaphore UI** (a modern, open-source web UI for Ansible). This stack allows you to manage homelab configurations, execute playbooks visually, and schedule automated tasks.

---

## Phase 1: Base System & Ansible Installation

### 1. LXC Preparation

Create a fresh Debian LXC container in Proxmox. Once booted, access the console and update the system:

```bash
apt update && apt upgrade -y

```

### 2. Install Ansible

Install the core Ansible engine:

```bash
apt install ansible -y

```

### 3. SSH Configuration (Target Nodes)

To allow your machine to manage this LXC (and for Ansible to manage other nodes), you need to configure SSH keys.

Temporarily allow root password login to easily copy your keys over:

```bash
nano /etc/ssh/sshd_config

```

Find `PermitRootLogin` and set it to `yes`. Restart the SSH service:

```bash
systemctl restart ssh

```

**From your personal workstation:**
Generate an SSH key (if you don't already have one) and copy it to the Ansible LXC.

* **Linux / macOS:**
```bash
ssh-keygen -t ed25519 -C "your-email-or-pc-name"
ssh-copy-id root@<YOUR_LXC_IP>

```

* **Windows (PowerShell):**
```powershell
ssh-keygen -t ed25519 -C "your-email-or-pc-name"
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh root@<YOUR_LXC_IP> "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

```

!!! danger "Security Cleanup"
    Once your SSH key is confirmed working, go back into `/etc/ssh/sshd_config` on the LXC, set `PermitRootLogin prohibit-password`, and restart the SSH service to secure the container.

---

## Phase 2: Ansible Project Structure

!!! tip "Development Environment"
    I highly recommend using **VSCodium** (or VS Code) configured with the Remote-SSH extension to connect directly to the LXC. This provides a visual editor for managing your YAML files.

Create your project folder and define the baseline structure.

### 1. Inventory (`machines.yaml`)

Create a `config` folder and inside it, create `machines.yaml`. This is where you will define your host groups and devices.

Add the local Ansible server as the first host:

```yaml title="config/machines.yaml"
all:
  children:
    local:
      hosts:
        master_ansible:
          ansible_host: localhost
          ansible_connection: local

```

### 2. Default Config (`ansible.cfg`)

In the root of your project folder, create an `ansible.cfg` file and set your custom inventory as the default:

```ini title="ansible.cfg"
[defaults]
inventory = ./config/machines.yaml

```

### 3. Playbooks & Variables

* **Playbooks:** Create a `playbooks/` folder. You can divide this into sub-folders based on task type or target system (e.g., `playbooks/truenas/`, `playbooks/updates/`).
* **Variables:** I strongly recommend keeping variables out of your playbooks. Use the `config/group_vars/` directory. Create `all.yaml` for global variables, or create group-specific files like `truenas.yaml` for variables that only apply to the TrueNAS group.

!!! example "My Ansible Playbooks"
    You can check out my personal collection of playbooks in my GitHub repository:
    **[https://github.com/xEska1337/ansible](https://github.com/xEska1337/ansible)**

!!! warning "Git Integration & Secrets"
    It is best practice to push your playbooks to a Git repository so Semaphore can pull them automatically. **Remember to configure a `.gitignore` file** so you don't accidentally publish sensitive configurations or plain-text passwords!

---

## Phase 3: Docker & Semaphore UI Deployment

### 1. Install Docker

Install the Docker engine on the LXC:

??? info "Docker Official Docs"
    [Docker Engine Installation on Debian](https://docs.docker.com/engine/install/debian/)

```bash
apt update
apt install ca-certificates curl -y
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

apt update
apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

```

### 2. Deploy Semaphore UI

Create the Semaphore directory and fix permissions so the container can write to the config volume:

```bash
mkdir semaphore
cd semaphore
mkdir semaphore_config
chown -R 1001:1001 ./semaphore_config

```

Create an `.env` file containing your database password:

```env title=".env"
SEMAPHORE_ADMIN_PASSWORD=your_secure_password

```

Create the Docker Compose file:

```yaml title="docker-compose.yml"
services:
  semaphore:
    image: semaphoreui/semaphore:v2.18.12
    container_name: semaphore
    ports:
      - "3000:3000"
    environment:
      SEMAPHORE_DB_DIALECT: sqlite
      SEMAPHORE_ADMIN: admin
      SEMAPHORE_ADMIN_PASSWORD: ${SEMAPHORE_ADMIN_PASSWORD}
      SEMAPHORE_ADMIN_NAME: Admin
      SEMAPHORE_ADMIN_EMAIL: admin@localhost
    volumes:
      - ./semaphore_config:/etc/semaphore
    restart: unless-stopped

```

Start the container:

```bash
docker compose up -d

```

---

## Phase 4: Semaphore UI Configuration

Access the web interface at `http://<YOUR_LXC_IP>:3000` and log in. Semaphore needs to be linked to your Git repository and your server infrastructure.

### 1. Key Store (Credentials)

Navigate to **Key Store** to add your credentials.

* **Git Key:** If your playbook repository is private, add your Git Personal Access Token or Git SSH key here.
* **Server Key (SSH):** Click **New Key**, select **SSH Key**, and paste the private SSH key that your LXC uses to connect to other servers.
* *To locate this key on the LXC, run:* `cat ~/.ssh/id_ed25519`



### 2. Link the Repository

Tell Semaphore where your playbooks live.

* Go to **Repositories > New Repository**.
* **Name:** `Ansible Playbooks` (or similar).
* **Clone URL:** Paste your Git Clone URL.
* **Key:** Select the Git Key from the Key Store (if the repo is private).

### 3. Define the Inventory

Semaphore needs to know the layout of your infrastructure.

* Go to **Inventory > New Inventory**.
* **Name:** `machines`.
* **SSH Key:** Select the Server SSH Key you created in Step 1.
* **Type:** Select **Static YAML** and paste the exact contents of your `machines.yaml` file into the text box.

### 4. Variable Groups

Semaphore manages variables in "Variable Groups" which can be applied to specific tasks.

!!! note "JSON Format Required"
    Semaphore requires variables in the GUI to be stored in **JSON format**, not YAML. You can either convert your YAML to JSON before pasting, or use the built-in table function to add variables row by row.

* Go to **Variable Groups > New Variable Group**.
* Create a group named `All` for global variables, and create separate groups for specific hosts/clusters.

### 5. Create a Task Template

A Task Template binds everything together into a clickable "Run" button.

* Go to **Task Templates > New Template** and select **Ansible Playbook**.
* **Name:** Give the task a descriptive name (e.g., "Update TrueNAS").
* **Playbook Name:** Enter the relative path to the playbook in your Git repository (e.g., `playbooks/truenas/update.yaml`).
* **Inventory:** Select your `machines` inventory.
* **Repository:** Select your linked Git repository.
* **Variable Group:** Attach any necessary Variable Groups.
* Click **Create**.

You can now click **Run** on the template to execute the playbook visually from the web dashboard!

### 6. Scheduling Automated Runs

Once your Task Templates are confirmed to run successfully, you can automate them using standard Cron scheduling. This is perfect for routine maintenance like updating LXC containers or triggering nightly backups.

* Go to **Schedule** and click **New Schedule**.
* **Template:** Select the Task Template you created in Step 5.
* **Cron Expression:** Enter the schedule using standard Cron syntax.
* *Example:* `0 2 * * *` (Runs daily at 2:00 AM)
* *Example:* `0 3 * * 0` (Runs every Sunday at 3:00 AM)


* Click **Save**. Semaphore will now automatically execute the playbook based on your defined schedule.