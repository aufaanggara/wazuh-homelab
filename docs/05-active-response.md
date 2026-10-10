[← Back to README](../README.md)

# Phase 7: Active Response (Automated Containment)

**Goal:** Automatically block an attacking IP address when the tuned custom rule (`100002`) fires, and safely reverse that block (both automatically and manually).

**Approach:**
Active Response commands execute on the **Agent side** (the victim host), not on the Manager — the Manager only decides *when* to trigger a response and pushes the command over the existing Manager↔Agent channel. For this reason, the attack target was switched from `wazuh-server` itself to `endpoint-linux-wazuh`, to avoid the Manager ever blocking access to itself and to better mirror a realistic scenario.

Before enabling blocking, a whitelist of hosts that must never be blocked was identified:
- `192.168.56.104` (`wazuh-server`, the Manager itself)
- `192.168.56.106` (`endpoint-linux-wazuh`, self)
- `192.168.56.1` (the Windows host's Host-only adapter IP, used for all remote SSH administration)

<img src="active-response/00-cek-ip-windows-host.png" width="700">

Wazuh ships a set of pre-registered Active Response commands (`firewall-drop`, `disable-account`, `host-deny`, `route-null`, etc.) — conceptually similar to pre-defined functions that only need to be "called" by name, rather than written from scratch. `firewall-drop` was selected, which manages `iptables` rules automatically.

```xml
<active-response>
  <disabled>no</disabled>
  <command>firewall-drop</command>
  <location>defined-agent</location>
  <agent_id>001</agent_id>
  <rules_id>100002</rules_id>
  <timeout>600</timeout>
</active-response>
```

`location: defined-agent` (targeting a specific agent ID) was chosen over `location: all` for this stage of the lab — with only one endpoint currently deployed, the practical effect is identical, but `defined-agent` gives tighter, more auditable control while the detection logic is still being validated.

<img src="active-response/01-tambah-config-active-response.png" width="700">

The Manager was restarted using the same full stop/start cycle established in Phase 6:

<img src="active-response/02-full-stop-wazuh-manager.png" width="700">
<img src="active-response/03-full-start-wazuh-manager-berhasil.png" width="700">

## Validation — automatic block

```bash
hydra -L sshbf.txt -P passwordssshbf.txt -t 16 192.168.56.106 ssh
```

The Dashboard confirmed the full detection chain firing in sequence: `5710` (noise) → `100002` (tuned correlation) → **`651` — "Host Blocked by firewall-drop Active Response."**

<img src="active-response/04-dashboard-active-response-berhasil.png" width="700">

A subsequent SSH attempt from Kali hung indefinitely with no error message — expected behavior for an `iptables DROP` rule (as opposed to `REJECT`), which silently discards packets rather than notifying the sender:

<img src="active-response/05-kali-terblokir-ssh-hang.png" width="700">

Confirmed directly on the endpoint's firewall table:

```bash
sudo iptables -L INPUT -n --line-numbers
```

```
num  target     prot opt source               destination
1    DROP       0    --  192.168.56.102       0.0.0.0/0
```

<img src="active-response/06-iptables-rule-block-verified.png" width="700">

## Validation — automatic unblock (timeout)

The block was confirmed present shortly after the attack, then re-checked several minutes later and found to have been automatically removed by Wazuh once the configured `600`-second timeout elapsed — with no manual intervention:

<img src="active-response/07-verifikasi-auto-unblock-timeout.png" width="700">

## Validation — manual unblock

A fresh attack was triggered to re-populate the block, and the rule was then removed manually before its timeout expired, to confirm an administrator can always intervene early if needed:

```bash
hydra -L sshbf.txt -P passwordssshbf.txt -t 16 192.168.56.106 ssh
sudo iptables -L INPUT -n --line-numbers   # confirm DROP rule present
sudo iptables -D INPUT 1                   # delete rule #1
sudo iptables -L INPUT -n --line-numbers   # confirm empty again
```

<img src="active-response/08-serangan-ulang-untuk-test-unblock.png" width="700">
<img src="active-response/09-unblock-manual-berhasil.png" width="700">

A final SSH attempt from Kali confirmed access was restored (host key verification prompt appeared, followed by a normal password prompt — no hang):

<img src="active-response/10-verifikasi-akses-normal-setelah-unblock.png" width="700">

**Result:** a fully closed-loop, automated blue-team pipeline was demonstrated end-to-end: detection → correlation → tuned escalation → automated containment → automatic and manual reversal — without any manual intervention required during the active-block phase.

---

[← Previous: Alert Tuning](04-alert-tuning.md) · [Back to README →](../README.md)
