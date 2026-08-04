# LTO Tape Integration (Bacula on TrueNAS)

To utilize an available PCIe slot on the TrueNAS server, hardware was added to support physical tape backups. An **LSI 9200-8e 6Gbps SAS HBA (IT Mode)** was installed and connected via a SAS cable to a **Dell LTO-7 tape drive**.

The backup orchestration is handled by a Debian virtual machine hosted on TrueNAS, utilizing **Bacula** with the **Bacularis** web interface. The SAS HBA is passed through directly to the VM to grant it hardware-level access to the tape drive.

!!! tip "Preventing Shoe-Shining"
    I strongly recommend using an SSD for the VM installation and configuring a dedicated data spooling disk. Tape drives require a constant, high-speed stream of data. If the data source is too slow, the tape drive will repeatedly stop, rewind, and restart (a phenomenon known as "shoe-shining"), which will quickly destroy both the tape media and the drive mechanism. Spooling to a fast SSD first solves this completely.

---

## Phase 1: Virtual Machine & Hardware Passthrough

### 1. Create the Virtual Machine

Create a new Virtual Machine on TrueNAS with the following parameters:

* **Guest Operating System:** Linux
* **Name:** `bacula`
* **Enable Display (VNC):** `false` (Unchecked)
* **CPU & RAM:** 6 Cores, 3 Threads,Model: Host Passthroug, 4 GiB RAM (Adjust based on your hardware)
* **Primary Disk:** Create new disk image (VirtIO, Size: 20 GiB)
* **Adapter:** VirtIO
* **Attached NIC:** `br0: vSwitch` (You must use a virtual switch to ensure the fastest speeds when accessing datasets, as this bypasses the physical NIC limits).


* **Installation Media:** Attach your Debian 13 ISO from your dataset.

**Configure SPICE Display:**
After creation, edit the VM devices. Add a new device:

* **Type:** Display
* **Display Type:** SPICE
* Set a password and click **Save**.

### 2. Install Debian & Spool Disk

1. Power on the VM and install **Debian 13** using the SPICE display. (A minimal console installation with SSH access is preferred over a graphical interface).
2. Power down the VM completely.
3. **Add Spool Disk:** Add a new ZVOL on your SSD pool for data spooling.
* **Name:** `Bacula-spool-disk`
* **Size:** 500 GiB (Or larger depending on your SSD size)
* **Sparse:** Checked


4. Attach this new disk to the VM.
5. Edit the VM settings and add the LSI SAS HBA as a **PCI Passthrough** device.
6. Power up the VM and verify the SAS device is recognized.

### 3. OS Preparation & Networking

Log into your Debian VM. Install `sudo` and add your user to the group:

```bash
su -
apt install sudo -y
/usr/sbin/usermod -aG sudo <USERNAME>

```

**Set a Static IP:**

```bash
ip a  # Note your interface name (e.g., ens3)
sudo nano /etc/network/interfaces

```

Modify the interface block:

```text
allow-hotplug ens3
iface ens3 inet static
    address <YOUR_IP>/24
    gateway <YOUR_GATEWAY_IP>

```

### 4. Format and Mount the Spool Disk

Find the new 500GB disk, format it, and mount it permanently.

```bash
lsblk
sudo mkfs.ext4 /dev/vdb
sudo mkdir -p /mnt/ssd_spool
sudo mount /dev/vdb /mnt/ssd_spool

```

Make the mount permanent in `fstab`:

```bash
sudo blkid /dev/vdb  # Copy the UUID
sudo nano /etc/fstab

```

Add this line (replace `<YOUR_UUID>`):

```text
UUID=<YOUR_UUID> /mnt/ssd_spool ext4 defaults,discard 0 2

```

---

## Phase 2: Bacularis Installation

To install Bacularis, you must first create a free account on their website to generate a repository authentication file. Log into the Debian VM and execute the following as root:

```bash
sudo su

# Install prerequisites
apt install -y gnupg nfs-common lsscsi rclone stenc

# Import Bacularis GPG Key
wget -qO- https://packages.bacularis.app/bacularis.pub | gpg --dearmor > /usr/share/keyrings/bacularis-archive-keyring.gpg

```

Replace the `<>` fields with your specific Bacularis repository credentials:

```bash
echo "machine https://packages.bacularis.app login <LOGIN> password <PASSWORD>" > /etc/apt/auth.conf.d/bacularis.conf

echo "# Bacularis - Debian 13 Trixie package repository
deb [signed-by=/usr/share/keyrings/bacularis-archive-keyring.gpg] https://packages.bacularis.app/stable/debian trixie main" > /etc/apt/sources.list.d/bacularis-app.list

# Install Bacula and Web GUI
apt update
apt install -y bacula-server bacula-console bacularis bacularis-nginx

```

