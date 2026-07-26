# Wealthfolio Deployment (TrueNAS SCALE)

This guide covers the deployment of **Wealthfolio**, a self-hosted personal finance and portfolio tracking dashboard, utilizing the native TrueNAS SCALE Apps ecosystem.

---

## 1. Storage Preparation

Before installing the application, a dedicated dataset must be created on your TrueNAS storage pool to ensure your financial data persists securely across container restarts and updates.

1. Navigate to **Datasets** in TrueNAS.
2. Create a new parent dataset for the application:

```text
<POOL_NAME>/
└── wealthfolio

```

!!! tip "Dataset Preset"
    When creating this dataset in the TrueNAS GUI, ensure you set the **Dataset Preset** to **Apps**. This automatically configures the optimal ACLs and record sizes for containerized workloads.

---

## 2. App Installation & Configuration

Navigate to **Apps > Discover Apps** in the TrueNAS GUI, search for **Wealthfolio**, and begin the installation process. You will need to configure the following key sections:

### Security Secrets

Wealthfolio requires a secure 32-character base64 secret key for internal encryption. Open any Linux terminal (or the TrueNAS shell) and generate one using:

```bash
openssl rand -base64 32

```

Copy the output string and paste it into the **Secret Key** field in the app configuration.

### Network & Reverse Proxy (CORS)

If you are putting Wealthfolio behind reverse proxy, you must explicitly allow the domain so the web interface can communicate with the backend.

* Under the Network/CORS settings, add your full public URL to the **CORS Allow Origins** list (e.g., `https://wealthfolio.<YOUR_DOMAIN>.com`).

### Authentication Setup

To protect your financial data, enable the **Require Authentication** checkbox. Wealthfolio requires your password to be hashed using the Argon2 algorithm before pasting it into the configuration.

!!! info "Argon2 Requirement"
    To generate the hash, you must have the `argon2` package installed on a Linux machine. You can install it using: `sudo apt install argon2 -y`

Run the following command, replacing `your-password` with your desired login password, and `yoursalt16chars!` with a random 16-character string:

```bash
printf 'your-password' | argon2 yoursalt16chars! -id -e

```

Copy the resulting hashed string (it will look something like `$argon2id$v=19...`) and paste it into the **Password Hash** field in the TrueNAS app settings.

### Storage Mapping

Navigate to the Storage section of the app config and map the dataset you created in Step 1 to the application's data directory.

Once all sections are configured, click **Install** and wait for the application to reach an "Active" state.

---

## 3. Accessing Wealthfolio

Once the container is fully deployed and active, you can access the web interface via:

* **Local IP:** `http://<YOUR_TRUENAS_IP>:30880/`
* **Reverse Proxy:** `https://wealthfolio.<YOUR_DOMAIN>.com`

Log in using the plain-text password you hashed earlier, and you can begin adding your accounts, assets, and transactions!