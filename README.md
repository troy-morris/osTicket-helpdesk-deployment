# osTicket Help Desk — Deployed on Windows Server & IIS (Azure)

A hands-on lab project: deploying the **osTicket** open-source support ticketing system on a **Windows Server** VM in **Microsoft Azure**, using **IIS**, **PHP (FastCGI)**, and **MySQL**.

This build was done manually (no automated stack installer) to demonstrate end-to-end setup and, more importantly, real-world troubleshooting of the IIS + PHP integration layer — the part that trips most people up.

---

## Stack

| Layer | Technology |
|-------|-----------|
| Cloud | Microsoft Azure (VM) |
| OS | Windows Server 2022 |
| Web Server | IIS 10 + URL Rewrite Module |
| Runtime | PHP 8.x (Non-Thread-Safe, via FastCGI) |
| Database | MySQL 8 |
| Application | osTicket (latest release) |

---

## Architecture

```
Browser  →  IIS (port 80)  →  URL Rewrite rules  →  FastCGI handler  →  php-cgi.exe  →  osTicket
                                                                                            │
                                                                                            ▼
                                                                                       MySQL (localhost)
```

A request to the helpdesk is received by IIS, routed by osTicket's URL Rewrite rules, handed to PHP through the FastCGI handler for execution, and backed by a MySQL database for ticket and user data.

---

## Build Steps

### 1. Provision the Azure VM
- Created a Resource Group (`rg-osticket-lab`) to keep all resources together for easy teardown.
- Deployed a Windows Server 2022 VM (D2ls size).
- Opened inbound ports **RDP (3389)** for management and **HTTP (80)** for web access.
- Connected via RDP.

### 2. Install IIS + CGI
- Added the **Web Server (IIS)** role via Server Manager.
- Under **Application Development**, enabled **CGI** (required for PHP via FastCGI).
- Verified IIS with the default welcome page at `http://localhost`.

### 3. Install URL Rewrite Module
- Downloaded and installed the **IIS URL Rewrite Module 2.1** from Microsoft.
- Required because osTicket's `web.config` defines `<rewrite>` rules; without the module, IIS cannot parse the config.

### 4. Install & configure PHP
- Downloaded **PHP 8.x (Non-Thread-Safe, x64)** and extracted to `C:\PHP`.
- Copied `php.ini-production` to `php.ini` and enabled the extensions osTicket requires:
  `mysqli`, `gd`, `imap`, `intl`, `mbstring`, `fileinfo`, `openssl`, `curl`.
- Set `extension_dir = "C:\PHP\ext"` and a `date.timezone`.
- Registered PHP with IIS as a **FastCGI** handler (`*.php` → `FastCgiModule` → `C:\PHP\php-cgi.exe`).
- Added a matching entry under **FastCGI Settings**.

### 5. Install MySQL & create the database
- Installed **MySQL 8 Community Server** and set a root password.
- Created the database and a dedicated application user:

```sql
CREATE DATABASE osticket;
CREATE USER 'ostadmin'@'localhost' IDENTIFIED BY 'YourStrongPassword!';
GRANT ALL PRIVILEGES ON osticket.* TO 'ostadmin'@'localhost';
FLUSH PRIVILEGES;
```

### 6. Deploy osTicket
- Downloaded the latest osTicket release and copied the `upload` contents into the IIS web root (`C:\inetpub\wwwroot`).
- Renamed `ost-sampleconfig.php` to `ost-config.php`.
- Ran the web installer at `http://localhost`, completed the System, Admin, and Database settings (hostname `localhost`).

### 7. Post-install hardening
- Deleted the `setup` folder.
- Set `include/ost-config.php` to read-only.
- Logged into the agent panel at `http://localhost/scp` and created a test ticket to confirm the full workflow.

---

## Troubleshooting Log

The value of this project was in the integration issues between IIS and PHP. Each is documented below as **symptom → cause → fix**.

### 1. HTTP 500.19 — config data invalid (`0x8007000d`)
- **Symptom:** Site returned 500.19 pointing at osTicket's `web.config`, which was valid XML.
- **Cause:** osTicket's `web.config` uses a `<rewrite>` section, but the **URL Rewrite Module was not installed**. IIS cannot parse a `<rewrite>` block it doesn't recognize.
- **Fix:** Installed the IIS URL Rewrite Module, then `iisreset`.

### 2. "Not recognized as a native module" when adding the PHP handler
- **Symptom:** Adding the `*.php` module mapping failed, stating FastCgiModule was not a recognized native module.
- **Cause:** The **CGI feature was not actually enabled**, so the FastCGI native module wasn't loaded.
- **Fix:** Enabled CGI via PowerShell, then `iisreset`:
  ```powershell
  Enable-WindowsOptionalFeature -Online -FeatureName IIS-CGI
  ```
  Confirmed **FastCgiModule** then appeared under server-level **Modules**.

### 3. PHP rendering as plain text
- **Symptom:** Browsing the site returned raw PHP source code instead of executing it.
- **Cause:** A `.php` **MIME type (`text/html`) had been added** to IIS. A MIME mapping tells IIS to serve the file as static content, which overrode the FastCGI handler meant to execute it.
- **Fix:** Removed the `.php` MIME type entirely. PHP files should have **no** MIME mapping — the FastCGI handler is what processes them.

### 4. HTTP 404.3 — no handler for `.php`
- **Symptom:** After removing the MIME type, `.php` requests returned 404.3 (no handler configured).
- **Cause:** The PHP handler existed at the **server level but was not applying to the site** (handler inheritance was disrupted by earlier config changes). Verified with:
  ```
  appcmd list config "Default Web Site" -section:system.webServer/handlers | findstr /i php
  ```
  which returned nothing.
- **Fix:** Added the handler **explicitly at the site level** so it no longer relied on inheritance:
  ```
  appcmd set config "Default Web Site" -section:system.webServer/handlers /+"[name='PHP_FastCGI_Site',path='*.php',verb='*',modules='FastCgiModule',scriptProcessor='C:\PHP\php-cgi.exe',resourceType='File',requireAccess='Script']"
  ```
  followed by `iisreset`. The handler then applied and PHP executed.

### 5. MySQL command appearing to "hang"
- **Symptom:** A SQL statement didn't return `Query OK`; the prompt changed to `->`.
- **Cause:** Missing terminating semicolon — the MySQL client waits for `;` before executing.
- **Fix:** Entered `;` to complete the statement (or re-entered the full statement on one line ending in `;`).

---

## Key Takeaways

- On IIS, PHP is executed by a **FastCGI handler**, not a MIME type — the two are mutually exclusive, and a stray `.php` MIME mapping will silently break execution.
- IIS handler mappings depend on **inheritance**; after heavy config changes, defining a handler directly at the site level removes ambiguity.
- The `appcmd` command-line tool is more reliable than the GUI for diagnosing *what configuration is actually applied* versus what appears set.
- "Recommended" PHP extensions (APCu, OPcache) are performance caches and are **not required** for a functional install.

---

## Cleanup

To avoid ongoing Azure charges, the entire resource group was deleted after documentation, removing the VM, disk, IP, and networking in a single action.

---

*Lab project for IT support / systems administration skill development.*
