# Troubleshooting

Find the symptom, read the cause, apply the fix. If it is not here, go to [What to send to support](#what-to-send-to-support).
This guide describes version 1.0.5.

**Contents**

- [The installer stops with AUDIT BLOCKER](#the-installer-stops-with-audit-blocker)
- [The installer fails or stops half way](#the-installer-fails-or-stops-half-way)
- [The portal does not open](#the-portal-does-not-open)
- [I cannot sign in](#i-cannot-sign-in)
- [The licence is refused or the Plan page shows a warning](#the-licence-is-refused-or-the-plan-page-shows-a-warning)
- [A device shows unreachable](#a-device-shows-unreachable)
- [A device shows no logs](#a-device-shows-no-logs)
- [Logs arrive but the dashboard is behind](#logs-arrive-but-the-dashboard-is-behind)
- [HTTPS could not be set up](#https-could-not-be-set-up)
- [An update failed or was undone](#an-update-failed-or-was-undone)
- [The disk is filling up](#the-disk-is-filling-up)
- [What to send to support](#what-to-send-to-support)

## The installer stops with AUDIT BLOCKER

The installer checks the machine **before it changes anything**. A blocker means nothing was installed; fix the cause and run
the same command again (it is safe to repeat).

| Message | Cause | Fix |
|---|---|---|
| `supported platform is Ubuntu 24.04 x86_64 (found: ...)` | Another Ubuntu version, another distribution or ARM | Use Ubuntu 24.04 LTS on x86_64 |
| `systemd is not running as PID 1` | A container or a system without systemd | Use a virtual machine or a physical server |
| `only N MiB RAM; at least 4 GiB is required` | Too little memory | Give the machine 4 GiB or more |
| `less than 4 GiB free under /var/lib` | Not enough disk | Free space or grow the disk (20 GiB or more is advised) |
| `port 80 (portal-http) is held by <process>` | Another web server uses port 80 | Stop or move it: `sudo ss -lntp '( sport = :80 )'` shows what holds it |
| `port 514 (syslog) is held by <process>` | Another syslog receiver uses the log port | Stop it, or install with `--syslog-port 5140` and point the routers at that port |
| `cannot reach api.github.com:443 within 10s` (or `packages.clickhouse.com`, `archive.ubuntu.com`) | A firewall, proxy or DNS problem on the way out | Test `curl -I https://api.github.com`; open the paths listed in the [Installation guide, section 2.2](INSTALL.md#22-network-and-firewall) |

Warnings do not stop the install: fewer than 2 CPU cores, less than 20 GiB free, or a clock that is not yet
NTP-synchronised (`sudo timedatectl set-ntp true`).

## The installer fails or stops half way

- **`curl: (6) Could not resolve host` or `curl: (22)`**: the server cannot reach GitHub. Check DNS and the outbound
  firewall, then run again.
- **The one-line form was cut off** (a dropped connection while piping into `bash`): download the installer first, then run
  it ([Installation guide, section 3.2](INSTALL.md#32-review-first-install)).
- **`Run with sudo`**: the installer must run as root.
- **A package step failed** (`apt` error): run `sudo apt-get update`, fix what it reports (a locked package manager, a
  full disk, an unreachable mirror), then run the installer again. It continues where the machine is, and keeps data and
  secrets that already exist.
- After any fix, `sudo /usr/local/lib/openlog/verify-log-server` tells you whether the system is healthy.

## The portal does not open

1. Is it running? `systemctl status log-portal nginx --no-pager`, and `sudo /usr/local/lib/openlog/verify-log-server`.
2. Does it answer on the server itself? `curl -I http://127.0.0.1/`. If yes, a firewall between you and the server
   is blocking TCP 80 or 443 (or the host firewall: `sudo ufw status`).
3. Is the address right? Use the one the installer printed. After a domain is set up, the server stays reachable at its IP
   address over plain HTTP.
4. Read the log: `sudo journalctl -u log-portal -n 100 --no-pager`.

## I cannot sign in

- The first administrator is `admin`; the password is in `/root/openlog.env`
  (`sudo grep '^PORTAL_ADMIN_' /root/openlog.env`).
- **Five failed attempts from one address lock that address out for ten minutes** (adjustable under **Log Settings**). Wait, then try again.
- Forgotten password: another administrator can set a new one under **Users & Access**. If there is no other administrator,
  contact your provider.

## The licence is refused or the Plan page shows a warning

| Message or state | Meaning | What to do |
|---|---|---|
| `this licence was issued for another server (XXXX-...)` | The licence is tied to a different **Server ID** | Send this server's ID (**Server > Plan**) to your provider for a new licence |
| `the licence signature is not valid` | The text was changed or cut when copying | Copy the whole licence again (one line, starting with `OL1.`) |
| `this licence has already expired; ask your provider for a renewed one` | The subscription ended long ago | Ask for a renewal |
| **Grace period** | The subscription ended less than 14 days ago | Renew now; nothing is limited yet |
| **Expired** | The grace period ended | No new devices or users can be added; collection and search carry on. Renew |
| **Updates ended** | The running version was built after your update period | The server works, but cannot add devices or users. Go back to an older version ([Updates and rollback](UPGRADE.md#7-going-back-rollback)) or renew |
| **Evaluation** | No licence | 3 devices, 3 users and 30 days of logs; install a licence to lift this |

A licence takes effect at once; no restart is needed.

## A device shows unreachable

The portal could not log in to the router's API. Open the device for the exact reason.

- **No answer from the address and port**: wrong address or API port; the router's API service is off
  (`/ip service print`); or a firewall blocks the portal ([Connect MikroTik routers, section 3](ROUTERS.md#3-allow-the-api-for-the-log-server-only)).
- **The router's log shows `login failure for user ... via api`**: the API user name or password saved in the portal is
  wrong. Open **Devices > Edit**, correct them and press **Save device**.
- The API user must belong to a group with at least `read,api` rights.

## A device shows no logs

Work down the list; stop at the first thing that is wrong.

1. **Is the device enabled and verified in the portal?** Disabled devices and routers that were never verified are refused.
2. **Does the router's identity match?** `/system identity print` on the router must equal the identity shown on the device.
   After a rename, press **Save device** to accept it. Use only letters, digits, dot, dash and underscore.
3. **Is the logging action right for the RouterOS version?** On RouterOS 7 use `remote-log-format=syslog`; on RouterOS 6
   `bsd-syslog=yes`. `bad parameter bsd-syslog` means you used the RouterOS 6 command on RouterOS 7; `action name can
   contain only letters and numbers` means the name had a hyphen (use `evidencelog`). The portal's **Log setup commands**
   writes the right ones ([Connect MikroTik routers, section 5](ROUTERS.md#5-send-the-logs)).
4. **Are the commands all there?** On the router: `/system logging action print where name=evidencelog` and
   `/system logging print where action=evidencelog`. If you pasted into a terminal and it stayed on `...` lines, a quote was
   left open; use the **Copy** buttons.
5. **Does the log port reach the server?** UDP and TCP 514 (or your port) from the router; check the router's output
   firewall and any firewall in between. On the server: `sudo ss -lnu | grep ':514 '`.
6. **Does the router generate NAT logs?** The mangle rule with `action=log` must exist and match your subscribers' range.
7. **Wait a minute**: the list of accepted routers is refreshed every few seconds, then logs flow.

## Logs arrive but the dashboard is behind

- `sudo /usr/local/lib/openlog/verify-log-server` shows the worker and the reconciler health, and a line
  `udp receive-buffer drops since boot`. A number above 0 means bursts were dropped by the kernel; ask support about tuning.
- The worker health: `curl -fsS http://127.0.0.1:9181/readyz`; its log:
  `sudo journalctl -u openlog-syslog-worker -n 100 --no-pager`.
- A full disk stops ingestion: see [The disk is filling up](#the-disk-is-filling-up).

## HTTPS could not be set up

The server stays on HTTP, so nothing is locked out. The usual causes:

- The DNS name does not point at this server yet (`getent hosts noc.example.com` must show it).
- Ports 80 and 443 are not reachable **from the internet** (Let's Encrypt must connect to port 80).
- **Organization > Contact** has no Support e-mail (the certificate needs a contact).
- A name that is an IP address cannot get a certificate.

Fix the cause, then press **Set up HTTPS** again on **Server > Domain**.

## An update failed or was undone

- The portal's progress window shows the failing step. **If the new version did not start, the previous one was put back
  automatically**: the server is as before the update.
- Read what happened: `sudo tail -n 80 /var/lib/openlog-sysops/update.log`.
- Common causes: the server cannot reach GitHub; the licence does not cover the new version (the Updates page says so);
  another change was still running (wait and try again); a full disk.
- To go back to an earlier installed version: [Updates and rollback, section 7](UPGRADE.md#7-going-back-rollback).

## The disk is filling up

1. See where it goes: `df -h` and `sudo du -xh /var/lib/clickhouse --max-depth=1 | sort -h | tail`.
2. Grow it: **Server > Storage** shows each file system and how far it can be grown without a restart (take a snapshot or
   backup first).
3. Or keep logs for fewer days: **Log Settings** (default 180 days; cleanup runs nightly).
4. Do not delete files under `/var/lib/clickhouse` or `/var/lib/postgresql` by hand.

## What to send to support

1. **In the portal: Report a problem** (bottom of the sidebar). It makes a report with a reference number and the
   technical details of the server, without passwords, keys, router logins or stored logs. Download it or e-mail it, and quote
   the reference number.
2. If the portal does not open, send these instead:

   ```bash
   sudo /usr/local/lib/openlog/verify-log-server
   sudo journalctl -u log-portal -n 100 --no-pager
   sudo tail -n 80 /var/lib/openlog-sysops/update.log
   ```

3. Say what you did, what you expected, what happened, and when it started (and which version you were on before an
   update).

Contact the provider who supplied your licence.