**Configure Nginx:**

```bash
rm -f /etc/nginx/sites-enabled/default
ln -sf /etc/nginx/sites-available/bacularis.conf /etc/nginx/sites-enabled/

```

Edit the Nginx configuration to listen on port 80 instead of 9097:

```bash
nano /etc/nginx/sites-available/bacularis.conf
# Change: listen 9097;  ->  listen 80;

```

**Set Permissions:**

```bash
chown -R bacula:bacula /mnt/ssd_spool
chmod -R 750 /mnt/ssd_spool
usermod -aG bacula www-data
chown -R bacula:bacula /etc/bacula
chmod -R g+w /etc/bacula

systemctl restart php8.4-fpm
systemctl restart nginx

```

> **Accessing the GUI:** The Bacularis web interface is now available at `http://<VM_IP_ADDRESS>`. The default credentials are **user:** `admin` and **password:** `admin`.

---

## Phase 3: Bacularis Initial Configuration

### 1. Database Connection

Retrieve the auto-generated PostgreSQL database password:

```bash
sudo grep dbpassword /etc/bacula/bacula-dir.conf

```

In the Bacularis GUI setup wizard, configure the database:

* **Database type:** PostgreSQL
* **Database name:** `bacula`
* **Login:** `bacula`
* **Password:** `<RETRIEVED_PASSWORD>`
* **IP address:** `localhost`
* Click **Test the connection** and proceed.

### 2. Sudo & Console Permissions

To allow the web interface to execute Bacula commands, edit the sudoers file:

```bash
sudo visudo

```

Add these lines to the very end of the file:

```text
www-data ALL = (root) NOPASSWD: /usr/sbin/bconsole
www-data ALL = (root) NOPASSWD: /usr/sbin/bdirjson
www-data ALL = (root) NOPASSWD: /usr/sbin/bsdjson
www-data ALL = (root) NOPASSWD: /usr/sbin/bfdjson
www-data ALL = (root) NOPASSWD: /usr/sbin/bbconsjson
www-data ALL = (root) NOPASSWD: /usr/bin/systemctl start bacula-dir
www-data ALL = (root) NOPASSWD: /usr/bin/systemctl stop bacula-dir
www-data ALL = (root) NOPASSWD: /usr/bin/systemctl restart bacula-dir
www-data ALL = (root) NOPASSWD: /usr/bin/systemctl start bacula-sd
www-data ALL = (root) NOPASSWD: /usr/bin/systemctl stop bacula-sd
www-data ALL = (root) NOPASSWD: /usr/bin/systemctl restart bacula-sd
www-data ALL = (root) NOPASSWD: /usr/bin/systemctl start bacula-fd
www-data ALL = (root) NOPASSWD: /usr/bin/systemctl stop bacula-fd
www-data ALL = (root) NOPASSWD: /usr/bin/systemctl restart bacula-fd
bacula ALL=(root) NOPASSWD: /usr/bin/stenc
```

Complete the setup wizard, testing the config and console connections. Set a new Administrator login and password. Under **Advanced Settings**, ensure the port is changed to `80`.

Once inside the main web interface, go to **API Menu > Settings > Actions**, enable it, and check **Use sudo**.

---

## Phase 4: Mounting TrueNAS Datasets

Any TrueNAS dataset that needs to be backed up to tape must first be shared via NFS and mounted inside the Debian VM.

Open the file systems table for editing:

```bash
sudo nano /etc/fstab

```

Add your datasets using the following syntax:

```text
<TrueNAS_IP>:/mnt/<POOL_NAME>/<DATASET_NAME> /mnt/<DATASET_NAME> nfs defaults,x-mount.mkdir 0 0

```

Reload the daemon and mount the newly added datasets immediately:

```bash
sudo systemctl daemon-reload
sudo mount -a

```

---

## Phase 5: Bacula Backup Configuration

### 1. Configure Storage & Spooling

In the Bacularis GUI, go to the **Storage** tab and click **Add tape storage wizard**.

* **API host:** Main
* **Configure:** "No, I do not have it configured yet. I want to add it to Bacula and to Bacularis."
* **Storage name:** `LTO7-Drive`
* **Device type:** Single tape device storage (tape drive)
* **Tape drive device file:** `/dev/nst0` *(Verify with `lsscsi`)*
* **Media type:** `Tape`

