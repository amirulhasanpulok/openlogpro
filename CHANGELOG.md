# Changelog

Each release has a section headed `## VERSION`. `deployment/release.sh publish` puts that section into the
release notes, and the portal shows it on Organization > Updates ("What is new").

## 1.0.5

- Documentation, published on the front page of the public repository together with each version: an **installation
  guide** (requirements, network and firewall, install options, what the installer changes, first sign-in, licence,
  HTTPS, production checklist, file and service reference), a guide to **connect MikroTik routers** (RouterOS 7 and 6),
  **updates and rollback**, and **troubleshooting**. Every limit, option and command in them is checked against the
  program when the release is built. There is no change to the programs.

## 1.0.4

- Fix: right after the first update that brings **Installed versions**, the list was empty until the next system
  check, because the update ran inside the older helper program, which wrote the machine state without it. The page
  now asks the new helper once, and the list fills in by itself.

## 1.0.3

- Fix: the copy buttons did nothing when the portal was opened by IP address over plain http (the browser's clipboard
  is only there on https pages). They work on every page now, and say so when they cannot copy.
- Fix: the **Log setup** commands for a MikroTik stopped at "bad parameter bsd-syslog" on RouterOS 7, which has no such
  parameter (it is `remote-log-format=syslog` there). The dialog now has a RouterOS 7 / 6 choice and writes the
  commands that fit; the NAT rule no longer ends in an empty quoted prefix (a half-copied quote left the router
  waiting for more input), carries the comment `nat-evidence`, and a check step is added. The logging action is now
  called `evidencelog`: RouterOS 7 accepts letters and digits only in an action name and refused `evidence-log`
  ("action name can contain only letters and numbers"). A router set up earlier under the old name keeps working;
  remove the old action and its rules when you run the new commands, so logs are not sent twice.
- New: **Server > Updates > Installed versions**. The last releases stay installed, and an administrator can go back
  to one (or switch to a newer one) with a password, in the portal or with `rollback-log-server`. The settings are
  backed up first, the earlier release is installed and verified, and the release that was running is put back if it
  does not start. Data is untouched. The install test now goes back through the portal after every update.

## 1.0.2

- New **Administration > Server** page: Domain, Network (with the log port), Storage, Plan (licence) and Updates moved
  there from Organization. Organization now holds only the company's own settings (name, branding, colours, locale,
  contact), so its preview and Save bar no longer appear on tabs that save nothing. Old links such as
  `/settings/company#network` lead to the new place.
- New **Report a problem** (bottom of the sidebar, and on Help & FAQ): a form for what went wrong and how serious it
  is. It creates a report with a reference number (SR-YYYYMMDD-XXXXXX) and the technical details of the server
  (versions, licence, service health, log collection, recent warnings; no passwords, keys, router logins or stored
  logs; the page shows exactly what is included before anything is made). Download it, copy it, or e-mail it to the
  support address of the organization. The portal sends nothing itself; the reference, severity and summary are noted
  in the Activity Log.
- Security: built with Go 1.26.9, which fixes 11 vulnerabilities in Go's standard library (net/http, html/template and
  others) that were published after 1.0.1 was built.

## 1.0.1

- Fix: a server could not be installed from a GitHub release channel. The machine check looked for a host called
  "github:owner" and stopped with "cannot reach"; it now checks api.github.com.
- Fix: Organization > Updates > Update now failed on a real server, because the root helper that runs the update
  could not start apt or sudo. It keeps the rights it needs now.
- Releases are built, tested and published by GitHub Actions. Every release is installed on a brand-new Ubuntu
  machine, used (sign-in, a router, logs, search, export, moving the log port) and updated from the previous release
  through the portal before it is offered.

## 1.0.0

First release.

- NAT evidence: logs from MikroTik routers are matched to PPPoE users and kept searchable (Search Log, Log Stream,
  Active Users, Public IPs) with CSV and Excel export.
- Devices: add, verify and monitor NAT and access routers; per-device log setup commands and a plain-words
  diagnosis when a router stops sending logs; Telegram alerts and a daily report.
- Licensing: signed subscription or perpetual licences, installed on Organization > Plan. A perpetual licence
  includes new versions for up to 5 years.
- Updates from the portal: Organization > Updates shows the newest version and installs it.
- Log receiving port: change the port routers send logs to (default 514) on Organization > Network, keeping the
  old one open until every router has been updated.
- Fixes found in the final check: a router could stop log collection by sending a line with invalid bytes; the NAT
  search answered "store failed" for a mistyped IP address and now says what is wrong; the protocol filter
  now accepts upper case; a router that never ends its reply can no longer fill the server's memory.
