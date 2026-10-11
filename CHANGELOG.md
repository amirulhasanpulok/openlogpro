# Changelog

Each release has a section headed `## VERSION`. `deployment/release.sh publish` puts that section into the
release notes, and the portal shows it on Organization > Updates ("What is new").

## 1.0.15

- Documentation, published on the front page of the public repository: the product description now names what it
  is in the terms people actually search for — **NAT/CGNAT log server**, alongside the existing PPPoE attribution
  wording — and the Public IPs section now says "CGNAT/NAT pool" instead of just "NAT pool". No behaviour change,
  no change to the programs.

## 1.0.14

- Fix: exporting a **Search Log** result that matched more rows than the export cap (Log Settings >
  Search and export policy > NAT export row cap, 20,000 by default) gave no sign that anything was
  left out — the CSV or Excel file simply ended early, indistinguishable from a complete export. The
  page now says so before you download ("Export is limited to the newest N of M; narrow the search for
  a complete export"), and the export itself carries the real matched total so it is never silently
  mistaken for the whole result.

## 1.0.13

- Changed: **Report a problem** now sends to `support@synavatech.com` (was the previous owner's personal
  address). No behaviour changed, only which address.

## 1.0.12

- Changed: the **Server** page is now **Server Settings** in the sidebar and its own title, and its
  **Plan** tab is now **Licence** (it always showed the licence and what it unlocks; the new name says
  so directly). Every customer guide that pointed at "Server > Plan" now says "Server Settings >
  Licence". No behaviour changed, only names.

## 1.0.11

- Fix: the Dashboard could lose previously-loaded Devices and Services data (the section would hide and
  reappear) if a single periodic refresh had a transient failure fetching it — one slow or failed request
  no longer wipes data the page already had; the next successful refresh updates it as normal.
- New: raw log rotation now keeps 1 compressed hour by default (previously 4), matching how little of it
  is normally needed before the data is safely in ClickHouse; the "Effective configuration" fact on Log
  Settings, which had been describing an old scheme ("daily or 256 MB; 7 copies") no longer in use, now
  states what actually runs (hourly; 1 compressed copy).

## 1.0.10

- New: the device and user limits a licence allows are now also enforced by the database itself, not
  only by the portal's own "add device" and "add user" pages. Inserting a row directly into the
  database — something only someone with the server's own root access could do in the first place —
  is refused the same way the portal's own forms already refuse it, instead of silently going around
  the limit. The signed licence is still the only thing that decides what the limit is; this only
  makes sure the database agrees with it.
- New: the audit trail (Activity Log) is a tamper-evident chain: every entry carries a hash of its own
  content linked to the one before it, filled in by the database itself as each entry is recorded, so
  editing or deleting an entry afterwards (outside the normal, logged retention clean-up) breaks the
  chain instead of leaving no trace. A **Verify integrity** button on Activity Log checks the whole
  chain and reports the first entry that does not check out, if any.

## 1.0.9

- Documentation, published on the front page of the public repository: a **"What you can do with it"** section
  describing the Dashboard, Devices, Search Log, Active Users and Public IPs pages, each with a screenshot, plus a
  summary of evidence retention, role-based access, server-bound licensing, signed updates and how **Report a
  problem** reaches the Provider. There is no change to the programs.

## 1.0.8

- Fix: **Report a problem** sent its e-mail link to the organization's own support address
  (Organization > Contact), the contact that organization shows its own subscribers — never to the
  Provider. A report now always goes to the Provider's own address, which does not depend on the
  organization's settings.

## 1.0.7

- New: **releases are signed.** `install.sh` and `update-log-server` now refuse a release whose signature does not
  verify against the Provider's public key; a release before 1.0.7 is unsigned and the new scripts will not install it
  by tag.
- New: a stronger, hardware-bound **Server ID** (24 hex digits: the machine id, the hardware's product UUID and the
  primary network interface's MAC address, each a salted hash). Copying a server (for example cloning a virtual
  machine) now gives it a different Server ID instead of silently reusing the licensed one. A genuine hardware change
  (new host, new network card) keeps working for a 14-day grace period shown on **Server > Plan**, so there is time to
  ask for a new licence. Licences issued for the old, 16-digit Server ID keep working.
- New: release binaries are now built **obfuscated** (names, build paths and string literals scrambled), raising the
  cost of reading them with `strings` or a decompiler. This does not replace the licence agreement's restriction on
  reverse engineering; it makes it more expensive.
- New: an **End User Licence Agreement** (`docs/customer/EULA.md`, also published as `LICENSE.md` on the public
  repository). The first administrator to sign in after installing or updating is asked to accept it in the portal; a
  later version of the agreement asks again. The agreement is a draft and has not had legal review; treat it as a
  starting point, not a final contract.

## 1.0.6

- Fix: the installer, given the example text from the instructions (`--licence 'OL1....'`), only said "must be the whole
  licence text". It now says that this is the example, that nothing was installed, and what to do instead: install without
  `--licence` (evaluation mode) and paste the licence under **Server > Plan** afterwards, because a licence is issued for
  one server from its Server ID. The install guide and the front page of the public repository now start with the command
  **without** `--licence`.

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
