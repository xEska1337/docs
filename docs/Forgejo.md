# Forgejo Deployment (TrueNAS SCALE)

This guide covers the deployment of **Forgejo**, a self-hosted lightweight software forge, using the native TrueNAS SCALE Apps ecosystem. Hosting a local Git repository ensures that all homelab configuration files, scripts, and Docker Compose stacks are securely backed up and version-controlled on-premises.

---

## 1. Storage Preparation

Before installing the application, dedicated datasets must be created on the TrueNAS storage pool to ensure data persists across container restarts and updates.

Create a parent dataset for the application, and two child datasets for the application data and database:

```text
<POOL_NAME>/
└── forgejo/
    ├── data
    └── db

```

---

## 2. App Installation & Configuration

Navigate to **Apps > Discover Apps** in the TrueNAS GUI, search for **Forgejo**, and begin the installation process. Configure the following key sections:

### Network & URL Configuration

* **Root URL:** Enter the full public URL you intend to use (e.g., `https://git.<YOUR_DOMAIN>.com`).

!!! info "Reverse Proxy Integration"
    To have a valid SSL cert, a DNS rewrite and reverse proxy rule must be created. Refer back to the **[Traefik Deployment](Traefik.md)** page.

### Database Setup

* **Database Password:** Generate a strong, secure password for the internal PostgreSQL database and save it to your password manager. The app will automatically provision the database using these credentials.

### Storage Mapping

Map the datasets created in Step 1 to the application:

* **Forgejo Data Storage:** Map to `/mnt/<POOL_NAME>/forgejo/data`
* **Postgres Data Storage:** Map to `/mnt/<POOL_NAME>/forgejo/db`

### Permissions

* **Host Path Permissions:** Ensure you check the box for **Automatic Permissions** (or `Enable ACL/Permissions`). This allows TrueNAS to automatically adjust the dataset ownership so the Forgejo container can write to the disks without throwing "Permission Denied" errors.

Once configured, click **Install** and wait for the application to reach an "Active" state.

---

## 3. Initial Admin Setup (First Login)

Once the Forgejo application is active, navigate to the web interface using your configured URL or local TrueNAS IP.

!!! danger "Security Warning: The First User Rule"
    In Forgejo, the **very first user account registered on the web interface automatically becomes the server Administrator**.

*Ensure you navigate to your Forgejo instance and register your account *immediately* after the container starts.*

**To secure your instance:**

1. Click **Register** in the top right corner of the Forgejo homepage.
2. Fill in your desired Admin Username, Email, and strong Password.
3. Once logged in, click your profile icon in the top right and navigate to **Site Administration**.
4. *Recommended:* Go to **Users > Settings** and disable public registration to prevent users from creating accounts on your server.

---

## 4. Post-Install: Repository Mirroring

One of the most powerful features of Forgejo for a homelab environment is its built-in **Migration & Mirroring** capability.

When creating a new migration (e.g., pulling a repository from GitHub or GitLab), you can check the **"This repository will be a mirror"** option.

!!! tip "Why use mirrors?"
    Mirroring allows you to continuously sync public or private repositories (provided you supply an access token) directly to your TrueNAS disks. If the original repository is ever deleted, taken offline, or modified maliciously on the upstream provider, your homelab retains a fully functional, exact clone of the code.

---