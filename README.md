## 📁 Directory Structure – `katello-ol9-secure-install/`

```bash
# Root directory
katello-ol9-secure-install/
└── playbooks/
    ├── site.yml                       # Top-level playbook
    └── roles/
        └── katello/
            ├── tasks/
            │   ├── main.yml           # Task orchestrator
            │   ├── certs.yml          # Cert/keytool deployment logic
            │   ├── firewall.yml       # Ports 443, 80, etc.
            │   ├── db-check.yml       # Optional Candlepin DB drop
            │   └── installer.yml      # foreman-installer execution
            ├── files/
            │   └── keystore.p12       # Optional: import pre-built keystore
            ├── templates/
            │   └── openssl.cnf.j2     # Optional: OpenSSL config template
            └── handlers/
                └── main.yml           # Restart handlers







# katello-ol9-secure-install
- [Purpose](#-purpose)
- [Environment Overview](#-environment-overview)
- [Step 0–6: Preflight Prep](#-step-06-preflight-prep)
...

![Katello 3.11.5](https://img.shields.io/badge/Katello-3.11.5-blue)
![Oracle Linux 9.6](https://img.shields.io/badge/OL-9.6-orange)
![Last Verified](https://img.shields.io/badge/Verified-June_2025-brightgreen)



---

# 🚀 Deploying Katello 3.11.5 on Oracle Linux 9: A PLATO-Grounded End-to-End Build

> By Jefferson | Infrastructure System Engineer | [Bare Metal Applied Science]

## 🧠 Purpose

This write-up is for engineers who want more than step-by-step commands—they want to *understand* what’s happening under the hood. We’re walking through a clean Katello deployment on Oracle Linux 9.6 with custom certs, hardened design, and full PLATO-style documentation: **Procedure-Logged, Administratively Tracked Operations.**

No fluff. Just what works, what breaks, and why it matters.

---

## 🧰 Environment Overview

- **Node**: `foobar` 
- **OS**: Oracle Linux 9.6
- **Target**: Full Katello 3.11.5 stack, secure and GUI-ready
- **Cert Strategy**: Manual `keytool`-based keystore with RSA fixes (installer bug workaround)
- **Goal**: Reproducible Day 0 build with rerun logic and diagnostics

---

## 🔧 Step 0–6: Preflight Prep

```bash
sudo dnf install -y foreman-installer-katello
```

- Installed: Ruby gems, Kafo stack, Puppet SRPMs
- SSL keystore manually built via:
  ```bash
  keytool -genkeypair -alias tomcat -keyalg RSA ...
  ```
- Custom keystore + truststore dropped into `/etc/candlepin` and owned by `root`.

✅ Verified file paths, permissions, and cert validity prior to installer invocation.

---

## 🧱 Step 7: The Launch

Ran:

```bash
sudo foreman-installer --scenario katello
```

Live output (snapshot at time of run):

```
[configure] 250 configuration steps out of 1346 steps complete
...
[configure] 500 configuration steps out of 1348 steps complete
...
[configure] 1000 configuration steps out of 1376 steps complete
...
[configure] 1250 configuration steps out of 1376 steps complete
```

⏳ Candlepin schema load took longer than expected—watched closely for `cpdb` errors (none surfaced thanks to cert prebuild).

---

## 💣 Disaster Avoidance: Rerun and DB Reset Logic

Anticipated failure modes built into install flow:

- If installer is interrupted:
  - Drop databases:
    ```bash
    sudo -u postgres psql -c 'DROP DATABASE candlepin;'
    sudo -u postgres psql -c 'DROP DATABASE foreman;'
    ```
  - Resume install: `foreman-installer --scenario katello`

- Future-proofed: Logic to prompt for DB drops during future re-installs is being built into the YML flow.

---

## ✅ Step 8: Installation Complete!

Final output:

```
Success!
* Foreman is running at https://foobar
    Initial credentials are admin / acme_products
...
* Smart Proxy is running at https://node-0.home.localdomain:9090
```

✔️ GUI validated via `curl -k` from node  
✔️ HTML redirect seen: `/users/login`  
✔️ DNS mappings confirmed via `/etc/hosts` and remote tests

---

## 🔥 Postflight Firewall Validation

Noticed: `firewall-cmd --list-ports` returned nothing. Corrected with:

```bash
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=9090/tcp
sudo firewall-cmd --permanent --add-port=5647/tcp
sudo firewall-cmd --permanent --add-port=8140/tcp
sudo firewall-cmd --reload
```

💡 Lesson: Always validate open ports post-install, even if the services start cleanly.

---

## 🌐 DNS Resolution Hurdles (Windows Client Access)

Despite working `/etc/hosts` entries, Windows clients failed until:

- Manually invoked `curl.exe`:
  ```powershell
  curl.exe -k https://node-0.home.localdomain
  ```
- Disabled Secure DNS in Chrome/Edge
- Used `ping`, `nslookup`, and validated host mapping via `ipconfig /flushdns`

📎 Takeaway: Cross-platform DNS resolution is not automatic. Handle it early.

---

## 📓 Appendices (in PLATO format)

- **A**: Manual DB reset
- **B**: Cert generation + keystore commands
- **C**: `firewalld` ruleset & validation
- **D**: Hostname resolution (Windows/Linux)
- **E**: Optional Day 1 tasks (admin user config, repo sync bootstrap)

---
