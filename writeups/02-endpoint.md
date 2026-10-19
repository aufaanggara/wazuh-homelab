[← Back to README](../README.md)

# Phase 3–4: Endpoint Deployment & Ingestion Verification

## Phase 3 — Endpoint Deployment & Log Forwarding

**Goal:** Deploy a second VM as a monitored endpoint and connect it to the Manager via the Wazuh Agent.

**Approach:**
A second VM (`endpoint-linux-wazuh`, 2 GB RAM / 2 vCPU / 20 GB) was built from a fresh Ubuntu Server ISO — rather than cloning `wazuh-server`, which would have carried over the full Manager/Indexer/Dashboard stack, unnecessarily heavy for a lightweight monitored endpoint — using identical dual-adapter networking so it would land on the same `192.168.56.0/24` subnet.

<img src="../endpoints/linux-agent/00-vm-name-and-os-selection.png" width="700">
<img src="../endpoints/linux-agent/01-unattended-install-config.png" width="700">
<img src="../endpoints/linux-agent/02-vm-hardware-allocation.png" width="700">
<img src="../endpoints/linux-agent/03-vm-hard-disk-config.png" width="700">
<img src="../endpoints/linux-agent/04-network-adapter1-nat.png" width="700">
<img src="../endpoints/linux-agent/05-network-adapter2-hostonly.png" width="700">
<img src="../endpoints/linux-agent/06-first-successful-login.png" width="700">
<img src="../endpoints/linux-agent/07-cek-ip-address-vm.png" width="700">

Connectivity between the two VMs (`192.168.56.106` ↔ `192.168.56.104`) was validated with `ping` before installing the agent:

<img src="../endpoints/linux-agent/08-ping-test-ke-wazuh-server.png" width="700">

> **Troubleshooting note:** after a first reboot, the VM briefly attempted PXE network boot instead of booting from disk (`PXE-E06: Option ROM requires DDIM support`). Resolved by confirming Hard Disk was top boot priority in VirtualBox → System → Motherboard.

<img src="../endpoints/linux-agent/09-fix-boot-order.png" width="700">

OpenSSH was enabled here as well:

<img src="../endpoints/linux-agent/10-ssh-server-setup.png" width="700">
<img src="../endpoints/linux-agent/11-ssh-login-dari-windows.png" width="700">

The Wazuh Agent repository and package were installed manually (unlike the Manager, the Agent does not ship with a repo pre-configured):

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" \
  | sudo tee -a /etc/apt/sources.list.d/wazuh.list

sudo apt update
```

<img src="../endpoints/linux-agent/12-setup-repository-wazuh.png" width="700">

```bash
WAZUH_MANAGER="192.168.56.104" sudo apt-get install wazuh-agent -y
```

<img src="../endpoints/linux-agent/13-instalasi-wazuh-agent-berhasil.png" width="700">

### Troubleshooting — environment variable dropped by `sudo`

The agent installed but failed to start:

```
ERROR: (4112): Invalid server address
ERROR: (1215): No client configured.
```

Root cause: `sudo` resets the calling shell's environment by default, so `WAZUH_MANAGER=...` placed **before** `sudo` never reached the privileged process (the fix would have been `sudo WAZUH_MANAGER="..." apt-get install ...`). Instead, the Manager address was set directly in the agent's config file:

<img src="../endpoints/linux-agent/14-error-agent-gagal-start.png" width="700">

```bash
sudo nano /var/ossec/etc/ossec.conf
# <address>192.168.56.104</address>
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent
```

<img src="../endpoints/linux-agent/15-edit-ossec-conf.png" width="700">
<img src="../endpoints/linux-agent/16-config-address-manager.png" width="700">
<img src="../endpoints/linux-agent/17-agent-berhasil-running.png" width="700">

**Result:** the agent registered successfully and appeared as **Active** in the Wazuh Dashboard.

<img src="../endpoints/linux-agent/18-agent-terdaftar-di-dashboard.png" width="700">
<img src="../endpoints/linux-agent/19-overview-agent-active.png" width="700">

---

## Phase 4 — Ingestion Verification

**Goal:** Confirm that logs from the new endpoint are actually being ingested and are searchable, not just that the agent shows "Active."

**Approach:**
The per-agent detail page (`Endpoints → endpoint-linux-wazuh`) was reviewed, showing system inventory, event-count evolution, MITRE ATT&CK mapping, vulnerability detection, and a Security Configuration Assessment (CIS Ubuntu 24.04 Benchmark) that ran automatically on agent registration.

<img src="../endpoints/linux-agent/20-detail-agent-dashboard.png" width="700">

The **Threat Hunting** module (Dashboard + Events tabs) confirmed raw, searchable log entries existed for the new agent — not just aggregate counters.

<img src="../endpoints/linux-agent/21-threat-hunting-dashboard.png" width="700">
<img src="../endpoints/linux-agent/22-threat-hunting-events-log.png" width="700">

**Result:** confirmed the full ingestion pipeline (Agent → Manager → Indexer → Dashboard) was functioning end-to-end for the new endpoint.

---

[← Previous: Infrastructure & Installation](01-setup.md) · [Next: Attack Simulation →](03-attack-simulation.md)
