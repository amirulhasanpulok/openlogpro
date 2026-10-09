# Openlog

**NAT evidence and PPPoE attribution for MikroTik networks.** A web portal that receives the logs of your MikroTik
routers, matches every connection to the subscriber who made it, and keeps it searchable as evidence. It runs on one
Ubuntu server: portal, PostgreSQL, ClickHouse, syslog-ng and Nginx, installed and updated by one command or from the
portal.

**Current version: 1.0.6** | built 2026-10-09T13:06:17Z | [Release notes](CHANGELOG.md) | [All releases](https://github.com/amirulhasanpulok/openlogpro/releases)

This repository holds **releases only**: finished programs, no source code.

## Install in one command

On a clean **Ubuntu 24.04 LTS (x86_64)** server, as root or with `sudo`, with internet access:

```bash
curl -fsSL https://github.com/amirulhasanpulok/openlogpro/releases/latest/download/install.sh | sudo bash
```

The server starts in **evaluation mode** (up to 3 devices, 3 users, 30 days of logs). **A licence belongs to one server,
so you get it after the install:** open **Server > Plan**, send the **Server ID** shown there to your provider, and
paste the licence you receive into the same page. Nothing is restarted.

The installer checks the machine first, installs everything, verifies it, and prints the address and the first
sign-in. It takes about 5 to 10 minutes. Prefer to read the script before running it? See
[Review first](docs/INSTALL.md#32-review-first-install).

## Documentation

| Guide | For |
|---|---|
| [Installation guide](docs/INSTALL.md) | Requirements, network and firewall, install options, first sign-in, licence, HTTPS, hardening, file and service reference |
| [Connect MikroTik routers](docs/ROUTERS.md) | The API account, the logging commands (RouterOS 7 and 6), verifying that logs arrive |
| [Updates and rollback](docs/UPGRADE.md) | Updating from the portal or the command line, licence coverage, going back to an earlier version |
| [Troubleshooting](docs/TROUBLESHOOTING.md) | Installer blockers, sign-in, licence, routers that send no logs, failed updates, what to send to support |
| [Release notes](CHANGELOG.md) | What changed in every version |

## At a glance

| | |
|---|---|
| Platform | Ubuntu 24.04 LTS, x86_64, systemd. 2 CPU cores or more, 4 GiB RAM or more, 4 GiB free disk to install (20 GiB or more advised; evidence grows with traffic) |
| Ports | In: 80 and 443 (portal), 514 UDP and TCP (routers' logs; changeable). Out: GitHub and the package mirrors for install and updates, and each router's API port |
| Licence | Signed licence tied to one server; subscription or perpetual. Evaluation mode without one |
| Updates | From **Server > Updates**, or `sudo /usr/local/lib/openlog/update-log-server`. The last releases are kept, so you can go back |
| Support | **Report a problem** in the portal sidebar creates a report with a reference number and the server's technical details, without passwords or logs |

## What is new in 1.0.6

- Fix: the installer, given the example text from the instructions (`--licence 'OL1....'`), only said "must be the whole
  licence text". It now says that this is the example, that nothing was installed, and what to do instead: install without
  `--licence` (evaluation mode) and paste the licence under **Server > Plan** afterwards, because a licence is issued for
  one server from its Server ID. The install guide and the front page of the public repository now start with the command
  **without** `--licence`.
