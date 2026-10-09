# Changelog

Each release has a section headed `## VERSION`. `deployment/release.sh publish` puts that section into the
release notes, and the portal shows it on Organization > Updates ("What is new").

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
