# Welcome to Szymon Kaczmarek Homelab Documentation

Welcome to the documentation for my personal datacenter and staging environment. 


This homelab serves as my primary sandbox for continuous learning, infrastructure administration, and self-hosting core services. I built this environment to gain hands-on experience with virtualization, network security, containerization, and storage management outside of a standard corporate environment.

---

## Architecture & Tech Stack Overview

Here is a high-level overview of the technologies I deploy and manage across the lab:

| Category | Technologies |
| :--- | :--- |
| **Virtualization & OS** | Proxmox VE, LXC (Linux Containers), Debian Linux, Windows |
| **Containerization** | Docker, Docker Compose |
| **Networking & Security**| OPNsense, UniFi, 802.1Q VLANs, Firewall Rules, WireGuard, Traefik, Netbird |
| **Storage** | TrueNAS SCALE, ZFS, NFS, SMB |
| **Core Services** | Home Assistant (IoT/Automation), Blue Iris (NVR), CodeProject.AI Server, AdGuard Home (DNS) |

---

## Hardware Summary

My infrastructure is distributed across purpose-built hardware to ensure stability, proper resource allocation, and physical network separation:

* **Storage Node:** Supermicro Server running TrueNAS SCALE.
* **Compute Nodes:** 
    * 1x Mini PC running Proxmox VE (hosting VMs and LXC containers).
    * 1x Standard PC running Debian (utilized for testing and game servers).
* **Physical Security & AI:** 1x Small Form Factor (SFF) PC running Windows (dedicated to Blue Iris NVR and local CodeProject.AI processing).
* **Network Edge & Backbone:** 
    * 1x Dedicated Mini PC running OPNsense (Bare-metal router/firewall).
    * 1x PoE Switch (providing power/data for APs and IP cameras).
    * 1x Standard Distribution Switch.
* **Edge Devices:** 3x UniFi Access Points, Reolink IP Cameras, PC.

---

## Key Infrastructure Highlights

1.  **Strict Network Segmentation (VLANs):** The network is purposefully segmented using VLANs managed by OPNsense. This applies the principle of least privilege, entirely isolating IoT devices (Home Assistant) and Reolink surveillance cameras from trusted internal traffic and the public internet.
2.  **Locally Hosted AI Computer Vision:** Integrated CodeProject.AI Server with Blue Iris to perform local, on-premise object detection. This eliminates reliance on cloud APIs for security, drastically improving response times and ensuring strict data privacy.
3.  **Optimized Compute & Virtualization:** I leverage a mix of heavy Virtual Machines and lightweight LXC containers via Proxmox, alongside Docker on Debian, to optimize hardware resources based on the specific needs of each application. 
4.  **Centralized Storage with ZFS:** Utilizing enterprise-grade Supermicro hardware running TrueNAS SCALE to provide resilient, ZFS-backed storage to the rest of the compute nodes and services.


---

## Contact

I am always looking to tackle new challenges and expand my skill set. If you are interested in my work, feel free to reach out!

* **GitHub:** [https://github.com/xEska1337](https://github.com/xEska1337)
* **Discord:** xeska
* **Email:** [szymonkaczmarek030@gmail.com](mailto:szymonkaczmarek030@gmail.com)

---