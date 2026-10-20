---
layout: post
title: "My Log Analyzer Only Knew One Attack"
date: 2026-06-15
category: journal
summary: "My first log analyzer only caught brute force because my test data only had one username. Learning about password spraying made me rebuild it as separate modules."
tags: [python, security, log-analysis, brute-force, password-spraying, detection]
---
# My Log Analyzer Only Knew One Attack

## 1. What was the issue or challenge?

My first log analyzer only detected one type of attack: brute force. It counted failed logins per IP and raised an alert once an IP reached 3 failures. The problem was that all my test data used `user=admin`, so the tool never saw anything else. It worked on my data, but my data was too narrow to show me what the tool couldn't do.

## 2. How did I notice it?

I didn't see it from a bug or a crash. I noticed it when I searched Google for what it's called when the same attacker tries many different accounts instead of one. That's how I found **password spraying**. Once I understood how it differs from brute force, I realized my tool had no way to detect it.

## 3. What did I do about it?

I wrote a new log (`s.log`) with several usernames failing from the same IP, so the data looked like a real spraying attempt. Then I rewrote the project from one script into separate modules: a parser, a detector with each rule as its own function, and a runner. I added a password spraying rule that tracks how many different accounts each IP fails against, and moved the thresholds to the top of the file so they're easy to change. The output is structured JSON.

## 4. What went wrong?

One challenge was moving from my initial brute-force-only script to a more structured analyzer. I had to make sure the parser in `p.py` extracted the date, event, username, and IP address from each line in `s.log` so the detection functions could use those fields. I also had to keep the two detection methods separate: `brute()` counts failed logins by IP, while `passwords()` tracks distinct usernames targeted by each IP. Finally, I organized the workflow in `a.py` to read the log file, parse each line, pass the parsed data to `detect()`, and print the resulting alerts. This helped me build a modular version that could analyze both brute-force and password-spraying patterns.

## 5. What did I learn?

- You can't test for something you don't know exists. My tool wasn't buggy, my knowledge was narrow, and my test data reflected that.
- Brute force and password spraying look different in logs: one IP and one account versus one IP and many accounts.
- Separating parsing, detection and output makes it much easier to add new rules. Adding spraying took one new function instead of a rewrite.

## 6. What would I do differently or improve next?

- Add timestamps to the logs so I can detect attacks within a time window, since a date alone can't tell three failures in ten seconds from three over a week.
- Handle edge cases in the parser, like blank or malformed lines.
- Bring back severity levels, which my first version had.
- Keep learning new attack patterns on TryHackMe and Hack The Box and turn each into a detection rule.
