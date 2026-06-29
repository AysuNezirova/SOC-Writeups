# TryHackMe Write-Up

## Custom Alert Rules in Wazuh

| Field | Value |
| --- | --- |
| **Platform** | TryHackMe |
| **Module** | Custom Alert Rules in Wazuh |
| **Completed by** | Aysu Nezirova |
| **Date** | June 29, 2026 |
| **Tasks completed** | 7 |

---

## Overview

This room covers how to create and fine-tune custom alert rules in Wazuh. Topics include understanding decoders, writing rule XML files, testing with the Ruleset Test tool, chaining rules using rule order, and reducing false positives via fine-tuning.

### Wazuh Rule Processing Pipeline

| Stage | Description |
| --- | --- |
| Pre-decoding | Extracts basic fields: timestamp, hostname, program_name |
| Decoding (Phase 2) | Maps raw log fields to structured Wazuh fields (e.g. sysmon.*) |
| Filtering/Rules (Phase 3) | Matches decoded fields against rule conditions, fires alerts |

---

## Task 1 — Introduction

No questions. Learning objectives reviewed and room overview studied. The room focuses on writing custom Wazuh rules to detect suspicious activity tailored to the environment.

---

## Task 2 — Decoders

Wazuh decoders extract structured fields from raw log data during Phase 2. The Ruleset Test tool in the Wazuh dashboard allows testing how a log sample is decoded and which rules fire against it.

A sample Sysmon log was submitted to the Ruleset Test page. Phase 1 extracted the timestamp, hostname, and program_name. Phase 2 (decoding) produced a full structured output with sysmon fields:

```
sysmon.image:       "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe"

sysmon.commandLine: "\"C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe\"
                     "-file" "C:\Users\Alberto\Desktop\test.ps1""

sysmon.currentDirectory: "C:\Users\Alberto\Desktop\"

sysmon.parentImage: "C:\Windows\explorer.exe"
```

The regex pattern `User: \S*` extracts a value by matching the literal string "User: " and then capturing all non-whitespace characters that follow.

**Q: Looking at the Sysmon Log, what will the value of sysmon.commandLine be?**
**A:** `"C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" "-file" "C:\Users\Alberto\Desktop\test.ps1"`

**Q: What would the extracted value be if the regex is set to `"User: \S*"`?**
**A:** `WIN-P57C9KN929H\Alberto`

---

## Task 3 — Rules

Wazuh rules are evaluated during Phase 3 (filtering). Each rule has an ID, a severity level (0–15), and a description. Rules can reference MITRE ATT&CK IDs and compliance frameworks.

Rule severity levels (from Wazuh documentation):

| Level | Classification | Description |
| --- | --- | --- |
| 12 | High importance event | Error/warning from system or kernel; may indicate an attack against a specific application |
| 13 | Unusual error (high importance) | Matches a common attack pattern most of the time |
| 14 | High importance security event | Triggered with correlation; indicates an attack |
| 15 | Severe attack | No chances of false positives. Immediate attention required |

Testing the Ruleset page with `sysmon.image` set to `"svchost.exe"` triggered rule 184666 (Sysmon - Suspicious Process - svchost.exe, level 12). Changing `sysmon.image` to `"taskhost.exe"` triggered a different rule:

```
Phase 3: Completed filtering (rules).

  id: '184736'
  level: '12'
  description: 'Sysmon - Suspicious Process - taskhost.exe'
  mitre.id: ['T1055']
  mitre.tactic: ['Defense Evasion', 'Privilege Escalation']
  mitre.technique: ['Process Injection']

** Alert to be generated.
```

**Q: From the Ruleset Test results, what is the `<mitre>` ID of rule id 184666?**
**A:** `T1055`

**Q: According to the Wazuh documentation, what is the description of the rule with a classification level of 12?**
**A:** High importance event

**Q: Change the value of sysmon.image to `'taskhost.exe'` and press Test. What is the ID of the rule that will get triggered?**
**A:** `184736`

---

## Task 4 — Rule Order

Wazuh evaluates rules in order. Child rules use `if_sid` to reference a parent rule — the parent must fire first for the child to be evaluated. This creates rule chains.

In the `sysmon_rules.xml` file, rule 184717 (level 0) is a child of the rule that fires for suspicious parent images. Inspecting the file:

```xml
<rule id="184717" level="0">
    <if_sid>184716</if_sid>
    <field name="sysmon.parentImage">smss.exe</field>
    <description>Sysmon - Legitimate Parent Image - wininit.exe</description>
</rule>
```

**Q: In the sysmon_rules.xml file, what is the Rule ID of the parent of 184717?**
**A:** `184716`

---

## Task 5 — Custom Rules

