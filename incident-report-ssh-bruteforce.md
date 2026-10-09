# Incident Report — SSH Brute Force Detection & Automated Containment

**Date:** July 2026
**Platform:** Self-built home lab (Wazuh SIEM, not a hosted CTF platform)
**Difficulty:** Intermediate
**Category:** Intrusion Attempt — SSH Brute Force
**Analyst:** Project owner (home lab)

---

## Executive Summary

Between **27–30 July 2026**, the monitored endpoint `endpoint-linux-wazuh` (`192.168.56.106`) was targeted by two independent SSH brute-force campaigns originating from host `192.168.56.102` (a lab-internal Kali Linux machine used to simulate an external attacker), plus an earlier set of test runs against the SIEM server itself (`wazuh-server`, `192.168.56.104`) during initial validation. A total of **50 login attempts per campaign** were made using a mix of valid and non-existent usernames against SSH (port 22). **No attempts succeeded** (0 valid credentials obtained). The Wazuh SIEM detected the activity, and a custom correlation rule (tuned specifically for this attack pattern) escalated the event to a high-severity alert **37.5% faster** than the SIEM's stock detection logic. An automated containment response (`firewall-drop`) blocked the attacking IP at the host firewall within seconds of the tuned alert firing, and the block was automatically lifted after a 10-minute cooldown. No lateral movement or data exposure occurred; this was a fully controlled validation exercise of the lab's own detection and response capability.

---

## Indicators of Compromise (IOC)

| Type | Value | Notes |
|---|---|---|
| Attacker IP | `192.168.56.102` | Kali Linux VM used as the simulated attacker; subject of automated block |
| Target IP (endpoint) | `192.168.56.106` | `endpoint-linux-wazuh`, final attack target used for tuning/response validation |
| Target IP (SIEM server) | `192.168.56.104` | `wazuh-server`; used as the initial attack target during Hydra/manual-script validation before the target was switched to the endpoint |
| Protocol / Port | SSH / TCP 22 | Attack vector |
| Tool | Hydra v9.6 | Parallel brute-force tool, `-t 16` threads |
| Tool | Custom `sshpass` bash loop | Sequential brute-force script, 10 attempts |
| Usernames attempted | `wazuh`, `root`, `admin`, `user` | 1 of 4 valid on target; 3 non-existent |
| Passwords attempted | 10-line subset of `rockyou.txt` + 10 common passwords (`123456`, `password`, `admin123`, `qwerty`, `letmein`, `wazuh123`, `toor`, `changeme`, `wazuh1`, `test123`) | No valid password found |
| Wazuh rule triggered (noise) | `5710` — *sshd: Attempt to login using a non-existent user* | Level 5, fired on every individual failed attempt (~1,984 events) |
| Wazuh rule triggered (built-in correlation, unused post-tuning) | `5712` — brute force, non-existent user (threshold: 8 attempts) | Superseded by custom rule after tuning |
| Wazuh rule triggered (parallel attack pattern) | `2502` — *syslog: User missed the password more than one time* | Level 10, fired during the Hydra (parallel) campaign |
| Wazuh rule triggered (sequential attack pattern) | `5763` / `5551` — *sshd: brute force...* / *PAM: Multiple failed logins in a small period of time* | Level 10, fired during the manual scripted (sequential) campaign |
| Custom detection rule | `100002` — *SSH brute force attempt; multiple non-existent user login failures detected* | Level 10, `frequency=5`, `timeframe=120s` — authored during this investigation |
| Active Response action | `651` — *Host Blocked by firewall-drop Active Response* | Level 3, automated `iptables DROP` rule against the attacker IP |

---

## Timeline of Events

| Timestamp (UTC) | Event |
|---|---|
| 2026-07-27 ~08:00–08:30 | First brute-force validation campaign executed against `wazuh-server` (`192.168.56.104`) via Hydra (40 attempts) and a manual `sshpass` loop (10 attempts), to generate test data for tuning. |
| 2026-07-27 08:29:16–08:29:28 | Custom rule `100002` (post-tuning) fires 6 times against `wazuh-server` dataset, confirming correlation logic works correctly against live traffic. |
| 2026-07-28 03:21 | Wazuh Manager fully restarted (`wazuh-control stop`/`start`) to load the finalized Active Response configuration. |
| 2026-07-30 07:37:36 | Target switched to `endpoint-linux-wazuh` (`192.168.56.106`). Rule `5710` and `5760` fire repeatedly during a fresh brute-force burst; rule `100002` fires at `07:37:36.139`. |
| 2026-07-30 07:37:36.507–540 | Active Response rule `651` (*Host Blocked by firewall-drop Active Response*) fires twice, confirming automated containment triggered immediately after the tuned alert. |
| 2026-07-30 ~20:37 | Attacker (`192.168.56.102`) attempts an SSH connection to `192.168.56.106` and the session hangs indefinitely — confirming the `iptables DROP` rule is active and silently discarding the attacker's packets. |
| 2026-07-30 (+10 min from block) | `iptables` rule for `192.168.56.102` confirmed automatically removed after the configured 600-second Active Response timeout elapsed, with no manual intervention. |
| 2026-07-30 ~20:57 | A second attack campaign re-triggers the block; this time the rule is removed **manually** (`iptables -D INPUT 1`) before timeout, to validate administrator override capability. |
| 2026-07-30 (immediately after manual unblock) | Attacker confirms restored SSH access to `192.168.56.106` (host key prompt + password prompt returned to normal, no hang). |

