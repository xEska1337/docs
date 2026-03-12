# LTO Tape Integration (Bacula on TrueNAS)

To utilize an available PCIe slot on the TrueNAS server, hardware was added to support physical tape backups. An **LSI 9200-8e 6Gbps SAS HBA (IT Mode)** was installed and connected via a SAS cable to a **Dell LTO-7 tape drive**.

The backup orchestration is handled by a Debian virtual machine hosted on TrueNAS, utilizing **Bacula** with the **Bacularis** web interface. The SAS HBA is passed through directly to the VM to grant it hardware-level access to the tape drive.

---

## Phase 1: Virtual Machine & Hardware Passthrough

1. Create a new Virtual Machine on TrueNAS.
2. Install **Debian 12 (Bookworm)** as the guest operating system.
3. Power down the VM completely.
4. Edit the VM settings and add the LSI SAS HBA as a **PCI Passthrough** device.
5. Power up the VM and verify the device is recognized.

---

## Phase 2: Bacularis Installation

Log into the Debian VM and execute the following commands as the root user to install Bacula and the Bacularis web GUI.

```bash
sudo su
```

```bash
wget -qO- https://packages.bacularis.app/bacularis.pub | gpg --dearmor > /usr/share/keyrings/bacularis-archive-keyring.gpg
```

```bash
echo "# Bacularis - Debian 12 Bookworm package repository
deb [signed-by=/usr/share/keyrings/bacularis-archive-keyring.gpg] https://packages.bacularis.app/stable/debian bookworm main" > /etc/apt/sources.list.d/bacularis-app.list
```

```bash
apt update
apt install bacularis bacularis-nginx
```

```bash
ln -s /etc/nginx/sites-available/bacularis.conf /etc/nginx/sites-enabled/
systemctl restart nginx
```

> **Accessing the GUI:** The Bacularis web interface is now available at `http://<VM_IP_ADDRESS>:9097`. The default credentials are user: `admin` and password: `admin`.

---

## Phase 3: Mounting TrueNAS Datasets

Any TrueNAS dataset that needs to be backed up to tape must first be shared via NFS and mounted inside the Debian VM.

Open the file systems table for editing:

```bash
sudo nano /etc/fstab

```

Add your datasets using the following syntax:

```text
<TrueNAS_IP>:/mnt/<POOL_NAME>/<DATASET_NAME> /mnt/<DATASET_NAME> nfs defaults 0 0

```

Mount the newly added datasets immediately:

```bash
sudo mount -a

```

---

## Useful Administrative Commands

**Verify PCI/SCSI Devices:**

```bash
lsscsi

```

**Access the Bacula Console:**

```bash
cd /usr/sbin
./bconsole

```

**Common Bconsole Commands:**

* `status jobid=1234` (Checks the status of a specific job)
* `list files jobid=1234` (Lists the files processed in a specific job)

---

## Disaster Recovery (VM Export & Import)

To protect the Bacula configuration, the Debian VM is exported directly from the TrueNAS ZFS volumes (ZVOLs) to external storage.

### Prerequisites (TrueNAS Shell)

1. Ensure your TrueNAS user has SSH access enabled.
2. Allow `sudo` command execution for your user.
3. Enable SSH with password login (if not using keys).
4. Create a temporary transfer directory: `mkdir /mnt/transfer`
5. Grant ownership to your user: `sudo chown <USERNAME>:<USERNAME> /mnt/transfer`

### Exporting the VM

**CRITICAL:** Power down the Debian VM before proceeding to avoid data corruption!

Run the following command in the TrueNAS shell to compress and export the ZVOL:

```bash
sudo dd if=/dev/zvol/<POOL_NAME>/<ZVOL_NAME> bs=1M status=progress | gzip -1 > /mnt/transfer/<ZVOL_NAME>.raw.gz

```

Once completed, use a tool like WinSCP to securely copy the `.raw.gz` file to your external USB drives.

### Importing / Restoring the VM

To identify existing volumes, use: `sudo zfs list -t volume`

1. Create a new ZVOL for the restored VM (adjust the size as needed):

```bash
sudo zfs create -V 20G <POOL_NAME>/<ZVOL_NAME>-restored

```

2. Unzip and write the image to the new ZVOL:

```bash
sudo gunzip -c /mnt/transfer/<ZVOL_NAME>.raw.gz | sudo dd of=/dev/zvol/<POOL_NAME>/<ZVOL_NAME>-restored bs=1M status=progress

```

---

## Troubleshooting (Missing GRUB After Import)

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