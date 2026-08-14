# Automated OPNsense Configuration & Network Management

A secure, automated configuration backup and network state tracking system for a multi-VLAN enterprise/homelab network powered by OPNsense and Git.

---

## Project Overview

This repository documents the physical hardware setup, network architecture, and automated backup pipeline for my OPNsense firewall and core routing environment. 

To maintain high availability and rapid disaster recovery capabilities, configuration changes (including firewall rules, routing parameters, and Zenarmor policies) are tracked using version control. 

> **Note on Security:** To follow security best practices, raw `config.xml` files containing sensitive environmental data, encrypted credentials, network topologies, and VPN keys are maintained in a private, restricted-access repository. This public documentation outlines the workflow and architecture.

### Network Topology

```mermaid
graph TD
    WAN["ISP / Cox Modem"] -->|2.5GbE Link| FW["OPNsense Firewall (Protectli)"]

    subgraph "Core Security Engine"
        FW <--> ZEN["Zenarmor NGFW (Layer 7 Inspection)"]
        FW <--> AGH["AdGuard Home (Port 53 DNS Filtering)"]
        AGH --> UNB["Unbound DNS (DoT Port 853)"]
        UNB --> CF["Cloudflare DNS 1.1.1.1"]
    end

    FW -->|802.1Q VLAN Trunk| SW["Zyxel Managed Switch"]

    SW -->|Trunk - All VLANs| AP1["UniFi AP Upstairs"]
    SW -->|Trunk - All VLANs| AP2["UniFi AP Downstairs"]

    AP1 -.->|VLAN 10 SSIDs| W1["Trusted Wireless Clients"]
    AP2 -.->|VLAN 10 SSIDs| W1
    AP1 -.->|VLAN 20 SSIDs| W2["Gaming Devices (Wi-Fi)"]
    AP2 -.->|VLAN 20 SSIDs| W2
    AP1 -.->|VLAN 30 SSIDs| W3["Guest Wireless Clients - 20Mbps Limit"]
    AP2 -.->|VLAN 30 SSIDs| W3
   ``` 
---

## Hardware & Physical Layer Setup

* **Firewall Appliance:** Protectli Vault Mini PC (upgraded with 8GB RAM) running OPNsense
* **WAN Hand-off:** Cox Modem configured in Bridge Mode, utilizing 2.5GbE high-speed interfaces
* **Core Switching:** Zyxel Managed Switch handling VLAN tagging, trunking, and inter-VLAN routing controls
* **Wireless Access Points:** 2x Ubiquiti UniFi Access Points (Upstairs and Downstairs) powered via inline PoE injectors
* **Physical Infrastructure:** Custom-built Cat6 Ethernet runs (stripped, untwisted, standard RJ45 crimping, and ceiling-mounted)

---

## Technical Stack & Services

* **Network Segmentation (VLANs):**
  * **VLAN 10 - Nexus:** Primary family network for trusted internal devices
  * **VLAN 20 - Nexus Gaming:** Dedicated isolated wireless segment reserved specifically for gaming traffic to prevent congestion on the main network
  * **VLAN 30 - Nexus Guest:** Isolated guest environment, strictly rate-limited (capped at 20 Mbps) to prevent local malware propagation and bandwidth abuse
* **Network Security & DNS Services:**
  * **AdGuard Home:** Integrated directly into OPNsense for network-wide DNS ad-blocking, tracking protection, and custom query filtering
  * **Unbound DNS:** Configured as an upstream recursive resolver paired with Cloudflare DNS over TLS (DoT on port 853) for encrypted, private DNS queries
  * **Geo-Blocking:** GeoIP alias rules dropping inbound/outbound connections from known high-risk threat actor regions
  * **Zenarmor (NGFW):** Layer-7 deep packet inspection and web content filtering
  * **DHCP Server:** Automated IP pool assignment per isolated interface
* **Automated Version Control:** Scheduled background execution pushing state changes to a secure remote Git repository

---

## Automated Backup Architecture

The configuration tracking pipeline operates via the OPNsense internal API / Os-git-backup plugin / custom automated script:

[ OPNsense Firewall ] 
        │
        ├── 1. Admin/Policy Changes Made
        ├── 2. Nightly Cron Job / API Triggered
        ├── 3. Encrypted config.xml Parsed & Committed
        │
        ▼
[ Private Git Repository ] ---> Automated Version History & Audit Trail

### Automation Workflow
1. **Trigger:** Changes to firewall rules or Zenarmor policies trigger a configuration revision check.
2. **Sanitization (Optional):** Pre-commit hooks run scripts to strip dynamic session state data while preserving key configuration trees.
3. **Commit & Push:** State changes are automatically committed with a timestamped message and pushed to a secure off-site repository over SSH.

---

## Security Best Practices Implemented

* **Secrets Management:** Sensitive tokens, local password hashes, and WireGuard/OpenVPN private keys are excluded from public repositories.
* **Access Control:** SSH keys used for Git synchronization use restricted deployment permissions.
* **Least Privilege Routing:** Admin interfaces and API endpoints are strictly restricted to a secure, isolated Management VLAN; Web GUI access (ports 80/443) is blocked across untrusted subnets.

---

## Disaster Recovery & Restoration

In the event of hardware failure or catastrophic configuration loss:
1. Provision a fresh OPNsense baseline image on the Protectli hardware.
2. Pull the latest known-good snapshot from the backup repository.
3. Import `config.xml` via the OPNsense web console or CLI recovery prompt.
4. Reboot the system to fully restore interfaces, 2.5GbE links, VLAN tags, firewall rulesets, AdGuard Home configurations, and Zenarmor policies.
