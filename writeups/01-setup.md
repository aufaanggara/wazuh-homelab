[← Back to README](../README.md)

# Phase 1–2: Infrastructure Setup & Wazuh Installation

## Phase 1 — Infrastructure Setup

**Goal:** Stand up an isolated VirtualBox lab with a SIEM server that can both reach the internet and be reached by other lab VMs and the host.

**Approach:**
Before deciding on networking, we reasoned through VirtualBox's three relevant modes:

| Mode | VM↔VM | VM↔Host | VM↔Internet |
|---|---|---|---|
| NAT | ❌ (each VM isolated) | ⚠️ limited | ✅ |
| Internal Network | ✅ | ❌ | ❌ |
| Host-only Network | ✅ | ✅ | ❌ |

Since the lab needed internet access, inter-VM communication (agent→manager), **and** host→dashboard access, a **dual-adapter** setup (NAT + Host-only) was used instead of a single mode.

```bash
mkdir wazuh-siem-lab
cd wazuh-siem-lab
mkdir docs/setup docs/ingest docs/alerts endpoints/linux-agent endpoints/windows-agent notes
```

The Manager VM (`wazuh-server`) was created via VirtualBox's *Unattended Installation* (Ubuntu Server 24.04, 8 GB RAM / 4 vCPU / 50 GB dynamically-allocated disk — increased from an initial 4 GB/2 vCPU baseline after a resource-related install failure, see Phase 2 below).

<img src="setup/00-buat-struktur-folder.png" width="700">
<img src="setup/01-vm-name-and-os-selection.png" width="700">
<img src="setup/02-unattended-install-config.png" width="700">
<img src="setup/03-vm-hardware-allocation.png" width="700">
<img src="setup/04-vm-hard-disk-config.png" width="700">
<img src="setup/05-network-adapter1-nat.png" width="700">
<img src="setup/06-network-adapter2-hostonly.png" width="700">
<img src="setup/07-grub-boot-menu.png" width="700">

*(optional)*

<img src="setup/08-first-successful-login.png" width="700">

After first boot, the dual-adapter design was validated:

```bash
ip a
```

Two interfaces confirmed: `enp0s3` (NAT, `10.0.2.15`) and `enp0s8` (Host-only, `192.168.56.104`).

<img src="setup/09-cek-ip-address-vm.png" width="700">

> **Troubleshooting note:** an accidental **hard power-off** of the VM (instead of a graceful shutdown) later caused Wazuh's internal database service to fail on the next boot — full recovery process documented in [`04-alert-tuning.md`](04-alert-tuning.md).

For convenient remote administration (and to enable normal clipboard copy/paste, which does not work over the raw VirtualBox console on headless Ubuntu Server), OpenSSH Server was installed and enabled, allowing all subsequent work to be done from Windows Terminal:

```bash
sudo apt update && sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh
```

<img src="setup/17-ssh-server-wazuh-server.png" width="700">
<img src="setup/18-ssh-login-dari-windows.png" width="700">

**Result:** A working Ubuntu Server VM with verified dual-network connectivity and remote SSH access, ready for Wazuh installation.

---

## Phase 2 — Wazuh Installation (Manager + Indexer + Dashboard)

**Goal:** Install the full Wazuh all-in-one stack on `wazuh-server`.

**Approach:**
Wazuh's official all-in-one installer was downloaded and verified against the official domain (HTTPS + official `packages.wazuh.com` source). An early attempt to verify a `.sha512` checksum failed, because Wazuh does not actually publish a separate checksum file for the installer script — the official documentation relies on HTTPS transport security alone.

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

### Troubleshooting — Indexer startup timeout

The first install attempt failed:

```
[ 2984.465432] watchdog: BUG: soft lockup - CPU#0 stuck for 295s!
Job for wazuh-indexer.service failed because a timeout was exceeded.
```

<img src="setup/10-instalasi-wazuh-gagal-timeout.png" width="700">

Root cause: Wazuh Indexer (OpenSearch/Java) requires substantial CPU/RAM during its first JVM startup; the original 4 GB RAM / 2 vCPU allocation was insufficient in a virtualized environment. Resources were increased to **8 GB RAM / 4 vCPU**.

<img src="setup/11-troubleshoot-ram-8gb.png" width="700">
<img src="setup/12-troubleshoot-cpu-4core.png" width="700">

Re-running the installer succeeded:

```
25/07/2026 01:34:23 INFO: wazuh-indexer service started.
25/07/2026 01:41:53 INFO: wazuh-manager service started.
25/07/2026 01:42:44 INFO: wazuh-dashboard service started.
25/07/2026 01:47:44 INFO: Installation finished.
```

<img src="setup/13-instalasi-wazuh-berhasil.png" width="700">
<img src="setup/14-cek-ip-setelah-instalasi.png" width="700">

The dashboard was accessed over HTTPS from the Windows host via the Host-only IP (self-signed certificate accepted — expected, since Wazuh generates its own SSL cert):

```
https://192.168.56.104
```

<img src="setup/15-login-page-wazuh-dashboard.png" width="700">
<img src="setup/16-wazuh-dashboard-overview.png" width="700">

**Result:** A fully running Wazuh all-in-one instance, accessible from the host browser, already generating baseline self-monitoring alerts.

---

[← Back to README](../README.md) · [Next: Endpoint Deployment →](02-endpoint.md)