To enable the SSD spooling, you must edit the Storage Daemon config file manually:

```bash
sudo nano /etc/bacula/bacula-sd.conf

```

Locate your `Device { ... }` block (named `LTO7-Drive`) and add the Spool parameters:

```conf
Device {
  DeviceType = "Tape"
  RemovableMedia = yes
  AutomaticMount = yes
  MaximumConcurrentJobs = 1
  Name = "LTO7-Drive"
  Description = ""
  ArchiveDevice = "/dev/nst0"
  MediaType = "Tape"
  Spool Directory = "/mnt/ssd_spool"
  Maximum Spool Size = 450G
  Maximum Job Spool Size = 450G
  Maximum Block Size = 1048576
  Maximum File Size = 20G
  Hardware End of Medium = yes
  Fast Forward Space File = yes
}

```

Restart the Storage Daemon:

```bash
sudo systemctl restart bacula-sd

```

### 2. Create the Pool

In Bacularis, create a new Pool:

* **Name:** `LTO7-Tape-Pool`
* **PoolType:** Backup
* **Storage:** Select `LTO7-Drive`
* **Recycle:** Checked
* **VolumeRetention:** `182 Days` (Protects backups from overwriting for 6 months)
* **AutoPrune:** Checked
* **JobRetention:** `14600 Days` (Protects job info for ~40 years)
* **FileRetention:** `14600 Days` (Protects file catalog for ~40 years)

### 3. Create the FileSet

Go to **Director > Configure director > FileSet**. Click **Add Fileset**.

* **Name:** `TrueNAS-Data`
* **Include Block:**
* **Options Block:** Compression: `Lzo`, Signature: `Sha256`
* **File/Directory:** `/mnt/<DATASET_NAME>`


* Save the FileSet.

### 4. Create the Backup Job

Click **New backup job wizard**:

* **Job Name:** `TrueNAS-Data-Backup`
* **JobDefs:** `DefaultJob`
* **Client:** `debian-bacula-fd`
* **FileSet:** `TrueNAS-Data`
* **Storage:** `LTO7-Drive`
* **Spool Data:** Checked
* **Pool:** `LTO7-Tape-Pool`
* **Level:** Full
* **Accurate:** Checked

### 5. Label Your Tapes

1. Insert one of your LTO-7 tapes into the drive.
2. In the Bacularis left sidebar, go to **Volumes**.
3. Click **Label volume**.
4. Select your `LTO7-Tape-Pool` and your `LTO7-Drive`.
5. Enter a Volume Name (e.g., `VOL-001`).

---

## Phase 6: Cloud Catalog Backup & Encryption

Here is the revised MkDocs section. I incorporated the fixes we discussed (running it as the `bacula` user solves the permission issues) and added two critical directives (`RunsOnClient = No` and `FailJobOnError = No`) to ensure the script executes securely and doesn't fail your backup if your internet drops.


### 1. Rclone BackupCatalog to Google Drive

To ensure the Bacula Catalog is safe off-site, configure Rclone.

```bash
sudo -u bacula rclone config

```

1. Type `n` (New remote).
2. **Name:** `gdrive`
3. **Storage:** Locate **Google Drive** and enter its corresponding number.
4. Leave `client_id`, `client_secret`, `root_folder_id`, and `service_account_file` blank (press Enter).
5. **Scope:** Type `1` (Full access).
6. **Edit advanced config?** Type `n`.
7. **Use auto config?** Type `n`.

Rclone will generate an authorization command (e.g., `rclone authorize "drive" "..."`).

* Copy this command, open a terminal on your **personal PC** (with Rclone installed), and paste it.
* Your browser will open. Log into Google and click **Allow**.
* Copy the long token string output in your PC's terminal, return to the Debian VM, paste it, and press Enter.
* Type `n` (No Shared Drive), `y` (Save), and `q` (Quit).

**Add the script to Bacularis:**

In the Bacularis GUI, edit the `BackupCatalog` job and click **Show all directives**.

Ensure your storage targets are set:

* **Pool:** LTO7-Tape-Pool
* **Storage:** LTO7-Drive

Scroll down and add a **RunScript** block with the following parameters:

* **RunsWhen:** `After`
* **RunsOnClient:** `No` 
* **Command:** `sh -c 'rclone copyto /var/lib/bacula/BackupCatalog.bsr gdrive:Bacula_DR_Backups/BackupCatalog-$(date +%%Y-%%m-%%d).bsr'`

