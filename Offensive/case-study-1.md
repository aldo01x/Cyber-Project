# Case Study 1 — Black-Box Web Application Penetration Test
### Target: "DibiShop" — Deliberately Vulnerable E-Commerce Lab

> **Authorization & scope:** This engagement was performed in an authorized bootcamp lab environment. All target identifiers (domain, IP address, company name, customer data, credentials, and session tokens) have been anonymized throughout. This document is for educational and portfolio purposes.

---

## 1. Introduction

| | |
|---|---|
| **Project title** | Black-Box Web Application Penetration Test — DibiShop |
| **Objective** | Identify and demonstrate security weaknesses in a web-based e-commerce application, assess their business impact, and recommend practical mitigations. |
| **Target (anonymized)** | `http://[REDACTED].duckdns.org/` — e-commerce web app |
| **IP (anonymized)** | `[REDACTED]` |
| **Roles tested** | Unauthenticated Guest, Registered User, Admin |
| **Methodology** | OWASP Top 10, CVSS v3.1 scoring |
| **Test type** | Black Box (started with IP only, no source or credentials) |
| **Tools used** | Nmap, Nikto, Shodan, Dig, Burp Suite, OWASP ZAP, SQLMap, Wireshark |

### Rules of Engagement (RoE)
- **Allowed:** active scanning (Nmap/Burp/ZAP), manual exploitation, proof-of-concept capture.
- **Not allowed:** Denial of Service, touching the underlying OS filesystem, altering checkout data/features, or defacing the site.

### Executive summary
Starting from a single IP address, I mapped the application and identified **10 vulnerabilities**: **3 Critical, 3 High, 3 Moderate, and 1 Low**. Six of them (Critical + High) are directly and easily exploitable by an unauthenticated attacker. The most serious issue is a chain in which SQL injection yields plaintext admin and customer credentials, leading to a full admin-panel takeover. Combined with client-side price manipulation and a payment-verification bypass, the application is exposed to direct financial loss and a customer-data breach.

---

## 2. Reconnaissance & Scanning

The goal of this phase was to understand the attack surface before touching any application logic.

### 2.1 OSINT & infrastructure
- **Shodan / Dig:** Confirmed the host is cloud-hosted and gathered public metadata (provider, region, last-seen date) without sending traffic to the target.
- **Nmap (`nmap -sC`):** Enumerated open ports. Four were open:
  - `22/tcp` SSH
  - `53/tcp` domain
  - `80/tcp` HTTP
  - `443/tcp` HTTPS
- **Nikto:** Baseline web-server fingerprinting (Apache) and quick checks for obvious misconfigurations.

### 2.2 Application mapping
Using Burp Suite's target sitemap and OWASP ZAP's spider, I built a map of reachable endpoints (`/login.php`, `/cart.php`, `/customer_register.php`, `/shop.php`, product pages, `/customer/confirm.php`, the account area, and an `/admin_area/`). ZAP's passive and active scan flagged a large cluster of alerts — most usefully **10 SQL Injection candidates**, an **exposed hidden file**, and several missing security headers — which gave me a prioritized list to confirm manually.

---

## 3. Exploitation & Findings

Severity follows CVSS v3.1. Each finding lists the risk, a sanitized proof of concept, the affected endpoint, and a concrete mitigation.

### Severity rating reference
| Severity | CVSS v3.1 | Meaning |
|---|---|---|
| Critical | 9.0–10.0 | Straightforward exploitation, system-level compromise. Patch immediately. |
| High | 7.0–8.9 | Harder, but can lead to privilege elevation or data loss. Patch ASAP. |
| Moderate | 4.0–6.9 | Exploitable only with extra steps. Patch after high-priority items. |
| Low | 0.1–3.9 | Not directly exploitable; reduces attack surface. Patch in next window. |

### Findings summary
| # | Finding | Severity |
|---|---|---|
| 3.1 | SQL Injection — Admin credential disclosure | **Critical (9.8)** |
| 3.2 | SQL Injection — Customer data exfiltration (plaintext storage) | **Critical (9.8)** |
| 3.3 | Exposed Git repository — credentials in source | **Critical (9.1)** |
| 3.4 | Stored XSS on account page | **High (7.6)** |
| 3.5 | Price manipulation (business logic) | **High (7.1)** |
| 3.6 | SQL Injection — payment-verification bypass | **High (7.1)** |
| 3.7 | Cryptographic failure — credentials over HTTP | **Moderate (6.8)** |
| 3.8 | IDOR on order-confirmation page | **Moderate (6.5)** |
| 3.9 | Missing rate limiting (brute force) | **Moderate (6.5)** |
| 3.10 | Unrestricted file type on upload | **Low (3.5)** |

---

### 3.1 SQL Injection — Admin Credential Disclosure
**Category:** Broken Access Control / Injection · **CVSS 3.1:** Critical (9.8) `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`

