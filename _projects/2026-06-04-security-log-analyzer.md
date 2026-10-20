---
layout: post
title: "Security Log Analyzer"
date: 2026-06-04
category: project
summary: "A Python tool that reads auth logs and flags brute force and password spraying attempts, saving alerts as JSON for triage."
---

A Python tool that reads auth logs and flags suspicious login activity. It detects brute force and password spraying by looking at failed logins, the source IP behind them, and which accounts each IP is targeting. Alerts are saved as JSON so they're ready for triage.

## Objective

I wanted to build a tool that takes raw authentication logs and picks out attack patterns using simple detection rules. It focuses on two attacks: one IP failing again and again (brute force), and one IP failing against many different accounts (password spraying).

## Tech Stack / Tools Used

- **Language:** Python
- **Libraries:** `collections.defaultdict` to count failed logins per IP and track targeted accounts, `json` to save alerts
- **Input:** A text auth log (`s.log`)
- **Main parts:** Log parser, detection rules, and a main script that runs everything

## How It Works

1. **Read the log.** Each line is one auth event with a date, event type, username and source IP, like `2026-01-01 login failed user=admin ip=6.3.5.8`.
2. **Parse it.** Every line is turned into a dictionary with the date, the event, and fields like `user` and `ip`.
3. **Brute force detection.** Count failed logins per source IP. If an IP reaches 3 or more, raise an alert.
4. **Password spraying detection.** Track how many different usernames each IP failed against. If one IP hits 3 or more accounts, raise an alert.
5. **Collect alerts.** Both rules feed into one list of alerts.
6. **Save output.** Alerts are written to `alerts_triage.json`, so a SOC analyst can go through them or a SIEM can pick them up later.

The limit for each rule (3 failures, 3 accounts) sits at the top of the detection file, so making a rule stricter or looser means changing one number.

## What It Found in the Sample Log

Based on the detection code, this is what `s.log` triggers.

**Password spraying:** The IP `6.3.5.8` fails against four accounts: `admin`, `test`, `root` and `guest`. That's over the limit of 3 accounts, so it raises a password spraying alert.

```json
{
    "Alert Type": "Password Spraying Attack",
    "IP": "6.3.5.8",
    "User Accounts": ["admin", "test", "root", "guest"]
}
```

**Brute force:** This rule counts all failed logins per source IP, no matter which account was targeted. In `s.log` that flags `6.3.5.8` (4 failures), `6.3.5.7` (3), `1.6.7.2` (6) and `5.1.6.3` (3).

Because `6.3.5.8` spreads its failed logins across four accounts, it triggers both alerts. That's how the rule works right now. A tighter version would count failures per IP per account, so hammering one account and spraying many accounts get separated.

This only shows the rules catch suspicious patterns in my sample data. It doesn't prove a real attack happened.

## What I Built

- **Log parsing:** Raw log lines become structured records, so every rule works on the same fields (event, user, IP).
- **Separate detection rules:** Brute force and password spraying are their own functions with their own limits, and `detect()` joins them.
- **One alert file:** All alerts go into a single JSON file, ready for triage and easy to feed into a SIEM later.
- **Split into modules:** Parsing, detection and running the script are in separate files. Adding a new detection rule means writing one function and adding one line to `detect()`.

## Challenges Faced

**My test data was too narrow.** My first version was one script that counted failed logins per IP and called everything brute force. Every line in my test log used `user=admin`, so the tool only ever saw one kind of attack and I couldn't see what it was missing. Then I looked up what it's called when one attacker tries many accounts instead of one, and found password spraying. I wrote a new log with different usernames failing from the same source IP, and the old script had no way to describe it properly.

**Splitting one script into modules.** The single script was getting messy, so I split it into a parser, a detection file and a main script. Once logs were parsed into structured records, both rules could use the same data. Each rule being its own function made it easy to add password spraying without touching the parser or the main script.

##  Scripts

| File | What it does |
|---|---|
| `parser.py` | Parser: pulls the date, event and key=value fields out of each log line |
| `detector.py` | Detection rules: brute force and password spraying, joined by `detect()` |
| `analyzer.py` | Main script: reads the log, parses it, runs detection and writes `alerts_triage.json` |
| `sample.log` | Sample auth log for testing |

Together they make a simple flow from raw logs to alerts.

**Repository:** [Security Log Analyzer](https://github.com/shuvivyas/Security-Log-Analyzer)

## Results

The analyzer catches the brute force and password spraying patterns in the sample log and saves alerts with the source IP, the number of failed attempts and the targeted accounts. Building it gave me hands-on practice with log parsing, reading authentication events, writing detection rules, and keeping Python code in separate modules.

## Limitations and Next Steps

- The logs only have a date, no time of day, so I can't detect attacks in a time window. Three failed logins in seconds and three over a few days look the same to the tool.
- The logs don't include passwords, so password spraying is spotted only from many usernames failing from one IP.
- The brute force rule counts failures per IP, so one IP can trigger both alerts, like `6.3.5.8` does.
- Counting failed logins alone can also flag normal users who forgot their password, so better detection would need timestamps and a way to link failed logins to later successful ones.