Save the job.


### 2. Configure Hardware Tape Encryption (stenc)

Instead of relying on software-based encryption, we will leverage the LTO drive's built-in hardware encryption. This offloads the cryptographic workload directly to the tape drive, ensuring maximum write speeds and lower CPU usage on the VM.

**1. Generate the Encryption Key**

First, create a secure directory and generate a 256-bit encryption key using `stenc`:

```bash
sudo mkdir -p /etc/bacula/certs
cd /etc/bacula/certs

# Generate the 256-bit key
sudo stenc -g 256 -k /etc/bacula/certs/lto-drive.key

# Secure the key permissions
sudo chown root:root /etc/bacula/certs/lto-drive.key
sudo chmod 400 /etc/bacula/certs/lto-drive.key

```

!!! danger "Backup Your Key!"
    Copy this key to a highly secure password manager or offline storage immediately. **If you lose this key, your encrypted tape backups will be permanently unreadable.**
    `bash sudo cat /etc/bacula/certs/lto-drive.key `

**2. Test the Drive Encryption**

Before automating the process, manually verify that the tape drive accepts the encryption key. *(Note: Replace `/dev/sg0` with your actual generic SCSI tape device path, which you can find using `lsscsi -g`)*.

```bash
# Turn encryption ON
sudo stenc -f /dev/sg0 -e on -a 1 -k /etc/bacula/certs/lto-drive.key

# Verify the drive status (Look for Encryption: ON)
sudo stenc -f /dev/sg0 --detail

```

**3. Automate Encryption in Bacula**

To ensure encryption is automatically enabled when a backup starts and disabled when it finishes, edit your backup job in the Bacularis GUI and add two **RunScript** blocks.

In the Bacularis GUI, edit your backup job, click **Show all directives**, and add the following:

**Runscript #1 (Enable Encryption)**

* **RunsWhen:** `Before`
* **RunsOnClient:** `No` *(Unchecked)*
* **Command:** `sudo /usr/bin/stenc -f /dev/sg0 -e on -a 1 -k /etc/bacula/certs/lto-drive.key`

**Runscript #2 (Disable Encryption)**

* **RunsWhen:** `After`
* **RunsOnClient:** `No` *(Unchecked)*
* **Command:** `sudo /usr/bin/stenc -f /dev/sg0 -e off`


**Troubleshooting: Unrecognized Labeled Tapes**

If you insert a previously labeled tape but Bacula fails to recognize it (or throws an error when trying to read the volume label), it is highly likely that the hardware encryption key is not currently loaded into the tape drive's memory.

You can confirm this by checking the kernel logs for tape-related read errors:

```bash
sudo dmesg | grep -i st0 | tail -n 15

```

To resolve this, manually push the encryption key to the drive before instructing Bacula to mount or read the tape:

```bash
sudo stenc -f /dev/sg0 -e on -a 1 -k /etc/bacula/certs/lto-drive.key

```

---

## Useful Administrative Commands

**Verify PCI/SCSI Devices:**

```bash
lsscsi

```

**Disk I/O stats:**

```bash
iostat -d -h -m 2

```

**Access the Bacula Console:**

```bash
cd /usr/sbin
./bconsole

```

**Common Bconsole Commands:**

* `status jobid=1234` (Checks the status of a specific job)
* `list files jobid=1234` (Lists the files processed in a specific job)
* `restore` (Start restore)
* `list jobs` (List all jobs)

---

## Restoring Files from Backup

!!! tip "GUI vs. CLI Restores"
    Before restoring, you need to create a restore job template. You can do that in the Bacularis web interface. However, if you are restoring from a very large backup job, it is highly recommended to use the `bconsole` CLI interface for restore process, as it handles building and navigating massive file trees much faster.

**1. Enter Bconsole:**

Access your Debian VM and open the Bacula console:
```bash
cd /usr/sbin
./bconsole
```

**2. Locate Your Job ID:**

List all previous jobs to find the ID of the backup you want to restore from:
```text
*list jobs
```

**3. Initiate the Restore:**

Start the restore process:
```text
*restore
```
Choose the appropriate option for your needs. To restore specific files, select **option 3** (*Enter list of comma separated JobIds to select*), then input your target Job ID.

**4. Select Files:**

Wait for the virtual file tree to build. Once loaded, you can navigate through the backup (using standard `cd` and `ls` commands) and select files to restore:
```text
*mark "your_file_or_directory"
```

**5. Finalize Selection:**

When you have marked all desired files, type:
```text
*done
```