**Risk:** An unauthenticated attacker can extract admin credentials and take over the admin panel, enabling persistent access and session hijacking even after the injection point is closed.

**Proof of concept (sanitized):**
1. Active recon with OWASP ZAP flagged an injectable `code` parameter on the `/cart.php` endpoint.
2. Captured the request header for that endpoint with Burp and saved it to a request file.
3. Ran SQLMap against the saved request to enumerate databases, then dumped the relevant credential table.
4. Logged into the admin panel with the recovered credentials.

**Endpoint:** `/cart.php` — parameter `code`
**Tools:** OWASP ZAP, Burp Suite, SQLMap
**Mitigation:** Use parameterized queries / prepared statements for the `code` parameter; apply least-privilege on the database account so a single injection cannot read credential tables.

---

### 3.2 SQL Injection — Customer Data Exfiltration (Plaintext Storage)
**Category:** Injection / Cryptographic Failure · **CVSS 3.1:** Critical (9.8) `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`

**Risk:** The same injection path exposes customer and admin tables. Credentials are stored in **plaintext**, so an attacker obtains usable passwords directly, with no cracking required.

**Proof of concept (sanitized):** Reusing the injection point from 3.1, the credential tables were dumped. The dumped tables contained admin and customer credentials in plaintext — no cracking required.

**Endpoint:** `/cart.php` — parameter `code`
**Tools:** SQLMap
**Mitigation:** Parameterized queries to close the injection; store passwords with a strong, salted hash (e.g. bcrypt). Never store credentials in plaintext.

---

### 3.3 Exposed Git Repository
**Category:** Security Misconfiguration · **CVSS 3.1:** Critical (9.1) `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N`

**Risk:** A `.git` directory was publicly reachable. Its contents pointed to a repository whose source code contained hard-coded admin credentials.

**Proof of concept (sanitized):**
1. ZAP's "Hidden File Found" alert reported a reachable `/.git/config`.
2. Retrieving `/.git/config` revealed the remote repository reference.
3. The source contained admin credentials.

**Endpoint:** `/.git/config`
**Tools:** OWASP ZAP
**Mitigation:** Block web access to `.git/`, make the repository private, rotate any exposed credentials, and disable directory listing.

---

### 3.4 Stored XSS on the Account Page
**Category:** Injection (XSS) · **CVSS 3.1:** High (7.6) `AV:N/AC:L/PR:L/UI:R/S:C/C:H/I:N/A:L`

**Risk:** JavaScript injected into the "edit account" form is stored and executes when the page is viewed — including by an admin. From the admin view the payload fires in multiple fields, so an attacker could run script in the admin's browser to steal a session cookie.

**Proof of concept (sanitized):** A harmless `alert()` probe was placed in the account fields. It executed for a normal user (one field) and, more severely, across several fields in the admin view — demonstrating stored, cross-context execution without doing anything malicious.

**Endpoint:** `/my_account.php?edit_account` — parameters `c_name`, `c_contact`, `c_country`, `c_city`
**Tools:** Burp Suite, manual testing
**Mitigation:** Output-encode user data on render (e.g. `htmlspecialchars`); set the `HttpOnly` flag so cookies can't be read by script.

---

### 3.5 Price Manipulation (Business Logic Flaw)
**Category:** Business Logic / Broken Access Control · **CVSS 3.1:** High (7.1) `AV:N/AC:L/PR:L/UI:N/S:U/C:L/I:H/A:N`

**Risk:** The product price is taken from a client-submitted `product_price` parameter and trusted by the server. An attacker can set it to `0`, buying products for nothing and causing direct financial loss.

**Proof of concept (sanitized):**
1. ZAP active scan flagged the price parameter on product endpoints.
2. Using Burp Repeater, the `product_price` value in an add-to-cart request was changed to `0`.
3. The cart reflected a total of `0`.

**Endpoint:** product pages — parameter `product_price`
**Tools:** OWASP ZAP, Burp Suite
**Mitigation:** Never trust client-side pricing. Look up the price server-side from the database using only the product ID; validate all client input.

---

### 3.6 SQL Injection — Payment-Verification Bypass
**Category:** Injection / Business Logic · **CVSS 3.1:** High (7.1) `AV:N/AC:L/PR:L/UI:N/S:U/C:L/I:H/A:N`

**Risk:** The payment-confirmation logic is injectable. A common boolean payload makes the application treat an order as paid.

**Proof of concept (sanitized):** Submitting a classic always-true SQL condition in the confirmation fields caused the order status to flip to **Paid** without a real transaction.

**Endpoint:** `/customer/confirm.php?order_id=<n>` — parameter `code`
**Tools:** Burp Suite
**Mitigation:** Prepared statements with parameterized queries; filter special characters; verify payment status against the payment provider, not a client-supplied field.

---