---

## Investigation Process

### Phase 1 — Baseline Detection Review
The investigation began by reviewing Wazuh's default behavior against unauthenticated SSH login attempts. Using the Dashboard's **Threat Hunting → Events** view filtered on `rule.groups: authentication_failed`, it was observed that rule `5710` (*sshd: Attempt to login using a non-existent user*, level 5) fired on **every single failed attempt** — nearly 2,000 events for the test campaigns run so far — while Wazuh's own built-in correlation rule `5712` only escalated after **8** matching events.

> Screenshot: `12-logtest-rule-baw....png`

This volume of low-severity noise was identified as a textbook case of **alert fatigue**: a SOC analyst reviewing this feed daily would be forced to scroll past thousands of level-5 entries to find the handful of genuinely escalated alerts, increasing the risk that a real attack is missed or dismissed.

### Phase 2 — Custom Detection Rule Authoring
A custom rule was authored in `/var/ossec/etc/rules/local_rules.xml` (ID `100002`, above Wazuh's reserved ID range of 100000) that correlates repeated `5710` events using `<if_matched_sid>` with `frequency=5` and `timeframe=120`. This was validated locally with `wazuh-logtest` before deployment, iterating through several rule-syntax errors (`<if_sid>` vs. `<if_matched_sid>`, missing `frequency`/`timeframe` attributes) before the rule fired correctly on the 5th simulated event.

> Screenshot: `01-custom-rule-100....png`
> Screenshot: `14-custom-rule-ber....png`

An important finding during this phase: the custom rule initially used `frequency=8` (matching Wazuh's built-in `5712`) and **never fired**, because `5712` consumed the correlated event window first. Lowering the threshold to `5` allowed the custom rule to win the race and fire *before* the stock detection — the actual mechanism by which "faster detection" was achieved.

```bash
sudo /var/ossec/bin/wazuh-logtest
```

```xml
<rule id="100002" level="10" frequency="5" timeframe="120">
  <if_matched_sid>5710</if_matched_sid>
  <description>SSH brute force attempt; multiple non-existent user login failures detected.</description>
  <group>authentication_failures,pci_dss_11.4,</group>
</rule>
```

### Phase 3 — Live Validation Against Simulated Traffic
Two independent attack methods were used to generate SSH authentication failures and validate the tuned rule under real conditions:

```bash
# Parallel brute force (Hydra)
hydra -L sshbf.txt -P passwordssshbf.txt -t 16 192.168.56.106 ssh

# Sequential brute force (manual bash + sshpass)
./ssh_bruteforce_manual.sh
```

> Screenshot: `05-eksekusi-hydra-s....png`
> Screenshot: `08-ssh-loop-manual....png`

Querying the Dashboard for `rule.id: 100002` confirmed **6 alerts** fired at level 10 — matching the expected math (30 of 50 total attempts used a genuinely non-existent username; `30 ÷ 5 = 6`).

> Screenshot: `22-validasi-final-cus....png`

### Phase 4 — Automated Containment (Active Response)
An Active Response binding was configured on `wazuh-server` so that any firing of rule `100002` would trigger the built-in `firewall-drop` command against the specific agent (`defined-agent`, agent ID `001` = `endpoint-linux-wazuh`):

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

> Screenshot: `01-tambah-config-a....png`

A trusted-host whitelist was established prior to enabling this response, to prevent the automated containment from ever blocking legitimate infrastructure:
- `192.168.56.104` (Wazuh Manager itself)
- `192.168.56.106` (the endpoint itself)
- `192.168.56.1` (Windows host's Host-only adapter, used for all remote administration)

Re-running the brute-force campaign confirmed the full detection → response chain fired correctly:

> Screenshot: `04-dashboard-active....png`
> Screenshot: `06-iptables-rule-blo....png`

```bash
sudo iptables -L INPUT -n --line-numbers
# 1    DROP    0    --  192.168.56.102    0.0.0.0/0
```

### Phase 5 — Response Reversal Validation
Both the automatic (timeout-based) and manual unblock paths were validated to confirm the containment does not become a permanent, unmanageable denial-of-service against the attacker IP (which, in a real environment, could later belong to a legitimate rotated/dynamic address):

> Screenshot: `07-verifikasi-auto-u....png` — automatic unblock confirmed after 600s
> Screenshot: `09-unblock-manual-....png` — manual unblock via `iptables -D INPUT 1`
> Screenshot: `10-verifikasi-akses-n....png` — SSH access confirmed restored post-unblock

---

## Root Cause Analysis

The endpoint was reachable over SSH on the lab's internal network with no rate-limiting, account lockout policy, or brute-force protection (e.g. `fail2ban`) configured at the OS level. This is not itself a misconfiguration specific to this lab — SSH does not rate-limit by default — but it meant that **detection and response had to be implemented entirely at the SIEM layer**. Additionally, Wazuh's **out-of-the-box correlation threshold** (`5712`, requiring 8 matching failures) was found to be slower than desired for this use case; the root enabler of the "noise" problem was that the default ruleset does not distinguish between "acceptable occasional failed login" and "sustained brute-force campaign" until a relatively high number of attempts have already occurred.

---

## Impact Assessment

- **Confidentiality:** No credentials were compromised. All 50+ attempts per campaign failed (`0 valid password found`).
- **Integrity:** No unauthorized configuration or file changes occurred on the target endpoint as a result of the attack itself (File Integrity Monitoring did not flag any unexpected changes attributable to the attacker).
- **Availability:** No denial-of-service impact to the endpoint. The automated containment response did briefly affect the *attacker's* own connectivity (by design), not the endpoint's availability to legitimate users.
- **Scope:** Single endpoint (`endpoint-linux-wazuh`) targeted; no evidence of lateral movement, as this was a controlled, isolated lab exercise with no other reachable hosts beyond the whitelisted lab infrastructure.

---

## Remediation & Recommendations

**Immediate actions (already implemented in this exercise):**
- Deployed custom correlation rule `100002` to escalate repeated non-existent-user SSH failures faster than the SIEM's default threshold.
- Enabled automated Active Response (`firewall-drop`) to block the attacking IP within seconds of detection.
- Configured a trusted-host whitelist to prevent the automated response from impacting legitimate infrastructure.

**Short-term:**
- Investigate and close the detection gap for the "valid username, wrong password" attack path — 20 of the 50 simulated attempts (using the valid `wazuh` username) were not conclusively isolated under a single rule/query during this investigation and should be re-examined.
- Apply the same tuned rule + Active Response binding to any additional endpoints added to the lab (currently only `endpoint-linux-wazuh` is covered under `defined-agent`).
- Evaluate switching Active Response `location` from `defined-agent` to `all` once multiple endpoints exist, to provide blanket coverage rather than requiring per-host configuration.

**Long-term:**
- Implement OS-level brute-force protection (e.g. `fail2ban`, SSH key-only authentication, disabling password auth entirely) as defense-in-depth alongside SIEM-level detection, rather than relying on detection/response alone.
- Formalize the custom detection logic developed here into a reusable **Playbook** (below) so that any future SSH brute-force alert is triaged consistently.
- Periodically re-run `apt upgrade` and vulnerability scans on monitored endpoints — the endpoint was found to be carrying 35 Critical / 606 High vulnerabilities at time of investigation, unrelated to this specific incident but relevant to overall attack surface.

---

## Lessons Learned

- Alert volume is not a proxy for detection quality — a rule firing on every individual event (`5710`) provided no actionable signal on its own; value only emerged once correlated against a time-bound frequency threshold.
- A custom correlation rule must be checked against existing built-in rules covering the same pattern; an identical threshold is functionally redundant. The "speed win" in this investigation came specifically from setting a *tighter* threshold (5) than the default (8), not from the mere existence of a custom rule.
- `<if_sid>` and `<if_matched_sid>` serve fundamentally different purposes (single-event match vs. accumulated frequency match) — using the wrong one silently prevents a correlation rule from ever firing.
- `systemctl restart` on `wazuh-manager` was found to be unreliable for reloading rule changes (reported services as "already running" without actually reloading `wazuh-analysisd`); a full `wazuh-control stop` → verify → `start` cycle was required to guarantee new rules were loaded into production.
- Attack *pattern* (parallel vs. sequential) influenced which built-in Wazuh rule fired (`2502` vs. `5763`/`5551`) even against the same underlying attack type — a reminder that detection logic should be validated against multiple attacker tooling styles, not just one.
- `iptables DROP` (used by Wazuh's `firewall-drop`) silently discards attacker traffic rather than rejecting it, denying the attacker feedback on whether their block succeeded — a deliberate and realistic blue-team design choice worth replicating in future response automation.

---

## Playbook: SSH Brute Force Alert (Rule 100002)

**Trigger:** Wazuh alert with `rule.id: 100002` (*SSH brute force attempt; multiple non-existent user login failures detected*), level 10.

**Triage Steps:**
1. Identify the source IP from the alert (`data.srcip` / `srcip` field) and the target agent (`agent.name`).
2. Confirm in **Threat Hunting → Events** whether the source IP also triggered correlated built-in rules (`2502`, `5763`, `5551`) — this indicates attack intensity/pattern (parallel vs. sequential).
3. Check whether the Active Response (`651` — *Host Blocked by firewall-drop Active Response*) fired automatically; if not, verify the `<active-response>` binding in `ossec.conf` is still enabled for the affected agent.
4. Cross-reference the source IP against the trusted-host whitelist (`192.168.56.104`, `192.168.56.106`, `192.168.56.1` in this lab) to rule out a false positive from legitimate infrastructure.
5. Review `rule.id: 5710` volume for the same source/target pair in the preceding minutes to confirm sustained activity rather than a single anomalous event.

**Escalation Criteria:**
- Any successful authentication (`sshd: authentication success`) from the same source IP following the brute-force burst — indicates a possible successful compromise and should be escalated immediately regardless of Active Response status.
- Source IP not resolvable to a known/expected host on the network.
- Active Response fails to trigger despite rule `100002` firing (indicates a broken containment path requiring manual IP blocking).

**Containment Actions:**
- Automated: Active Response `firewall-drop` blocks the source IP via `iptables DROP` for 600 seconds by default.
- Manual override (if early unblock or extended block is needed):
  ```bash
  sudo iptables -L INPUT -n --line-numbers   # identify the rule
  sudo iptables -D INPUT <rule_number>       # remove early
  # or, to make permanent until manually reviewed:
  sudo iptables -I INPUT -s <attacker_ip> -j DROP
  ```

**False Positive Indicators:**
- Source IP belongs to the whitelist (Manager, endpoint itself, or administrator's own host) — likely caused by a misconfigured script or a legitimate administrator mistyping credentials repeatedly.
- Low total volume (fewer than 5 failures in the `timeframe` window) — will not trigger rule `100002` by design, but should still be reviewed if combined with other suspicious indicators.
- Attempts using only the single valid system username (no non-existent usernames) — this pattern is *not* covered by rule `100002` (which keys off `5710`, the non-existent-user path) and should be triaged separately.

---

## Detection Rule

**Platform:** Wazuh (local custom ruleset, `local_rules.xml`)
**Purpose:** Detect sustained SSH brute-force attempts using non-existent usernames, faster than Wazuh's built-in correlation threshold.

**Rule/Query:**
```xml
<group name="local,syslog,sshd,">
  <rule id="100002" level="10" frequency="5" timeframe="120">
    <if_matched_sid>5710</if_matched_sid>
    <description>SSH brute force attempt; multiple non-existent user login failures detected.</description>
    <group>authentication_failures,pci_dss_11.4,</group>
  </rule>
</group>
```

**Logic Explanation:**
This rule does not parse raw log lines directly — it builds on top of Wazuh's existing decoder/rule pipeline. The built-in rule `5710` already matches the raw `sshd` log line pattern for "login attempt using a non-existent user" and fires at level 5 on every single occurrence. Rule `100002` uses `<if_matched_sid>5710</if_matched_sid>` (a **frequency-based** correlation condition, distinct from `<if_sid>` which only checks a single event in isolation) to track how many times rule `5710` has fired recently. Once it has fired **5 times within a 120-second window** (`frequency=5`, `timeframe=120`) from the same correlation context, rule `100002` fires itself at level 10 — a severity high enough to be treated as actionable, and (via the Active Response binding) to automatically trigger IP containment. The threshold was deliberately set *below* Wazuh's built-in `5712` rule (threshold 8) so that this custom rule reliably "wins" the correlation race and fires first, since Wazuh's first rule to consume a given event window suppresses redundant correlation from firing on the same events again.

**False Positive Notes:**
- A user who genuinely forgets their username and mistypes it 5+ times within 2 minutes (e.g. trying several old/former usernames) would trigger this rule despite not being a malicious actor. In a shared or high-turnover environment, this threshold may need to be raised slightly, or paired with a secondary check against known/expected usernames.
- The rule is scoped specifically to the **non-existent-user** path (`5710`); it does **not** catch brute-force attempts using a valid username with repeated wrong passwords — that traffic is handled by different built-in rules (observed as `5760`/`5503` in this investigation) and was not covered by a dedicated custom correlation rule in this iteration.