**6. Review and Run:**

Bacula will display a summary of the restore job. 

* To restore to the **original location**, you don't need to change anything. 
* To restore to a **different location**, type `mod`, select the `Where` option to modify the restore path, and specify the new directory.

Finally, run the job and wait for it to complete.

---

## Disaster Recovery Strategies

To ensure maximum resilience and prevent a single point of failure, I maintain two separate disaster recovery methods for the backup infrastructure itself. You can rebuild using Bacula's native catalog backup, or perform a full bare-metal restore by exporting the TrueNAS VM disk.

### Method 1: Bacula Catalog Backup (Native)

Bacula automatically includes a `BackupCatalog` job in its default configuration. This job creates a database dump (containing all your file records, volumes, job histories, and configuration paths) and writes it directly to your storage media. To ensure you can actually find and restore this catalog during a disaster, you must securely store the generated `.bsr` (bootstrap) file off-site or on a separate machine. This file marks the exact physical position of the catalog dump on the tape.

!!! tip "Why keep both?"
    While restoring the catalog requires you to manually rebuild the Debian VM and reinstall Bacula first, having the `BackupCatalog` on tape is invaluable. If your TrueNAS server completely fails and you need to move your tape drive to a completely different hypervisor or bare-metal Linux machine, the catalog dump allows you to resume operations without needing to import a TrueNAS-specific ZVOL.

### Method 2: Full VM Export & Import (ZVOL)

To protect the entire Bacula configuration, web interface, and OS layer simultaneously, the Debian VM is exported directly from the TrueNAS ZFS volumes (ZVOLs) to external storage. This is the fastest way to recover if the TrueNAS server is still healthy but the VM becomes corrupted.


#### Prerequisites (TrueNAS Shell)

1. Ensure your TrueNAS user has SSH access enabled.
2. Allow `sudo` command execution for your user.
3. Enable SSH with password login (if not using keys).
4. Create a temporary transfer directory: `mkdir /mnt/transfer`
5. Grant ownership to your user: `sudo chown <USERNAME>:<USERNAME> /mnt/transfer`

#### Exporting the VM

!!! danger "CRITICAL"
    You **must** power down the Debian VM completely before proceeding. Taking a snapshot of a running VM without guest-agent quiescing can lead to severe data corruption.

Run the following command in the TrueNAS shell. This one-liner generates a timestamp, takes a fast ZFS snapshot, and pipes the serialized data through `pv` (for a progress bar) and `gzip` for compression:

```bash title="Export ZVOL to external storage"
TS=$(date +"%Y-%m-%d_%H-%M-%S") && sudo zfs snapshot <POOL_NAME>/<ZVOL_NAME>@$TS && sudo zfs send <POOL_NAME>/<ZVOL_NAME>@$TS | pv | gzip > /mnt/transfer/bacula_vm_backup_$TS.gz
```

Once completed, use a secure transfer tool (like WinSCP or `rsync`) to securely copy the resulting `.gz` file to an external USB drive or off-site storage location.

#### Importing / Restoring the VM

To import the VM and create a new ZVOL from your compressed backup, use the following command (be sure to replace `<TIMESTAMP>` with your actual file's timestamp):


```bash title="Restore ZVOL from backup"
pv /mnt/transfer/bacula_vm_backup_<TIMESTAMP>.gz | gunzip -c | sudo zfs recv <POOL_NAME>/<ZVOL_NAME>-restored
```
Once the `zfs recv` command finishes, you can attach this newly created `<ZVOL_NAME>-restored` block device to a new Virtual Machine in TrueNAS and boot it up.

---

#### Troubleshooting (Missing GRUB After Import)

If the restored VM boots into the UEFI shell because GRUB is missing, follow these steps inside the UEFI shell prompt:

1. Map the filesystems to locate the EFI partition:

```text
map

```

2. Look for the device mapped as `FS0:` (or `FS1:`) and change to it:

```text
FS0:
cd EFI\debian
dir

```

3. If you see `grubx64.efi`, boot into Debian manually by typing:

```text
grubx64.efi

```

**Fixing the Automatic Boot Entry:**
Once you confirm Debian can boot, you can permanently fix the boot menu from the UEFI shell:

```text
# View existing boot entries
bcfg boot dump  

# Add the Debian bootloader (Replace <YOUR_DEVICE_ID> with FS0, FS1, etc.)
bcfg boot add 0 <YOUR_DEVICE_ID>:\EFI\debian\grubx64.efi "Debian Boot"

# Reboot the VM
reset

```