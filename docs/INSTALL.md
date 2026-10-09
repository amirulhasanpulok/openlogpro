# Installation guide

This guide installs Openlog on a new Ubuntu 24.04 server and takes it to a verified, secured, licensed system with its first router connected. It describes version 1.0.5.

**Contents**

1. [At a glance](#1-at-a-glance)
2. [Before you begin](#2-before-you-begin)
3. [Install](#3-install)
4. [First sign-in](#4-first-sign-in)
5. [Activate your licence](#5-activate-your-licence)
6. [Address and HTTPS](#6-address-and-https)
7. [Connect your first router](#7-connect-your-first-router)
8. [Verify the installation](#8-verify-the-installation)
9. [Run it in production](#9-run-it-in-production)
10. [Reference: what is installed and where](#10-reference-what-is-installed-and-where)
11. [Getting help](#11-getting-help)

## 1. At a glance

| | |
|---|---|
| Time | About 10 minutes for the install, plus the time to add your routers |
| Effort | One command on the server; everything else is in the web portal |
| Result | A portal at `http://<server address>/` that receives, matches and stores your routers' NAT and PPPoE logs |
| Safe to repeat | Yes. Running the installer again keeps your data, settings and secrets |

## 2. Before you begin

### 2.1 The server

| Item | Requirement | What the installer does if it is not met |
|---|---|---|
| Operating system | Ubuntu **24.04 LTS**, x86_64, systemd | Stops (blocker). Other systems are not supported |
| CPU | 2 cores or more | Warns below 2 cores |
| Memory | **4 GiB** or more | Stops below about 3.5 GiB |
| Disk | **4 GiB** free under `/var/lib` to install; **20 GiB or more** advised | Stops below 4 GiB, warns below 20 GiB |
| Role | A server of its own (a VM is fine). Do not share it with other services | The installer installs and configures PostgreSQL, ClickHouse, syslog-ng and Nginx |
| Address | A fixed IP address | The portal and the routers use it |
| Clock | NTP time synchronisation on (Ubuntu's default) | Warns when the clock is not synchronised |

**Sizing the evidence storage.** Logs are kept for the retention you choose (default **180 days**, 7 to 365,
**Log Settings**). Storage depends on how much your routers log, so measure instead of guessing: after the first week of
real traffic, read the storage figure on the Dashboard and multiply it out to your retention. A disk can be grown later
from **Server > Storage** without a reinstall. If the machine is a VM, take a **snapshot before you install**.

### 2.2 Network and firewall

Open these paths **before** you install.

| Direction | From / to | Port | Purpose |
|---|---|---|---|
| In | Your administrators | TCP 80, TCP 443 | The portal (443 once HTTPS is on) |
| In | Your routers | UDP **and** TCP 514 | Their logs. The log port can be changed later (**Server > Network**) |
| In | Your administrators | TCP 22 | SSH |
| Out | This server | TCP 443 to `api.github.com`, `github.com` and `*.githubusercontent.com` | Download the release (install, updates, the update check) |
| Out | This server | TCP 443 to `packages.clickhouse.com`; TCP 80 to `archive.ubuntu.com` and `security.ubuntu.com` | Packages. Also needed by later updates that bring new packages |
| Out | This server | TCP 443 to Let's Encrypt (`acme-v02.api.letsencrypt.org`) | Only if you turn on HTTPS |
| Out | This server | Each router's **API port** (TCP; 8728 is MikroTik's default) | The portal reads each router's identity and PPPoE sessions |

A proxy in the path is not configured by the installer; if downloads must go through one, set it up for `apt` and
`curl` on the server first.

### 2.3 What to have ready

- Your **licence** (a line starting with `OL1.`). It is optional at install time.
- If you want HTTPS now: a **DNS name** that already points at this server, and an e-mail address for the certificate
  notices. You can also do this later ([section 6](#6-address-and-https)).
- Whether the routers' logs will use port 514 or another port.
- The management address and API credentials of your first router ([Connect MikroTik routers](ROUTERS.md)).

### 2.4 What the installer changes on the machine

You should know this before running it on any machine that is not brand new.

- **Installs** PostgreSQL, ClickHouse (from its own package repository), syslog-ng, Nginx, certbot, logrotate, ufw
  (installed, **not switched on** unless you ask), and a few helpers.
- **Sets the system time zone to `Asia/Dhaka`.** The server keeps one time, in storage and on screen.
- **Creates** two system users (`logportal`, `openlog-collector`) with no shell, and the services listed in
  [section 10](#10-reference-what-is-installed-and-where).
- **Raises** the kernel's UDP receive buffer limit (`/etc/sysctl.d/90-openlog.conf`) so bursts of router logs are not
  dropped.
- **Writes** its secrets (database passwords, the key that protects saved router logins, the first administrator's
  password) to `/root/openlog.env`, readable by root only.
- **Does not** touch your firewall unless you turn it on, does not open any port by itself, and keeps the databases,
  the portal's internal ports and the metrics on the loopback interface only.

## 3. Install

### 3.1 Quick install

On the server, as root or with `sudo`:

```bash
curl -fsSL https://github.com/amirulhasanpulok/openlogpro/releases/latest/download/install.sh | sudo bash -s -- --licence 'OL1....'
```

The address always points at the **newest published release**. Leave `--licence` out to start in evaluation mode.

### 3.2 Review first install

For change-controlled environments, download the installer, read it, then run it:

```bash
curl -fsSLO https://github.com/amirulhasanpulok/openlogpro/releases/latest/download/install.sh
less install.sh
sudo bash install.sh --licence 'OL1....'
```

The release itself is checked by the installer against its published SHA-256 before anything is unpacked. To check it by
hand, download the two files from the [release page](https://github.com/amirulhasanpulok/openlogpro/releases/latest) and run:

```bash
sha256sum -c openlog-linux-amd64.tar.gz.sha256
```

### 3.3 Options

Put options after `install.sh` (or after `bash -s --` in the one-line form).

| Option | Meaning |
|---|---|
| `--licence 'OL1....'` | Install your licence now. Optional; it can be pasted later under **Server > Plan** |
| `--domain noc.example.com` | The address people will open. Default: the server's own IPv4 address |
| `--email you@example.com` | Turns on HTTPS with Let's Encrypt. `--domain` must then be a DNS name that points at this server, and ports 80 and 443 must be reachable from the internet |
| `--syslog-port 5140` | The port routers send logs to (default 514; otherwise 1024 to 65535). Changeable later in the portal |
| `--release-tag v1.0.0` | Install a specific older release instead of the newest |

Examples:

```bash
# HTTPS on your own name, licence installed
curl -fsSL https://github.com/amirulhasanpulok/openlogpro/releases/latest/download/install.sh | \
  sudo bash -s -- --licence 'OL1....' --domain noc.example.com --email noc@example.com

# Routers will send to port 5140
curl -fsSL https://github.com/amirulhasanpulok/openlogpro/releases/latest/download/install.sh | \
  sudo bash -s -- --syslog-port 5140
```

### 3.4 What happens

1. **Machine audit.** The platform, memory, disk, free ports (80, 443 and the log port), reachability of the download
   hosts, the clock and the SSH port are checked. Anything that would make the install fail is reported as a
   **blocker** and nothing is changed; warnings are reported and the install goes on.
2. **Download and verify.** The release is fetched from this repository and checked against its SHA-256.
3. **Install.** Packages, users, databases, services, the web server and the helper programs are installed; secrets are
   generated; the release is unpacked under `/opt/openlog/releases/<version>` and activated.
4. **Verify.** Every service and health endpoint is checked.

When it succeeds, the last lines look like this:

```
VERIFICATION OK
==============================================================
 Installation verified.
 Open:      http://203.0.113.10/
 Username:  admin
 Password:  sudo grep '^PORTAL_ADMIN_PASSWORD' '/root/openlog.env'
 Saved in /root/openlog.env (readable by root only). Change the password after you sign in.
==============================================================
```

If it stops with `AUDIT BLOCKER`, fix what it names and run the same command again; see
[Troubleshooting](TROUBLESHOOTING.md#the-installer-stops-with-audit-blocker).

## 4. First sign-in

1. Read the administrator password on the server:

   ```bash
   sudo grep '^PORTAL_ADMIN_' /root/openlog.env
   ```

2. Open the address the installer printed and sign in as `admin`.
3. **Change the password** now (**Users & Access**), and create a named account for each person instead of sharing
   `admin`. The roles are:

   | Role | Can use |
   |---|---|
   | Operator | Dashboard and Help |
   | Manager | The above, plus Log Stream, Search Log, Devices and Activity Log |
   | Administrator | Everything, including users, settings, the Server page and updates |

   Custom roles with exactly the menus you choose are available under **Users & Access**.
4. **Keep a copy of `/root/openlog.env` in a safe place off the server.** It holds the key that protects the routers'
   saved logins; without it they cannot be read back after a restore.

Sessions last 12 hours by default, and five failed sign-ins from one address lock that address out for ten minutes
(both adjustable under **Log Settings**).

## 5. Activate your licence

Until a licence is installed the server runs in **evaluation mode**: up to 3 devices, 3 users and 30 days of log
retention. Nothing stops working, and nothing is deleted, when a limit is reached; new devices or users are simply not
added.

1. Open **Server > Plan**. Copy the **Server ID** (for example `5F15-EF2A-3419-4633`).
2. Send it to your provider. A licence is issued for exactly that server.
3. Paste the licence into **Install or renew a licence** and press **Install licence**. No restart is needed, and the
   page shows the licence type, the customer, the limits and the dates.

A **subscription** licence covers the versions built while it is valid (plus 14 days of grace). A **perpetual** licence
never expires and includes new versions for the period stated on it (up to 5 years from issue). A version outside the
period is not offered; one that is installed anyway keeps working but cannot add devices or users. Nothing is ever
stopped or deleted because of a licence.

## 6. Address and HTTPS

By default the portal answers on the server's IP address over HTTP. For a name and a certificate:

**At install time:** `--domain noc.example.com --email noc@example.com` (see [3.3](#33-options)).

**Later, from the portal:** **Server > Domain**.

1. Create a DNS `A` record for the name that points at this server.
2. Set the **Support e-mail** under **Organization > Contact** (the certificate notices go there).
3. Make sure ports 80 and 443 reach the server from the internet.
4. On **Server > Domain** enter the name and press **Set up HTTPS**, and confirm with your password.

A certificate from Let's Encrypt is issued and renews by itself. The server stays reachable at its IP address over HTTP,
so a mistake cannot lock you out. **Remove domain** goes back to the plain setup.

## 7. Connect your first router

Follow [Connect MikroTik routers](ROUTERS.md). In short: create a read-only API account on the router, add the
router under **Devices**, paste the **Log setup commands** the portal shows into the router, and wait for the device to
show **Receiving logs**.

**Important:** logs are accepted only from routers that were added in the portal and whose name (the router's
`/system identity`) matches the one saved for the device. Everything else is dropped, whatever address it comes from.

## 8. Verify the installation

```bash
sudo /usr/local/lib/openlog/verify-log-server
```

It must end with `VERIFICATION OK`. Then check:

| Check | Where | Expected |
|---|---|---|
| Services | Dashboard, or `systemctl status log-portal openlog-syslog-worker router-reconciler syslog-ng nginx` | All running |
| Licence | Server > Plan | Your licence, or evaluation mode on purpose |
| Updates | Server > Updates | The installed version and the newest one on offer |
| A router | Devices | **Receiving logs** after you run the setup commands |
| Evidence | Search Log | Events of the router's subscribers appear |

## 9. Run it in production

**Firewall.** The hostname inside a UDP syslog packet is written by the sender, so anyone who can reach the log port can
claim a router's name. Allow the log port (514 by default) **only from your routers' addresses**, on your network
firewall. The portal ports (80 and 443) should only be reachable by the people who use the portal; use HTTPS for any
access that crosses an untrusted network. The server's own firewall (`ufw`) is installed but off by default; if you want
it, set `ENABLE_UFW=1` and `SYSLOG_CIDRS=<comma separated router networks>` in `/root/openlog.env`, review the SSH port,
and run:

```bash
sudo bash /opt/openlog/current/deployment/install-log-server.sh /root/openlog.env
```

**Backups.** A daily backup timer keeps seven copies of the settings (the PostgreSQL databases: devices, users, saved
logins, licence) with checksums under `/var/backups/openlog`. Run one before every change:

```bash
sudo systemctl start openlog-backup.service
```

Copy `/var/backups/openlog` and `/root/openlog.env` to another system. **The evidence itself (the log events) is not
in these backups**: protect it with VM or disk snapshots, or exports from Search Log, according to your retention duty.

**Monitoring.** Watch the free disk space (Dashboard and **Server > Storage**), the service health (Dashboard), and turn on
Telegram alerts under **Notifications** for devices that stop sending logs. Keep NTP time synchronisation on.

**Retention.** Set how long logs are kept under **Log Settings** (default 180 days). Cleanup runs nightly in the
server's time zone.

**Changes.** Make changes from the portal or by updating the release; do not edit files the installer manages, because
the next update puts them back.

**Updates and rollback.** See [Updates and rollback](UPGRADE.md).

**Decommissioning.** There is no automated uninstaller; the data lives only on this server. Export what you must
keep (Search Log export), then remove the machine. A reinstall on the same machine keeps the existing data.

## 10. Reference: what is installed and where

| Item | Detail |
|---|---|
| Releases | `/opt/openlog/releases/<version>`; `/opt/openlog/current` points at the running one. The last three are kept |
| Configuration | `/root/openlog.env` (secrets, root only), `/etc/log-portal.env`, `/etc/openlog/` (including `deployment.env`, `syslog.json`, `site.json`) |
| Helper commands | `/usr/local/lib/openlog/`: `verify-log-server`, `update-log-server`, `rollback-log-server`, `openlog-backup` |
| Backups | `/var/backups/openlog` (seven daily copies) |
| Raw log spool | `/var/spool/openlog-syslog/raw/` (rotated hourly by the worker's own rules) |
| Update and rollback logs | `/var/lib/openlog-sysops/update.log`, `/var/lib/openlog-sysops/rollback.log` |

| Service | What it does |
|---|---|
| `log-portal` | The web portal and its API (loopback port 9080, behind Nginx) |
| `openlog-syslog-worker` | Reads the raw log, parses NAT events, writes them to ClickHouse |
| `router-reconciler` | Reads each router's identity and PPPoE sessions through its API |
| `syslog-ng` | Receives the routers' logs on the log port and drops what is not from a verified router |
| `nginx` | Public web server (80 and 443) |
| `postgresql`, `clickhouse-server` | Settings and evidence stores (loopback only) |
| `openlog-allowlist-sync.timer` | Refreshes the list of accepted routers every few seconds |
| `openlog-backup.timer` | The daily backup |
| `openlog-sysops.path` | Lets the portal's Server page (domain, network, storage, updates) act on the machine through a small root helper, after your password |

Read a service's log with, for example, `sudo journalctl -u log-portal -n 100 --no-pager`.

## 11. Getting help

1. In the portal use **Report a problem** (bottom of the sidebar). It creates a report with a reference number and the
   server's technical details (versions, licence, service health, log collection, recent warnings), never passwords,
   keys, router logins or stored logs. Download it or e-mail it to your support contact.
2. If the portal itself does not open, send the output of `sudo /usr/local/lib/openlog/verify-log-server` and of
   `sudo journalctl -u log-portal -n 100 --no-pager`.
3. Start with [Troubleshooting](TROUBLESHOOTING.md): most installation problems are listed there with their fix.

Contact the provider who supplied your licence for support.
