# 🔐 Phishing Simulation & MFA Bypass — AiTM Case Study

> **To what extent do modern offensive tools enable MFA bypass, and how do psychological vectors amplify their effectiveness?**

Academic red team exercise conducted at [ESME Sudria](https://www.esme.fr/) (2025–2026) demonstrating how Adversary-in-the-Middle (AiTM) attacks defeat multi-factor authentication through session cookie interception.

![MITRE ATT&CK](https://img.shields.io/badge/MITRE%20ATT%26CK-T1557%20%7C%20T1539%20%7C%20T1566.001%20%7C%20T1078-red?style=flat-square)
![Platform](https://img.shields.io/badge/Platform-AWS%20EC2-orange?style=flat-square)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-blue?style=flat-square)

---

## 📋 Table of Contents

- [Overview](#overview)
- [Attack Architecture](#attack-architecture)
- [Attack Flow — How AiTM Bypasses MFA](#attack-flow--how-aitm-bypasses-mfa)
- [Social Engineering Design](#social-engineering-design)
- [Key Results](#key-results)
- [MITRE ATT\&CK Mapping](#mitre-attck-mapping)
- [Defensive Recommendations](#defensive-recommendations)
- [Legal \& Ethical Framework](#legal--ethical-framework)
- [Tech Stack](#tech-stack)
- [Team](#team)
- [References](#references)

---

## Overview

This project is a **supervised academic exercise** simulating a real-world phishing campaign targeting students via their university's Microsoft 365 SSO portal. The objective was to demonstrate that **MFA can be bypassed** using freely available open-source tools, and to quantify the role of **psychological manipulation** in attack success.

### Key Findings

| Metric | Value |
|--------|-------|
| Sessions intercepted | 36 |
| Full token bundles captured | 3 |
| MFA bypassed | ✅ Yes |
| Account access without credentials | ✅ Demonstrated |
| Time from click to full compromise | < 30 seconds |

> **Core insight:** MFA protects the authentication event — not the session. The cookie issued after MFA is all an attacker needs.

---

## Attack Architecture

```
┌─────────────────────┐     SSH Tunnel (port 3333)     ┌──────────────────────────┐
│   Attacker Machine  │◄──────────────────────────────►│     AWS EC2 Server       │
│   Kali Linux        │                                │  Ubuntu 26.04 LTS        │
│   VirtualBox        │                                │  X.X.X.X                 │
└─────────────────────┘                                │  eu-west-3 (Paris)       │
                                                       └──────────┬───────────────┘
                                                                  │
                                              ┌───────────────────┼───────────────────┐
                                              │                   │                   │
                                     ┌────────▼────────┐ ┌───────▼────────┐ ┌────────▼────────┐
                                     │   Mailgun SMTP  │ │    GoPhish     │ │   Evilginx2     │
                                     │ smtp.mailgun.org│ │   :3333        │ │   :443          │
                                     │   :587          │ │  (local only)  │ │  phishlet: o365 │
                                     └────────┬────────┘ └───────┬────────┘ └────────┬────────┘
                                              │                  │                   │
                                              │     Phishing     │    Proxied HTTPS  │
                                              │      email       │                   │
                                              ▼                  ▼                   ▼
                                     ┌─────────────────────────────────────────────────────────┐
                                     │                 moodle-esme.com                         │
                                     │          A record + wildcard → X.X.X.X                  │
                                     └─────────────────────────┬───────────────────────────────┘
                                                               │
                                                    Clicks link│
                                                    login.moodle-esme.com
                                                               │
                                                               ▼
                                                  ┌────────────────────────┐
                                                  │   Victim (Student)     │
                                                  │   Outlook / Firefox    │
                                                  └────────────────────────┘
```

### Infrastructure Components

| Component | Role | Detail |
|-----------|------|--------|
| **AWS EC2** | Cloud hosting | t3.micro, Ubuntu 26.04, eu-west-3 Paris |
| **GoPhish** | Campaign management | Email delivery, target tracking, real-time dashboard |
| **Evilginx2** | AiTM reverse proxy | Transparent MFA bypass, session cookie capture |
| **Mailgun** | SMTP relay | SPF/DKIM reputation, domain-authenticated sending |
| **SSH Tunnel** | Secure access | Local port-forwarding to GoPhish dashboard |
| **Let's Encrypt** | TLS certificates | Auto-provisioned, victim sees valid HTTPS padlock |

---

## Attack Flow — How AiTM Bypasses MFA

```
 ┌──────────┐    ┌──────────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐
 │  Victim  │    │   Clicks link    │    │  Evilginx2   │    │  Microsoft   │    │  Session cookie  │
 │ receives │───►│ login.moodle-    │───►│  proxies     │───►│authenticates │───►│  intercepted     │
 │  email   │    │  esme.com        │    │  request     │    │  (MFA incl.) │    │  (MFA bypassed)  │
 └──────────┘    └──────────────────┘    │  MitM        │    └──────────────┘    └────────┬─────────┘
                                         │  transparent │                                 │
                                         └──────────────┘                                 ▼
                                                                                ┌──────────────────┐
                                                                                │ Attacker reuses  │
                                                                                │ cookie → full    │
                                                                                │ account access   │
                                                                                │ No password,     │
                                                                                │ no MFA needed    │
                                                                                └──────────────────┘
```

### Step-by-step breakdown

1. **Email delivery** — GoPhish sends a crafted phishing email via Mailgun SMTP from `noreply@moodle-esme.com`
2. **Victim clicks** — The link points to `login.moodle-esme.com`, which resolves to the Evilginx2 server
3. **Transparent proxy** — Evilginx2 forwards the request to the real `login.microsoftonline.com` and relays the legitimate Microsoft login page back to the victim
4. **Authentication** — The victim enters credentials and completes MFA normally — everything is proxied in real time
5. **Cookie interception** — Microsoft issues session cookies; Evilginx2 captures them before forwarding to the victim
6. **Session hijack** — The attacker imports the stolen cookies into a clean browser via Cookie-Editor → full account access, no credentials required
7. **Redirect** — The victim is redirected to an innocuous page (Tenor GIF) and suspects nothing

### Why MFA fails here

Traditional MFA (SMS, authenticator apps, push notifications) verifies the **authentication event** but does not bind the resulting session to the authenticating device. The session cookie issued post-MFA is a **bearer token** — whoever possesses it gets access, regardless of how it was obtained.

---

## Social Engineering Design

The phishing email was designed using **Cialdini's 6 Principles of Influence** to maximize click-through rate:

| Principle | Implementation | Effect |
|-----------|---------------|--------|
| **Authority** | Email from "Head of Cybersecurity" | Establishes credibility |
| **Scarcity** | "Only 2 positions remaining" | Creates FOMO |
| **Urgency** | "Link deactivates once slots filled" | Pressures immediate action |
| **Relevance** | Thales SOC internship — perfect for target profile | Makes it personal |
| **Social Proof** | Official ESME branding and communication style | Implicit trust |
| **Fear** | Missing a unique career opportunity | Emotional override of caution |

**Result:** Recipients who noticed Outlook's "Not Verified" sender badge still clicked — because the scenario was too compelling to dismiss.

---

## Key Results

### Campaign Metrics

```
Emails Sent          ████████████████████  1
Emails Delivered     ████████████████████  1
Link Clicked         ████████████████████  1
Credentials Captured ████████████████████  1
Full Session Capture ████████████████████  1
```

### Evilginx2 Session Database

Over the testing period, Evilginx2 recorded **36 sessions** from multiple sources:

- **3 complete token bundles** with full Microsoft session cookies
- Credentials captured in plaintext (username + password)
- Session tokens valid for account takeover without re-authentication
- Multiple user-agents and IPs logged

### Account Takeover — Proof of Concept

The captured cookies were imported into a clean Firefox browser (private mode) using Cookie-Editor:

1. Navigate to `microsoftonline.com`
2. Delete all existing cookies
3. Import the JSON cookie bundle from Evilginx2
4. Refresh → **full account access, zero credentials entered, MFA completely bypassed**

> ⚠️ No emails were read, no settings were modified. Access was demonstrated and immediately terminated. All data was deleted.

---

## MITRE ATT&CK Mapping

| Technique ID | Name | Phase | How we used it |
|-------------|------|-------|----------------|
| [T1566.001](https://attack.mitre.org/techniques/T1566/001/) | Spearphishing Link | Initial Access | Crafted email with malicious link to login.moodle-esme.com |
| [T1557](https://attack.mitre.org/techniques/T1557/) | Adversary-in-the-Middle | Credential Access | Evilginx2 reverse proxy between victim and Microsoft |
| [T1539](https://attack.mitre.org/techniques/T1539/) | Steal Web Session Cookie | Credential Access | Session cookies intercepted post-MFA |
| [T1078](https://attack.mitre.org/techniques/T1078/) | Valid Accounts | Persistence | Captured cookies grant legitimate session access |

---

## Defensive Recommendations

### 🥇 1. FIDO2 / WebAuthn Hardware Keys

**The only defense that fundamentally resists AiTM attacks.**

FIDO2 keys bind authentication cryptographically to the **origin domain**. If the victim is on `login.moodle-esme.com` instead of `login.microsoftonline.com`, the key **refuses to activate**. The AiTM attack becomes impossible.

```
Real site:    login.microsoftonline.com  →  FIDO2 key responds    ✅
Proxy site:   login.moodle-esme.com      →  FIDO2 key refuses     🚫
```

### 🥈 2. Conditional Access Policies

Configure Azure AD / Entra ID Conditional Access to:
- Require **device compliance** before issuing session tokens
- Block sessions from **anomalous IPs or geolocations**
- Enforce **token binding** to the authenticating device
- Require **managed devices** for sensitive applications

### 🥉 3. Session Anomaly Detection

- Flag cookie reuse across **different IPs or geolocations**
- Detect **impossible travel** patterns (login from Paris, then from another country minutes later)
- Monitor for AiTM proxy signatures in authentication logs
- Implement **continuous access evaluation** (CAE) to revoke tokens in real time

### 4. User Awareness — URL, Not TLS

> 🔒 The padlock means **encrypted connection**, NOT **trustworthy site**.

Train users to verify the **full URL** in the address bar, not just the presence of HTTPS. Our attack used a valid Let's Encrypt certificate — the victim saw a legitimate padlock on a malicious domain.

---

## Legal & Ethical Framework

This project was conducted under strict legal and ethical constraints:

| Requirement | Compliance |
|------------|------------|
| **Art. 323-1 Code Pénal** | No unauthorized access — all infrastructure owned by the team |
| **GDPR Art. 5(1)(b)** | Purpose limitation — data used only for academic demonstration |
| **GDPR Art. 5(1)(e)** | Storage limitation — session database cleared after demonstration |
| **GDPR Art. 17** | Right to erasure — all captured data deleted on project completion |
| **NIS2 Directive** | Attack vector demonstrated is the same used against NIS2 entities |
| **Supervision** | Project supervised by Nathan Duverger at every stage |
| **Debrief** | All recipients were debriefed after the campaign |
| **Data deletion** | All credentials, cookies, and session data permanently deleted |

---

## Tech Stack

```
┌─────────────────────────────────────────────────┐
│  OFFENSIVE                                      │
│  GoPhish · Evilginx2 · Kali Linux · Mailgun     │
├─────────────────────────────────────────────────┤
│  INFRASTRUCTURE                                 │
│  AWS EC2 · Ubuntu 26.04 · SSH · Let's Encrypt   │
├─────────────────────────────────────────────────┤
│  DEFENSIVE ANALYSIS                             │
│  MITRE ATT&CK · FIDO2 · Conditional Access      │
├─────────────────────────────────────────────────┤
│  FRAMEWORKS                                     │
│  Cialdini · RGPD · NIS2 · Code Pénal Art. 323-1 │
└─────────────────────────────────────────────────┘
```

---

## Team

| Name | Role |
|------|------|
| **Mohammadi** | Infrastructure & Cloud |
| **Ndebe-Bongelo** | Attack Execution & Testing |
| **Hadj-Naceur** | Social Engineering & Analysis |
| **Nathan Duverger** | Supervisor |

---

## References

- [MITRE ATT&CK — Adversary-in-the-Middle (T1557)](https://attack.mitre.org/techniques/T1557/)
- [MITRE ATT&CK — Steal Web Session Cookie (T1539)](https://attack.mitre.org/techniques/T1539/)
- [Microsoft — Token theft playbook](https://learn.microsoft.com/en-us/security/operations/token-theft-playbook)
- [Microsoft — Defending against AiTM phishing attacks](https://www.microsoft.com/en-us/security/blog/2022/07/12/from-cookie-theft-to-bec-attackers-use-aitm-phishing-sites-as-entry-point-to-further-financial-fraud/)
- [Verizon DBIR 2023](https://www.verizon.com/business/resources/reports/dbir/)
- [COMCYBER-MI — État de la menace cyber 2026](https://www.interieur.gouv.fr/)
- [FIDO Alliance — How FIDO Works](https://fidoalliance.org/how-fido-works/)
- [Cialdini, R. — Influence: The Psychology of Persuasion (2006)](https://www.influenceatwork.com/)
- [Proofpoint — State of the Phish 2024](https://www.proofpoint.com/us/resources/threat-reports/state-of-phish)

---

## ⚠️ Disclaimer

This repository is published for **educational and defensive purposes only**. It documents a supervised academic exercise and does not contain any attack tools, exploit code, credentials, or operational configurations. The goal is to help organizations understand and defend against AiTM threats.

**Do not use the techniques described here for unauthorized access.** Unauthorized computer access is a criminal offense under French law (Art. 323-1 Code Pénal) and equivalent legislation worldwide.

---

<p align="center">
  <i>ESME Sudria — Cybersecurity Project 2025–2026</i><br>
  <i>"The best defense is a user who knows what a trustworthy URL looks like, and why the padlock is not enough."</i>
</p>
