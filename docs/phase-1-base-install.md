# Phase 1 – Base Install

> **Status:** 🟡 In progress

> **Date:** 2026-11-06 – (Open end)

> **Previous phase:** [Phase 0 – Preparation](phase-0-preparation.md)

<!--
REDACTION RULE (public repo / GDPR):
- No real passwords, private keys, tokens or key contents
- No public IP, no MAC addresses, no serial numbers
- Mask third-party IPs or user data in logs/screenshots before committing
Delete this comment block before publishing if you like.
-->

---

## 1. Goal & Done-when

**Goal:** A minimal, hardened Debian 13 VM on Proxmox as the base for the LAMP stack.

- [x] VM created with the documented settings (`qm config` matches decisions)
- [x] Debian minimal installed (no desktop, SSH server + standard utilities only)
- [ ] sudo works for the admin user; root account is locked
- [ ] SSH key login works; password login and root login are rejected
- [ ] QEMU Guest Agent running; Proxmox shows the VM's IP
- [ ] Snapshot `base-install` exists

---

## 2. Starting point

Proxmox is up and running, the ISO has been [verified](phase-0-preparation.md), and the IP plan is in place.

---

## 3. Decisions

| Topic | Decision | Alternatives considered | Reasoning | Source |
|-------|----------|-------------------------|-----------|--------|
| CPU type | `host` | `x86-64-v2-AES` | Full CPU feature set for best performance; live migration not needed on a single node. | [Proxmox: CPU Type](https://pve.proxmox.com/pve-docs/chapter-qm.html#_cpu_type) |
| SCSI controller | VirtIO SCSI single + IO thread | VirtIO SCSI, LSI | One controller per disk enables a dedicated I/O thread; LSI is slow legacy emulation. | [Proxmox: Hard Disk](https://pve.proxmox.com/pve-docs/chapter-qm.html#qm_hard_disk) |
| Disk size | 32 GB | 40 GB, 60 GB, 100 GB | Sufficient for a lean LAMP stack; LVM-thin allocates space only when written. | [Proxmox: Trim/Disk](https://pve.proxmox.com/pve-docs/chapter-qm.html#qm_hard_disk:~:text=Trim/Discard) |
| Discard / SSD Emulation | Discard on & SSD emulation on | Discard off; SSD emulation off | Discard lets deleted guest data shrink the LVM-thin pool; SSD emulation presents the disk as non-rotational. | [Proxmox: SSD emulation](https://pve.proxmox.com/pve-docs/chapter-qm.html#qm_hard_disk:~:text=SSD%20emulation) |
| RAM | 4 GB, minimum = maximum | 2 GB, dynamic | Comfortable headroom for Debian, Apache, PHP-FPM and MariaDB; fixed allocation avoids memory pressure. | [Proxmox: Memory](https://pve.proxmox.com/pve-docs/chapter-qm.html#qm_memory) |
| QEMU Guest Agent | Enabled in Proxmox and installed VM | Agent disabled | Enables guest IP visibility, clean shutdown and filesystem freeze for more consistent backups | [Proxmox: QEMU Agent](https://pve.proxmox.com/pve-docs/chapter-qm.html#qm_system_settings:~:text=QEMU%20Agent) |
| Firmware / machine | OVMF (UEFI) + q35 | SeaBIOS + i440fx | Modern UEFI and PCIe-capable machine type; not required for LAMP, but future-proof. | [Proxmox: Machine Type](https://pve.proxmox.com/pve-docs/chapter-qm.html#qm_system_settings:~:text=Machine%20Type) |
| Static IP | Static IP configured directly in Debian; no FRITZ!Box DHCP reservation | DHCP reservation in FRITZ!Box | Static IP survives router changes; DHCP reservation is tied to the VM's MAC address and must be recreated after a router change. | [Debian: Network Configuration](https://www.debian.org/doc/manuals/debian-reference/ch05.de.html#5-3-moderne-netzwerkkonfiguration-ohne-grafische-oberflaeche) |
| CPU Cores | 2 vCPUs | 4vCPUs, 6 vCPUs | Enough parallelism for a small LAMP stack; leaves host capacity for Proxmox, backups and future VMs. | [Proxmox: CPU](https://pve.proxmox.com/pve-docs/chapter-qm.html#qm_cpu) |
| Firewall on NIC | Enabled | Disabled | Enables future per-VM rules and IP filtering; with the datacenter firewall off, no rules are applied yet. | [Proxmox: Firewall](https://pve.proxmox.com/pve-docs/chapter-pve-firewall.html#pve-firewall-vm-container-configuration) |
| Root password | Empty; root disabled; sudo for first user | Set root password | Avoids a directly attackable root login; administration happens through the regular user with sudo. | [Debian: Installation - Root-Passwort](https://www.debian.org/releases/stable/amd64/apas03.en.html) |
| Secure Boot | Enabled (pre-enrolled keys) | Disabled | Prevents untrusted code from running before the OS. | [Debian Wiki: SecureBoot](https://wiki.debian.org/SecureBoot) |

---

## 4. Implementation

*To be completed in the next commit.*

---

## 5. Verification

*To be completed in the next commit.*

---

## 6. Error log

*To be completed in the next commit.*

---

## 7. Security notes

*To be completed in the next commit.*

---

## 8. Open points

*To be completed in the next commit.*

---

## 9. Sources

*To be completed in the next commit.*