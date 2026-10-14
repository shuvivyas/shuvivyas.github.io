---
layout: post
title: "Detecting New Local User Creation with Wazuh"
date: 2026-09-23
category: journal
summary: "Building a custom Wazuh rule to detect new local user creation on Windows (Event ID 4720)."
tags: [wazuh, siem, windows, detection-engineering, mitre-attack]
---

## 1. Definition
Local user account creation is a Windows activity in which a new user account is added to a system. Monitoring this activity helps identify if it is being misused by attackers or if accounts are being created without authorization. MITRE tracks this technique as T1136.001 (Create Account: Local Account), and Windows logs it as Security Event ID 4720.
In this blog, I'll explore how Wazuh can be used to monitor this activity and detect when a new local user account is created on a Windows machine.

## 2. Context / Why It Matters
Creating a local account is a common trick attackers use to get a persistence back door. Once the threat actors get access to a machine, they mostly add a new user so they still have access even if the stolen password gets reset or the entry point gets patched. They can also add that new user to the Admin group to have privileged access.
Windows logs this (Security Event ID 4720), but nobody checks raw logs all day. If there's no alert about account creation, a new account can go unnoticed for weeks. Wazuh fills that gap: it collects and analyses these events, and with a custom detection rule, we can configure it to generate an alert whenever a new local user account is created.

## 3. Problem / Scenario
For this lab I used VirtualBox on my main Windows machine. The Wazuh manager runs as a VM inside VirtualBox, and a Windows VM acts as the monitored endpoint with the Wazuh agent installed, reporting to the manager VM.
Wazuh already has a built-in rule for new user creation, but I wanted my own custom rule so I could control the alert level, the description and the MITRE tagging. So the problem was: how can I make Wazuh pick up Event ID 4720 from the Windows Security logs and raise a clear alert for it, and how can I prove it actually works?
To test it, I planned to create a dummy user on Windows, the same way an attacker would, and then check whether Wazuh caught it.

## 4. Approach / Steps Taken

- **Understanding the Windows event:** I started by finding which Windows Security event is tied to local user creation. Event ID 4720 is generated when a user account is created, so that was the one I wanted Wazuh to pick up.
- **Checking Wazuh's existing detection:** Wazuh already has a built-in rule for user account creation. I still wanted a custom one so I could set my own alert description, severity level and MITRE ATT&CK mapping instead of relying only on the default.
- **Creating the custom rule:** I wrote a custom rule for Event ID 4720 on the manager, so the alert clearly says a local user was created and includes the details needed for investigation.
- **Verifying the detection:** I made sure the agent on the Windows VM was sending Security logs, then created a test user and checked the Wazuh dashboard to see if the event was collected and my rule matched it.

## 5. Solution / Fix

The fix was to create a custom rule on the Wazuh manager to detect Windows Event ID 4720 and generate a clear alert whenever a new user account was created.
I added the following rule to `/var/ossec/etc/rules/local_rules.xml`:
```xml
<group name="windows,persistence,account_creation,">
  <rule id="100100" level="10">
    <if_group>windows</if_group>
    <field name="win.system.eventID">^4720$</field>
    <description>New local user created: $(win.eventdata.targetUserName) by $(win.eventdata.subjectUserName)</description>
    <mitre>
      <id>T1136.001</id>
    </mitre>
  </rule>
</group>
```

The rule matches Windows events with Event ID 4720 and generates a level 10 alert. The description includes the newly created username and the username of the account that initiated the creation. I also mapped the rule to MITRE ATT&CK technique T1136.001 (Create Account: Local Account) to provide additional context for the alert.
After saving the rule, I restarted the Wazuh manager to apply the changes:

```bash
systemctl restart wazuh-manager
```

To test the detection, I created a test user account on the Windows VM using the following command:

```
net user testuser P@ssw0rd123! /add
```

I then opened Threat Hunting → Events in the Wazuh dashboard and filtered the results using `rule.id: 100100`.
The alert appeared with my custom description, severity level 10, and MITRE ATT&CK mapping. This confirmed that the custom rule was working as expected and that Wazuh was detecting the account creation event.

![Wazuh alert for rule 100100](/assets/images/wazuh-alert.png)

## 6. Key Commands / Payloads / Tools

**Windows VM (elevated prompt)**

```
net user testuser P@ssw0rd123! /add
net localgroup administrators testuser /add
net user testuser /delete
Restart-Service -Name wazuh
```

**Wazuh manager**

```bash
systemctl restart wazuh-manager
/var/ossec/bin/wazuh-logtest
```

| Item | Value |
|---|---|
| Windows Event ID | 4720 (user account created) |
| Related events | 4722 (enabled), 4726 (deleted), 4732 (added to local group) |
| MITRE ATT&CK | T1136.001, Create Account: Local Account |
| Rules file | `/var/ossec/etc/rules/local_rules.xml` |
| Agent config | `C:\Program Files (x86)\ossec-agent\ossec.conf` |
| Custom rule ID | 100100 |
| Dashboard filter | `rule.id: 100100` |

**Tools:** Wazuh (manager, agent, dashboard), VirtualBox, Windows Event Viewer.

## 7. Lessons / Notes

- Custom rule IDs need to be 100000 or higher, otherwise they clash with Wazuh's built-in rules.
- Always restart the manager after editing `local_rules.xml`. A typo in the XML can stop it from starting, so check `/var/ossec/logs/ossec.log` if it fails.
- The Security event channel must be enabled in the agent's `ossec.conf`, otherwise Wazuh never sees 4720.
- If the agent can't reach the manager (VM network mode), no alerts show up and it looks like the rule is broken.
- Delete the test user afterwards.
- Next time: add a second rule for 4732 (user added to Administrators) and whitelist known admin accounts to cut false positives.