Custom rules are written in `/var/ossec/etc/rules/local_rules.xml`. The first custom rule (100002) detects file creation in suspicious directories by checking the `audit.cwd` field against a list of paths.

```xml
<group name="audit,">
  <rule id="100002" level="3">
    <if_sid>80790</if_sid>
    <field name="audit.cwd">downloads|tmp|temp</field>
    <description>Audit: $(audit.exe) created a file with filename
    $(audit.file.name) the folder $(audit.directory.name).</description>
    <group>audit_watch_write,</group>
  </rule>
</group>
```

After saving and restarting Wazuh, the Ruleset Test tool was used to verify. Submitting a log with `cwd=/var/log/audit/tmp_directory1` triggered rule 100002 (level 3):

```
Phase 3: Completed filtering (rules).

  id: '100002'
  level: '3'
  description: 'Audit: /bin/touch created a file with filename
  /var/log/audit/tmp_directory1/malware.py the folder /var/log/audit.'
  groups: '["audit","audit_watch_write"]'
  mail: 'false'
```

**Q: What is the regex field name used in the local_rules.xml?**
**A:** `audit.cwd`

**Q: Looking at the log, what is the current working directory (cwd) from where the command was executed?**
**A:** `/var/log/audit`

---

## Task 6 — Fine-Tuning

Fine-tuning reduces false positives and adds precision. The full custom rule chain in `local_rules.xml` is shown below:

```xml
<group name="audit,">

  <!-- Base rule: file created in suspicious directory -->
  <rule id="100002" level="3">
    <if_sid>80790</if_sid>
    <field name="audit.directory.name">downloads|tmp|temp</field>
    <description>Audit: $(audit.exe) created a file with filename
    $(audit.file.name) in the folder $(audit.directory.name).</description>
    <group>audit_watch_write,</group>
  </rule>

  <!-- Child: suspicious file extension (.py, .sh, .elf, .php) -->
  <rule id="100003" level="12">
    <if_sid>100002</if_sid>
    <field name="audit.file.name">.py|.sh|.elf|.php</field>
    <description>Audit: $(audit.exe) created a file with a suspicious
    file extension: $(audit.file.name).</description>
    <group>audit_watch_write,</group>
  </rule>

  <!-- Child: file creation failed -->
  <rule id="100004" level="6">
    <if_sid>100002</if_sid>
    <field name="audit.success">no</field>
    <description>Audit: $(audit.exe) created a file with filename
    $(audit.file.name) but failed.</description>
    <group>audit_watch_write,</group>
  </rule>

  <!-- Child: suspicious file name keywords -->
  <rule id="100005" level="12">
    <if_sid>100003</if_sid>
    <field name="audit.file.name">>malware|shell|dropper|linpeas</field>
    <description>Audit: $(audit.exe) created a file with suspicious
    file name: $(audit.file.name).</description>
    <group>audit_watch_write,</group>
  </rule>

  <!-- Exception: malware-checker.py is a known red team tool -->
  <rule id="100006" level="0">
    <if_sid>100005</if_sid>
    <field name="audit.file.name">malware-checker.py</field>
    <description>False positive. "malware-checker.py" is used by our
    red team for testing.</description>
    <group>audit_watch_write,</group>
  </rule>

</group>
```

**Rule chain logic:**

- **test.php** → matches `.php` extension → 100003 fires (level 12)
- **malware-checker.sh** → matches `.sh` extension → 100003 fires → name doesn't match 100005 keyword list → stays at 100003 → level 12

**Q: If the filename in the logs is `'test.php'`, what rule ID will be triggered?**
**A:** `100003`

**Q: If the filename in the logs is `'malware-checker.sh'`, what is the rule classification level in the generated alert?**
**A:** `12`

---

## Task 7 — Conclusion

Room completed successfully! Custom Wazuh rules were written, tested, chained, and fine-tuned to detect real threats while reducing false positives.

### Key Takeaways

- Wazuh rule processing has 3 phases: pre-decoding → decoding → filtering (rules)
- Rules use `if_sid` to chain — child rules only fire after their parent fires
- Rule level 0 acts as a suppressor/exception (no alert generated)
- Custom rules go in `/var/ossec/etc/rules/local_rules.xml`
- The Ruleset Test tool is the fastest way to validate rules without live traffic
- Fine-tuning with `level="0"` exception rules reduces false positives effectively
- MITRE ATT&CK IDs (e.g. T1055 - Process Injection) can be attached to rules for context
- `audit.cwd` and `audit.file.name` are key fields for detecting suspicious file write activity

### References

- Wazuh Documentation — https://documentation.wazuh.com
- Wazuh Rules Classification — https://documentation.wazuh.com/current/user-manual/ruleset/rules-classification.html
- TryHackMe — https://tryhackme.com
