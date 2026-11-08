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
- [x] sudo works for the admin user; root account is locked
- [x] Network configured (static IPv4), verified including reboot test
- [x] QEMU Guest Agent running; Proxmox shows the VM's IP
- [x] Snapshot `post-install` created and rollback tested
- [ ] SSH key login works; password login and root login are rejected
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

### 4.1 Base system check

After installation I verified the base system (2026-11-08): `hostnamectl` reports KVM, Debian 13 (trixie), machine type q35 and OVMF firmware. `sudo` works, root is locked (`sudo passwd -S root` shows `L`), `ssh.service` is enabled and running, and `fstrim.timer` is active.

### 4.2 Network configuration

The check showed that `ens18` had link but no IPv4 address and no route. The installer had written only an `inet6 auto` stanza to `/etc/network/interfaces`.

Final configuration:

```
# Primary interface: static IPv4 (outside the FRITZ!Box DHCP pool .20-.200), IPv6 via autoconfiguration
auto ens18
iface ens18 inet static
        address 192.168.178.11/24
        gateway 192.168.178.1

iface ens18 inet6 auto
```

Steps: checked that `.11` is free (ping returned "Destination Host Unreachable" from my own LAN address) and lies outside the DHCP pool, edited the file with `sudo nano`, verified how ifupdown parses it (`sudo ifquery -l`, `sudo ifquery ens18`), then applied it with `sudo sh -c 'ifdown ens18; ifup -v ens18'`.

| Address | Purpose |
|---------|---------|
| .1 | FRITZ!Box (gateway, DNS) |
| .11 | lamp01 (static) |
| .20-.200 | DHCP pool of the FRITZ!Box |

### 4.3 QEMU Guest Agent

`apt-cache policy qemu-guest-agent` showed the package was already installed (the Debian installer included it). The service runs. Connection to the host was verified from Proxmox (see section 5).

### 4.4 Snapshot

Snapshot `post-install` was taken without RAM on LVM-thin storage (`local-lvm`). With the guest agent running, Proxmox freezes the guest filesystem for snapshots without RAM ([Proxmox: fsfreeze](https://pve.proxmox.com/pve-docs/chapter-qm.html#qm_qga_fsfreeze)). The state is: network tested, agent running, before SSH hardening.

---

## 5. Verification

| Check | Command | Result |
|-------|---------|--------|
| IPv4 address | `ip -br a` | `ens18 UP 192.168.178.11/24` |
| Default route | `ip -4 route show default` | `default via 192.168.178.1 dev ens18 onlink` |
| Gateway reachable | `ping -c 3 192.168.178.1` | 0% packet loss |
| Internet and DNS over IPv4 | `ping -4 -c 3 deb.debian.org` | 0% packet loss |
| Persistence | reboot, then `ip -br a`, `ip -4 route show default`, ping gateway | same results as before the reboot |
| Package sources | `sudo apt update` | sources reachable, all packages up to date |
| Root locked | `sudo passwd -S root` | `L` |
| Agent to host | `qm agent 100 ping` on the Proxmox host | no output, exit code 0 |
| Agent to Proxmox UI | VM 100 -> Summary | IP `192.168.178.11` shown |
| Snapshot consistency | task log of the snapshot | freeze guest filesystem -> snapshot `scsi0` and `efidisk0` -> thaw -> `TASK OK` |
| Snapshot exists | `qm listsnapshot 100` | `post-install` listed |
| Rollback | created marker file, rolled back to `post-install`, started VM | marker file gone, network up immediately |

---

## 6. Error log

| Problem | Cause | Fix / lesson |
|---------|-------|--------------|
| `qm: command not found` | `qm` needs root and its PATH | Worked in the Proxmox shell as root. |
| No network after installation | `/etc/network/interfaces` had no IPv4 stanza; `networking.service` still reported success | Added `inet static` stanza. A running service does not prove the configuration does what is intended. |
| Typo `auto_ens18` (underscore) | Typing error while editing | Found by reviewing the file before saving. `ifquery -l` is the tool that confirms the keyword is parsed. |
| Backup created after the edit | Order of steps | The backup contained the new file, so it was renamed to `interfaces.v1-untested`. The original is documented by the earlier check. Rule: backup first, then editor. |
| `dns-nameservers` has no effect | `resolvconf` is not installed | DNS works through the static `/etc/resolv.conf` written by the installer. Parsed does not mean effective. |
| First DNS test ran over IPv6 | `ping deb.debian.org` prefers IPv6 | Repeated with `ping -4`. A test must exercise the path under test. |
| Assumed the guest agent was missing | Assumption without a check | `apt-cache policy` showed it was installed. Check before installing. |
| Installer message "No network mirror selected" | Unknown | `sources.list` contains the correct mirrors, so the message had no visible effect. Cause unresolved. |

---

## 7. Security notes

- The root account is locked, administration uses sudo. SSH hardening is not done yet (section 8).
- The static IP lies outside the DHCP pool to avoid address conflicts.
- After a manual `ifup`, the VM obtains a global IPv6 address through autoconfiguration. Before exposing any service (Phase 2), I must verify how inbound IPv6 traffic is handled by the router and decide on a firewall.
- A snapshot is a rollback point, not a backup: it lives on the same storage as the VM.
- Redaction: MAC addresses, global IPv6 addresses, interface alternative names (derived from the MAC) and the real username are replaced or omitted.

---

## 8. Open points

- [ ] SSH key login, then disable password and root login; test in a second session before applying
- [ ] Snapshot `base-install` after SSH hardening
- [ ] After a reboot, `ens18` has no IPv6 addresses (not even link-local); they appear after a manual `ifdown`/`ifup`. Cause unknown, same with `allow-hotplug` and `auto`
- [ ] Clarify the installer message "No network mirror selected"
- [ ] Screenshots: select 6-8, redact, add to the repo
- [ ] Switch man-page sources from bookworm to trixie
- [ ] Rewrite Phase 0 using the template

---

## 9. Sources

- [Proxmox: QEMU Guest Agent, fsfreeze](https://pve.proxmox.com/pve-docs/chapter-qm.html#qm_qga_fsfreeze)
- [Debian: interfaces(5)](https://manpages.debian.org/bookworm/ifupdown/interfaces.5.en.html)
- [Debian: ifup(8) / ifdown(8) / ifquery(8)](https://manpages.debian.org/bookworm/ifupdown/ifquery.8.en.html)
- [AVM: Fixed IP address and DHCP pool](https://fritz.com/apps/knowledge-base/FRITZ-Box-4060/549_Kein-Zugriff-auf-andere-Gerate-im-FRITZ-Heimnetz-moglich)