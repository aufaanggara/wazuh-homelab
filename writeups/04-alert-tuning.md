[← Back to README](../README.md)

# Phase 6: Alert & Rule Tuning

**Goal:** Reduce alert noise from rule `5710` and produce a faster, more sensitive custom detection for repeated non-existent-user login attempts — the central objective of this project.

**Approach:**
Analysis of the generated logs showed that rule **5710** (level 5) fired on *every single* failed attempt — thousands of times — while Wazuh's own escalation logic only kicked in after a much larger threshold. This is a textbook case of **alert fatigue**: high-volume, low-severity noise that can bury genuinely actionable high-severity alerts and desensitize a SOC analyst reviewing the feed daily.

Custom rules in Wazuh live in `/var/ossec/etc/rules/local_rules.xml` rather than the default ruleset (`/var/ossec/ruleset/rules/`), so that upstream Wazuh updates never overwrite locally authored logic, and custom rules can be trivially removed to revert to stock behavior. Custom rule IDs must be **≥ 100000** to avoid colliding with Wazuh's reserved built-in rule ID space.

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

First iteration:

```xml
<rule id="100002" level="10" frequency="8" timeframe="120">
  <if_matched_sid>5710</if_matched_sid>
  <description>SSH brute force attempt: multiple non-existent user login failures detected.</description>
  <group>authentication_failures,pci_dss_11.4,</group>
</rule>
```

<img src="alerts/00-cek-local-rules-default.png" width="700">
<img src="alerts/01-custom-rule-100002-final.png" width="700">

Validated (without needing a full service restart) using Wazuh's built-in rule tester:

```bash
sudo /var/ossec/bin/wazuh-logtest
```

<img src="alerts/02-validasi-rule-logtest.png" width="700">

## Troubleshooting — Wazuh Manager not running

`wazuh-logtest` failed with `Wazuh-logtest error when connecting with wazuh-analysisd`. Investigation revealed the Manager had crashed after the VM was hard-powered-off in an earlier session, leaving `wazuh-db` and other core services down while `wazuh-authd`/`wazuh-apid` remained stuck in a partially-running state:

```bash
sudo systemctl status wazuh-manager     # Active: failed (Result: timeout)
sudo tail -n 50 /var/ossec/logs/ossec.log
# wazuh-authd: ERROR: Unable to connect to socket 'queue/db/wdb'
sudo /var/ossec/bin/wazuh-control status
# wazuh-db not running...
```

<img src="alerts/03-error-logtest-dan-status-awal.png" width="700">
<img src="alerts/04-cek-ossec-log-error.png" width="700">
<img src="alerts/05-cek-status-komponen-wazuh.png" width="700">
<img src="alerts/06-filter-log-wazuh-db.png" width="700">

The fix was a full, explicit stop → verify → start cycle (a plain `systemctl restart` alone was later found to be insufficient):

```bash
sudo /var/ossec/bin/wazuh-control stop
sudo /var/ossec/bin/wazuh-control status   # confirm everything "not running"
sudo /var/ossec/bin/wazuh-control start
sudo /var/ossec/bin/wazuh-control status   # confirm core services "is running"
```

<img src="alerts/07-stop-semua-service-wazuh.png" width="700">
<img src="alerts/08-verifikasi-semua-service-mati.png" width="700">
<img src="alerts/09-start-ulang-service-wazuh.png" width="700">
<img src="alerts/10-verifikasi-service-berhasil-running.png" width="700">

## Iterative rule testing

Feeding the same sample SSH log into `wazuh-logtest` repeatedly (simulating repeated attempts) revealed an important discovery — **rule 100002 never fired**. Instead, Wazuh's own **built-in rule 5712** (also frequency-based, threshold of 8) fired first and "consumed" the correlated event window before the custom rule could trigger:

<img src="alerts/11-test-logtest-1x-percobaan-rule-5710.png" width="700">
<img src="alerts/12-logtest-rule-bawaan-5712-menang-duluan.png" width="700">

This was an important tuning insight: a custom rule with an identical threshold to an existing built-in rule is redundant. To make the custom rule meaningfully **faster** than Wazuh's stock detection, the frequency was lowered from 8 to **5** — sensitive enough to beat the default, but still high enough to avoid flagging normal human typos:

```xml
<rule id="100002" level="10" frequency="5" timeframe="120">
  <if_matched_sid>5710</if_matched_sid>
  <description>SSH brute force attempt; multiple non-existent user login failures detected.</description>
  <group>authentication_failures,pci_dss_11.4,</group>
</rule>
```

<img src="alerts/13-revisi-frequency-jadi-5.png" width="700">

Re-tested — rule `100002` fired correctly on the 5th repeated attempt this time:

<img src="alerts/14-custom-rule-berhasil-trigger.png" width="700">

The Manager was fully restarted to load the change into production:

<img src="alerts/15-restart-wazuh-manager-berhasil.png" width="700">
<img src="alerts/18-full-stop-wazuh-manager.png" width="700">
<img src="alerts/19-full-start-wazuh-manager-berhasil.png" width="700">

## Live validation

Both attack methods (Hydra and the manual script) were re-run against `wazuh-server` and the results checked against a live Dashboard query:

```
rule.id: 100002
```

<img src="alerts/16-validasi-serangan-manual-setelah-tuning.png" width="700">
<img src="alerts/17-validasi-serangan-hydra-setelah-tuning.png" width="700">
<img src="alerts/20-validasi-final-serangan-manual.png" width="700">
<img src="alerts/21-validasi-final-serangan-hydra.png" width="700">
<img src="alerts/22-validasi-final-custom-rule-berhasil-di-dashboard.png" width="700">

**Result:** custom rule `100002` fired **6 times** in the Dashboard, all at level 10. The math checked out: of the 50 total login attempts across both attack methods, only 30 used a genuinely non-existent username (the rest used the valid `wazuh` username with a wrong password, which is handled by a different rule path) — `30 ÷ 5 (frequency) = 6` alerts, exactly matching the observed count.

**Tuning outcome:** the SIEM now escalates a non-existent-user SSH brute-force pattern to a high-severity alert after **5 failed attempts within 2 minutes**, instead of the stock **8-attempt** threshold — a **37.5% faster** detection time.

---

[← Previous: Attack Simulation](03-attack-simulation.md) · [Next: Active Response →](05-active-response.md)