### 3.7 Cryptographic Failure — Credentials Sent Over HTTP
**Category:** Cryptographic Failure · **CVSS 3.1:** Moderate (6.8) `AV:A/AC:L/PR:N/UI:R/S:U/C:H/I:L/A:L`

**Risk:** Login and registration are served over plain HTTP. An attacker on the same network can capture credentials in cleartext.

**Proof of concept (sanitized):** The browser flagged the site as "Not secure." A Wireshark capture of a simulated login showed the `POST` body — email and password — travelling in cleartext.

**Endpoints:** `/login.php`, `/customer_register.php`
**Tools:** OWASP ZAP, Wireshark
**Mitigation:** Enforce TLS (HTTPS) site-wide and add HSTS so browsers only connect over HTTPS.

---

### 3.8 IDOR on the Order-Confirmation Page
**Category:** Broken Access Control · **CVSS 3.1:** Moderate (6.5) `AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N`

**Risk:** `confirm.php` does not check that the requesting user owns the `order_id`. By changing the ID, a user can view other customers' orders, leaking personal data.

**Proof of concept (sanitized):** Burp Intruder iterated `order_id` values 1–50; `200` responses confirmed other users' orders were readable.

**Endpoint:** `/customer/confirm.php?order_id=<n>`
**Tools:** Burp Suite
**Mitigation:** Enforce server-side ownership checks tied to the authenticated session, not a client-supplied ID.

---

### 3.9 Missing Rate Limiting (Brute Force)
**Category:** Broken Access Control / Misconfiguration · **CVSS 3.1:** Moderate (6.5) `AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:L/A:N`

**Risk:** The login page accepts unlimited attempts — no delay, lockout, or CAPTCHA — so passwords can be brute-forced.

**Proof of concept (sanitized):** Repeated login requests via Burp Intruder all processed normally, confirming no rate limiting.

**Endpoint:** `/login.php`
**Tools:** Burp Suite
**Mitigation:** Add per-IP and per-account rate limiting/lockout and a CAPTCHA to slow automated guessing.

---

### 3.10 Unrestricted File Type on Upload
**Category:** Security Misconfiguration · **CVSS 3.1:** Low (3.5) `AV:N/AC:L/PR:L/UI:N/S:U/C:L/I:H/A:N`

**Risk:** The profile-picture upload does not validate file type on the client side (a `.pdf` was accepted and processed). In this instance the write was ultimately blocked by server file permissions, limiting impact — but the missing validation is still a weakness.

**Proof of concept (sanitized):** On the profile-picture upload, a `.pdf` file passed the client-side control and was submitted. The server then rejected the write due to file permissions, so no file was stored — but the type check should have happened first.

**Endpoint:** `/customer/my_account.php?edit_account` — parameter `c_image`
**Tools:** manual testing
**Mitigation:** Validate extension **and** MIME/magic bytes server-side; only accept `.jpg`/`.png`; store uploads outside the web root.

---

## 4. Post-Exploitation & Clean-Up

Per the RoE, no OS-level access or persistence was attempted; the engagement stayed within the application layer. After confirming each finding, I ran a **clean-up phase** to return the application to its pre-test state so results would not disrupt the environment:

- Removed test accounts created during registration testing.
- Removed test checkout/order history generated during payment and IDOR testing.
- Removed any rogue admin artifacts created while validating the admin takeover.

---

## 5. Reporting & Recommendations

### 5.1 Conclusion
The black-box test found **10 vulnerabilities (3 Critical, 3 High, 3 Moderate, 1 Low)**, each with proof of concept. The application's current security posture is **low**: an attacker with no account can extract admin and customer credentials and take over the admin panel. The combination of injection, plaintext storage, business-logic flaws, and missing transport encryption makes both a data breach and direct financial loss realistic.

### 5.2 Prioritized recommendations

**Critical & High — fix immediately**
- Convert all database access to parameterized queries / prepared statements.
- Rotate every exposed credential and store passwords with bcrypt (salted).
- Block web access to `.git/` and make the repository private.
- Derive product prices server-side from the database, never from client input.
- Verify payment status against the payment provider, not a client field.

**Moderate & Low — fix next**
- Enforce HTTPS site-wide with HSTS.
- Add server-side ownership checks on order IDs (session-based).
- Add login rate limiting / lockout and CAPTCHA.
- Validate upload file type by extension and magic bytes; store outside the web root.

**Non-technical**
- Re-test after remediation to confirm fixes.
- Prepare a data-breach incident-response procedure.

---

## References
- OWASP Top 10 — https://owasp.org/Top10/
- CVSS v3.1 Calculator — https://www.first.org/cvss/calculator/3.1
- PortSwigger Web Security Academy — https://portswigger.net/web-security/

> Target identifiers (domain, IP, repository URL, customer data, credentials) are anonymized throughout. Testing was authorized and confined to a lab environment.
