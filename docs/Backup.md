## Windows Endpoints (PC Backups)

Windows workstations are backed up using **Veeam Agent for Microsoft Windows**, utilizing TrueNAS as the primary network backup repository.

* **Schedule:** Weekly full backups combined with daily incremental backups.
* **Retention Policy:** Retains the two most recent backup chains; older backup sets are automatically pruned to save space.

---

## Proxmox Virtual Environment

Virtual machines and containers hosted on Proxmox are backed up directly to the TrueNAS server on a weekly basis.

* **Backup Mode:** Snapshot (ensures zero interruption or downtime for running VMs).
* **Compression:** Zstandard (`zstd`) is used for an optimal balance of speed and storage efficiency.
* **Retention Policy:** Retains the three most recent backups; older backups are automatically purged.
* **Alerting:** Automated notifications are triggered and dispatched upon the completion of the backup job.

---

## TrueNAS Archive (LTO Tape)

To ensure long-term, cold storage and protect against catastrophic data loss, all critical files residing on the TrueNAS system are archived to physical LTO tapes using **Bacula**.


* **Schedule:** Monthly full archival backup.
* **Retention Policy:** Retains the two most recent tape backup sets. Older sets are rotated out and overwritten.
* **Disaster Recovery (Bacula VM):** Following the tape backup process, the Debian virtual machine hosting the Bacula service is exported from TrueNAS and securely stored on two separate USB flash drives. This ensures the backup configuration and catalogs are preserved in case the host fails.

---