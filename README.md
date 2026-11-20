# Cybersecurity Portfolio — Aldorino Muharram

## About Me

Hi! I'm Aldorino, a cybersecurity enthusiast with a focus on **offensive security and web application testing**. I'm currently completing a Cyber Security Bootcamp (Batch 08), where I practice the full penetration testing workflow against deliberately vulnerable lab applications.

**Current Role**: Cyber Security Bootcamp Student
**Focus**: Web application penetration testing, vulnerability assessment, and clear, reproducible technical reporting
**Approach**: I care less about collecting tools and more about documenting *why* each step was taken, what the business impact is, and how to fix it.

## Technical Skills

### Tools
- **Reconnaissance & Scanning**: Nmap, Nikto, Shodan, Dig
- **Web Proxies & Testing**: Burp Suite (Repeater, Intruder), OWASP ZAP
- **Exploitation**: SQLMap, manual payload crafting
- **Traffic Analysis**: Wireshark
- **Platform**: Kali Linux

### Methodologies & Frameworks
- **OWASP Top 10** — primary testing methodology
- **CVSS v3.1** — severity scoring and risk rating
- **PTES-style workflow** — scoping, recon, exploitation, post-exploitation, reporting
- **Responsible disclosure & anonymization** — sanitizing evidence before publishing

### Vulnerability Classes I've Tested
Injection (SQLi), Cross-Site Scripting (XSS), Broken Access Control (IDOR), Business Logic Flaws (price manipulation, payment bypass), Cryptographic Failures, Security Misconfiguration (exposed `.git`), and Missing Rate Limiting (brute force).

## Projects

### [Project 1: Black-Box Web Application Penetration Test — "DibiShop" E-Commerce Lab](./Offensive/case-study-1.md)
A full black-box engagement against a deliberately vulnerable e-commerce application. Starting from only an IP address, I mapped the application, identified and exploited **10 vulnerabilities** (3 Critical, 3 High, 3 Moderate, 1 Low), demonstrated business impact, and produced prioritized remediation guidance.

**Highlights:** unauthenticated SQL injection leading to admin takeover, plaintext credential storage, stored XSS, payment-verification bypass, exposed Git repository, and client-side price manipulation.

## Learning Journey

### What I've learned
- Running a structured engagement end-to-end instead of firing off isolated tool commands.
- Chaining low-signal findings (an exposed `.git`, a missing rate limit) into real impact.
- Writing findings a non-technical stakeholder can act on: severity, business impact, and a concrete fix.
- The discipline of sanitizing evidence — masking credentials, IPs, domains, and session tokens before anything leaves the lab.

### Challenges
- **Prioritization:** deciding which of 20+ scanner alerts were real and worth manual confirmation.
- **Reporting clarity:** translating a raw SQLMap dump into a one-line business risk.

## Career Objective

I aim to work as a **Penetration Tester / Offensive Security Analyst**, helping organizations find and fix vulnerabilities before attackers do. My near-term goal is to build a track record of well-documented engagements and earn hands-on certifications in web and network exploitation.

---

> **Ethics note:** Every technique shown here was performed against applications I was explicitly authorized to test (bootcamp lab environments). Target identifiers (domain, IP, company name, user data) are anonymized. Nothing in this repository is intended to be run against systems you do not own or have written permission to test.
