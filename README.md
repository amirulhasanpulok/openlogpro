# Openlog

**Current version: 1.0.1** | built 2026-10-08T19:57:38Z

[Download this release](https://github.com/amirulhasanpulok/openlogpro/releases/tag/v1.0.1) | [All releases](https://github.com/amirulhasanpulok/openlogpro/releases) | [Changelog](CHANGELOG.md)

This repository holds **releases only**, no source code. To install a new server, download `install.sh` from the
release page and run it as described below. Servers that are already installed update from the portal
(Organization > Updates) or with `sudo /usr/local/lib/openlog/update-log-server`.

## What is new in 1.0.1

- Fix: a server could not be installed from a GitHub release channel. The machine check looked for a host called
  "github:owner" and stopped with "cannot reach"; it now checks api.github.com.
- Fix: Organization > Updates > Update now failed on a real server, because the root helper that runs the update
  could not start apt or sudo. It keeps the rights it needs now.
- Releases are built, tested and published by GitHub Actions. Every release is installed on a brand-new Ubuntu
  machine, used (sign-in, a router, logs, search, export, moving the log port) and updated from the previous release
  through the portal before it is offered.

---

NAT evidence collection and MikroTik PPPoE attribution: a web portal, PostgreSQL, ClickHouse, syslog-ng and
Nginx on one Ubuntu server. This release contains the finished programs only, no source code.

## Files in this release

| File | What it is |
|---|---|
| `install.sh` | the installer for a **new server** (it already points at the channel this release is on) |
| `openlog-linux-amd64.tar.gz` | the release itself: programs, service files and install scripts |
| `openlog-linux-amd64.tar.gz.sha256` | its SHA-256 (the installer checks it; to check by hand: `sha256sum -c openlog-linux-amd64.tar.gz.sha256`) |
| `RELEASE.json` | version, build time, commit and checksum (the portal reads it to tell what is new) |
| `README.md` | this file |

## What you need

- A server with **Ubuntu 24.04 (x86_64)**, root or `sudo`, internet access, 4 GB RAM and 4 GB free disk
  (log storage needs more, depending on how much traffic your routers log).
- Free ports **80** (and **443** for HTTPS) and the **log port** (UDP and TCP, **514** unless you choose another).
- Your **licence** (a line starting with `OL1.`). Without one the server runs in evaluation mode: up to 3 devices
  and 3 users.
- Only if the channel is private: a read-only GitHub token for it. The installer asks for it and does not keep it,
  unless you pass `OPENLOG_RELEASE_TOKEN` (needed for the portal's Updates tab to see a private channel).

## Install a new server

Download `install.sh` from this release to the server, then:

```bash
sudo bash install.sh --licence 'OL1....'
```

Options (put them after `install.sh`):

| Option | Meaning |
|---|---|
| `--licence 'OL1....'` | install your licence now (you can also paste it later under Organization > Plan) |
| `--domain noc.example.com` | the address people open (default: this server's IPv4 address) |
| `--email you@example.com` | turn on HTTPS with Let's Encrypt (needs `--domain` to be a DNS name that points here, ports 80 and 443 open) |
| `--syslog-port 5140` | the port routers send logs to (default 514; 1024 to 65535 otherwise; changeable later in the portal) |

The installer checks the machine, installs PostgreSQL, ClickHouse, syslog-ng and Nginx, unpacks this release under
`/opt/openlog/releases/1.0.1`, creates fresh passwords and keys in `/root/openlog.env` (readable by root only;
**keep a copy**: without its encryption key the routers' saved logins cannot be read back), starts everything,
verifies it, and prints the address and the first login. Run it again at any time: existing data and settings are kept.

The first administrator's password is in `/root/openlog.env`:

```bash
sudo grep '^PORTAL_ADMIN_' /root/openlog.env
```

Change it after you sign in (Users & Access).

## After installing

1. **Licence.** Organization > Plan shows "This server ID". Send it to your provider to get a licence tied to this
   server, then paste the licence there. No restart is needed.
2. **Routers.** Devices > Add device. Each device has **Log setup commands** to paste into the MikroTik; they name the
   address and log port of this server. Allow the log port on any firewall in front of the server.
3. **Check.** `sudo /usr/local/lib/openlog/verify-log-server` prints `VERIFICATION OK` when every service is healthy.

## Updating

- **From the portal:** Organization > Updates shows the version you run, the newest version on the channel, what is
  new, and whether your licence covers it. **Update now** (asks for your password) installs it.
- **From the command line:** `sudo /usr/local/lib/openlog/update-log-server`

An update downloads the new release, checks its checksum and your licence, takes a backup of the settings, installs it,
restarts the portal for a minute or two (log collection carries on), and goes back to the previous release by itself
if the new one does not start. The last three releases are kept in `/opt/openlog/releases`.

A licence covers the versions **built** while it is in force: a subscription while it lasts (plus 14 days), a perpetual
licence for up to 5 years from the day it was issued. A newer version than that is not offered, and if one is installed
anyway it keeps working but cannot add devices or users. Nothing is ever stopped or deleted because of a licence.

To go back to an earlier release by hand (replace `1.0.0` with the folder name under `/opt/openlog/releases`):

```bash
sudo ln -sfn releases/1.0.0 /opt/openlog/current
sudo bash /opt/openlog/current/deployment/install-log-server.sh /root/openlog.env
```

## Changing the log port

Organization > Network > Log receiving port. The old port stays open next to the new one until you press
**Stop listening**, so routers that still use it are not cut off while you update them.

## Good to know

- Backups of the settings (devices, users, saved logins) are made daily and before each update under `/var/backups`;
  log evidence itself is not copied by them.
- Everything the installer sets up (database passwords, the encryption key) is in `/root/openlog.env`.
  Treat the file like a password.
- Questions or problems: contact the provider who gave you this software, and send the output of
  `sudo /usr/local/lib/openlog/verify-log-server`.
