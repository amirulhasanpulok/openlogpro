# Changelog

Each release has a section headed `## VERSION`. `deployment/release.sh publish` puts that section into the
release notes, and the portal shows it on Organization > Updates ("What is new").

## 1.0.39

- The NAT log worker now automatically spreads its work across every CPU core the server has,
  on every update -- not only on a fresh install. A server that has been running as a single
  process grows into one process per core (up to 16) the next time it is updated, with no manual
  step. Shard count only ever grows to match more CPUs; it is never reduced on its own.

## 1.0.38

- The "Powered by Synava Technologies Ltd" credit on the sign-in page and in the sidebar is now
  mandatory -- it always shows, with no setting, environment variable or API field anywhere that
  can hide it or rename who it credits.
- Fix: the Devices card on the dashboard could flash hidden-then-visible on every page reload,
  unlike every other card. It now renders immediately with the same placeholders the rest of the
  dashboard uses, instead of starting hidden and popping in once its data arrives.
- The sign-in page's left panel now shows a network topology graphic instead of a plain gradient.

## 1.0.37

- Fix: a router's very first logs, sent right after it was added, could be silently and
  permanently lost. The NAT log worker only learns about a newly added router on its next
  background refresh (a few seconds later), but reads each log line exactly once -- any line that
  arrived in that short gap was dropped for good, with no retry. In practice this meant adding a
  router and sending it traffic right away (the normal thing to do) could lose its opening
  activity. The worker now double-checks directly with the database the moment it sees a router
  identity it does not yet recognize, instead of only trusting its own cache, so nothing arriving
  in that gap is lost. Found and reproduced via this project's own install tests (1.0.33's and
  1.0.36's diagnostics additions); confirmed not a one-off, and not specific to any particular
  server size.

## 1.0.36

- No functional change. 1.0.35's own install test still failed in CI on "a router sends logs and
  they can be found" -- confirmed, with the new diagnostics 1.0.35 just added, that the syslog
  lines really do arrive at the worker (`syslog_received`) but are then dropped every time
  (`syslog_dropped`, `syslog_parsed` staying 0) because the worker's own router-attribution cache
  never recognizes the router's identity, for reasons the existing metrics could not narrow down
  further: the router genuinely exists, enabled and verified, by every direct database check, and
  the worker logs no error of any kind. This release adds three more `/metrics` fields
  (`syslog_router_cache_size`, `syslog_ident_cache_size`, `syslog_router_poll_cursor_unixtime`) so
  the next failing run shows whether the worker's background poller ever saw the row at all,
  rather than requiring another guess.

## 1.0.35

- Fix: a router whose stored RouterOS API credential could not be decrypted (corrupted, or set
  before a key rotation) used to stop the reconciler from reconciling *every* router that cycle,
  not just the broken one. It now skips the broken router and keeps going. Caught by this
  project's own install tests before reaching this channel; customers never saw this.
- The install tests themselves now print each NAT syslog worker's own counters and raw-log spool
  state on failure, so a future "events never arrived" failure says where in the pipeline they
  were lost instead of just that they were. No change to the shipped software's own behavior.

## 1.0.34

- No functional change from 1.0.33. That release's own software was correct, but its own
  `checks / tests (race, real databases)` CI job failed on a genuine test bug (not a product
  bug): `collector/router-reconciler`'s rotation regression test started its log watcher with no
  synchronization, then wrote and rotated a tiny (200-line) backlog fast enough that under a
  loaded CI runner the watcher's very first file-open could land *after* the rotation had already
  completed -- a known, documented, out-of-scope gap for this kind of watcher (see that module's
  sibling test in `collector/syslog-worker`), not something the fix for mid-backlog rotation can
  address, because there is no open file descriptor yet to protect. The test now waits for the
  watcher's first read pass to actually land before triggering the rotation it's meant to test,
  matching its own stated intent. Re-cut only to ship that test fix; nothing in the shipped binary
  changed.

## 1.0.33

- No functional change from 1.0.32. That release's own software was already correct (confirmed
  by reproducing its install tests' exact scenario by hand on a running server: 20 events arrived
  in 21 seconds), but a small CI runner doing a fresh database and multi-process install at the
  same time this project's own install tests ran needed longer than this project gave it to
  confirm that. Re-cut only to give that check more time.

## 1.0.32

- Fix: 1.0.31's installer could still fail, a different way than 1.0.30's -- starting the
  allowlist-refresh timer right before the installer's own one-time allowlist refresh could make
  both write the same file at once ("File exists"). The timer now only starts once that one-time
  refresh has already finished. Caught by this project's own install tests before reaching this
  channel; customers never saw 1.0.31.

## 1.0.31

- Fix: 1.0.30's installer could abort partway through on a server where the environment did not
  allow setting up compressed swap (zram) -- for example, no permission to create block devices --
  instead of simply continuing without it. Caught by this project's own install tests before
  reaching this channel; customers never saw 1.0.30.

## 1.0.30

- New: the NAT log worker can now run as several processes sharing the work across CPU cores
  instead of one process regardless of how many cores the server has. A fresh install detects the
  core count automatically; an existing install stays single-process until an administrator asks
  for more, after which it grows automatically on a later reboot if more CPUs become available,
  and never shrinks on its own. Which processor a router's logs land on uses consistent hashing,
  so growing the count later moves only a minority of routers, not nearly all of them.
- Performance: each of those processes used to keep a full, independent copy of every router's and
  subscriber's session data in memory, so splitting the work across more processes multiplied
  memory use by that many times over. Each one now only keeps the data for the routers whose logs
  it actually receives (measured on one server: a 4.2 GB working set dropped to 0.7 GB at 10
  processes).
- Performance: the Dashboard could take several seconds to load, worst on a server with a long
  history of stored evidence. Its unique-subscriber count was reading far more history from disk
  than the requested time window actually needed; it now reads only what the window needs
  (measured: 1.8-4.2s dropped to under 0.1s for that one number, and the page itself now caches its
  summary briefly so a normal auto-refresh does not repeat the full cost every time).
- Fix: a router that merely timed out during a reconciliation cycle used to disappear from the
  Dashboard's Active Subscribers count entirely, even though its sessions were still open; it now
  keeps contributing its last known count until the next successful check.
- Fix: during an outage affecting several routers at once, reconciliation worked through them in no
  particular order, so a healthy router queued behind several unreachable ones could wait minutes
  for its turn. Routers that failed last cycle are now reconciled last, so healthy ones are never
  delayed by ones already known to be down.
- Fix: a single oversized, malformed line reaching the router reconciler's login/logout tracking
  could stop it entirely instead of being skipped and logged, the same protection the NAT log
  worker already had.
- Fix: a transient database error while checking the server's own clock (used to detect it being
  rolled back to revive an expired licence) used to skip that check instead of still cross-checking
  the audit trail, narrowing a defense already in place for exactly this kind of bypass.
- Hardening: the database connection pools the NAT log worker and the router reconciler open are
  now bounded instead of unlimited, matching the portal's own pools -- a burst of connection
  attempts (for example, several processes starting at once) can no longer exhaust the database's
  connection limit on its own.
- New installs and updates now also set up compressed RAM-backed swap ahead of any disk-backed swap
  already configured, giving a server under temporary memory pressure a faster fallback than disk
  swap alone.

## 1.0.29

- Fix: hardened licence enforcement against two more edge cases involving the server's own clock and
  hardware-change grace period, so an installation cannot be kept artificially within a grace period
  that has really already ended.
- Performance: the NAT log worker's router/subscriber lookup no longer re-reads the full
  `routers`/`sessions` tables every few seconds on the same path that parses every log line. It
  now keeps its lookup cache current with small, incremental queries instead, so performance no
  longer degrades as the number of devices and subscribers grows. No change in what gets matched
  or logged.

## 1.0.28

- Hardening: the database migrations that run on every startup now take a brief lock first, so
  two processes can never race to create the same table/trigger for the first time. No change for
  a normal single-server install (the lock is acquired and released instantly); this only matters
  for a future clustered deployment, which cannot start safely without it.

## 1.0.27

- Server Settings > Updates > Installed versions now only offers going back to the release right
  before the one running (to undo the last update), not an arbitrary older kept release.
- Idle sign-out (Settings) now defaults to 5 minutes instead of 30 on a new install.
- Performance: several places that made one database query per row in a loop (checking a batch of
  device/user ids, assigning several devices to a user, the notifier explaining a batch of silent
  devices) now make one query for the whole batch; connection pools to Postgres and ClickHouse are
  now bounded instead of unlimited.

## 1.0.26

- Removed the **Services** card from the Dashboard. Service status is still on System > Service info;
  this only removes the dashboard shortcut card. An administrator who had added it to a customized
  minimal dashboard view will see one fewer card there; nothing else changes.

## 1.0.25

- Fix: the "Powered by" credit (Organization > General > "Show 'Powered by ...'", off by default) named
  a generic placeholder vendor instead of Synava Technologies Ltd, the company that develops this
  product.

## 1.0.24

- New: a **Network** card on the Dashboard graphs this server's own network interface traffic
  (in/out, last hour, peaks, and the link speed when the driver reports one) so a bottleneck on the
  server's own NIC is visible at a glance, the same way CPU and memory already are. Sampled by the
  portal itself every 10 seconds, kept in memory only; nothing is installed and nothing is written to
  disk.

## 1.0.23

- New: the MikroTik setup script's **subscriber private ranges** field now takes several ranges,
  separated by commas (for example `10.10.1.0/24,10.10.2.0/24,10.10.3.0/24`), not just one. The
  script builds a firewall address list from them and the NAT logging rule matches against that
  list, instead of a single subnet written directly into the rule — so a network whose subscribers
  span more than one private range no longer has to pick just one or leave some of it unlogged.

## 1.0.22

- Fix: hardened licence enforcement against an edge case that could let an installation end up with
  more devices or users than its licence allows. It now notices such a mismatch on its own and locks
  (no new devices or users, the same way an expired licence locks) until it is resolved.

## 1.0.21

- Fix: Server Settings > Storage > Expand refused a data disk that was formatted directly with no
  partition table at all (a common way to add and use a new disk), even when the disk itself had
  genuinely grown. It is now offered the same as a partition or a logical volume.

## 1.0.20

- New: **Users & Access > System Log access** — restrict a user to specific System Log devices
  (MikroTik, Cisco, OLT), the same way device access already restricts which NAT routers they can
  search. Before this, anyone with the System Log menu saw every configured device's changes
  regardless of their NAT router assignment, because System Log devices are kept in their own,
  separate list and were not covered by that restriction at all. Administrators always see every
  device, same as before.
- New: **idle sign-out**. After a period with no activity (Settings, default 30 minutes; 0 turns it
  off), the portal warns with 60 seconds left and signs out on its own if nothing answers; any
  response keeps the session going. Protects a session left open and unattended. Staying active in
  one browser tab keeps every other open tab of the same session from timing out on its own too.

## 1.0.19

- Fix: setting up HTTPS (Server Settings > Domain) could fail with a confusing "Invalid response ...
  404" from Let's Encrypt and no clue why, when the domain resolved to this server **and** to some
  other address too — most often a stale or wrong AAAA (IPv6) record left on a server that has no
  IPv6. Let's Encrypt is free to use any address a domain resolves to, so it would sometimes pick the
  wrong one and fail. The domain check now catches this itself, before anything is touched, and names
  the extra address so it can be removed at the DNS provider.

## 1.0.18

- Fix: Server Settings actions that restart something (an update, a domain/network/storage change) could
  show a raw, unreadable server error page instead of a real message if a request landed in the instant
  the portal itself was down restarting — most visibly, clicking Update again right after an update was
  already underway. These actions now say plainly that the server did not answer and to try again in a
  moment, same as any other error on that page.

## 1.0.17

- New: **Server Settings > Storage > Disks for logs** — choose which file systems hold the evidence
  logs, any time after install, without touching the command line. Tick the system disk and any added
  disks to hold logs together (each day's new logs go to whichever has the most room); untick one and
  its older logs stay readable while the portal moves them onto the ticked disks in the background,
  a partition at a time. A blank, unused disk can be prepared from the same page — partitioned,
  formatted and mounted at `/data`, `/data2` and so on, with a password and the device name typed again
  to confirm — and added to the disks for logs in the same step. This replaces needing to pick a log
  directory at install time with something better: change it whenever storage needs change, not only
  once.
- New: **System Log** (Operations, was **Change Log**) — who changed what, on MikroTik, Cisco and OLT
  devices: the command, the user, where it came from and the device's own log line, newest first, with
  date/device/user/kind/text filters, CSV export and its own audit trail. A chip per device filters to
  just that device's changes in one tap, and a Type filter (MikroTik/Cisco/OLT) sits beside it. The
  Devices tab shows exactly how to add a device — the log-setup steps and commands for RouterOS 7,
  RouterOS 6, a router already sending NAT logs, Cisco IOS/IOS XE and NX-OS, and OLT settings — and how
  long changes are kept (a year by default). Configuration-change lines travel their own path end to
  end (their own syslog spool, never mixed with NAT evidence) so collecting them cannot affect NAT log
  collection. Saved links and roles for the old name, Change Log, keep working.
- Changed: **Dashboard** —
  - **Settings > Dashboard** (administrators) now chooses what everyone sees: **Detailed** (every
    figure and graph, as before) or **Minimal** (collector, events today, log rate, active subscribers
    and public NAT IPs in one row, then devices, services and the log partition — no graphs, so that
    data is not even requested). In Minimal, which of 16 cards show is also chosen there.
  - The header is now the title and a small light instead of a Refresh button: green while the figures
    are current, amber if part of them could not be read or they are getting old, red if the server did
    not answer; a click refreshes at once. The server's own clock sits beside it (the sidebar no longer
    carries a separate status card).
  - "Raw log size" is now "Raw log 24 h" and actually covers the last 24 hours; before, it grew with
    however many days of logs the server had kept, which was not what the card's name implied.
  - Storage by date's bar labels are now real calendar dates ("7 Oct"), evenly spaced and matching the
    organization's date format everywhere on the card, instead of unevenly rounded "10-07"-style labels
    that could skip a day.
- Changed: the sidebar's **Settings** section is now a group that opens and closes, holding a new
  Settings > Dashboard page alongside Users & Access, Log Settings, Notifications, Organization, Server
  Settings and Activity Log.
- Changed: a server with no company name set of its own now shows **Log Server** instead of **Evidence
  Portal** as its default name. Nothing changes for any organization that has already set its own name
  or branding.
- Changed: the Telegram **daily report** is sent at midnight (the storage time zone) by default instead
  of a time administrators had to set themselves, and the hour is now chosen from a readable list
  ("12:00 AM (midnight)", "1:00 AM", ...) instead of typed as a bare number.
- Fix: a Devices page could split one NAT router from its own access servers across two pages. Rows that
  belong together now always land on the same page.
- Fix: on a phone (390px and narrower), a few controls were hard to use — the page-number list on a
  wide table could push First/Last off screen, a handful of buttons and switches were under the
  comfortable 32px tap size, and some labels were set smaller than they should be. All fixed; nothing
  scrolls sideways on a phone now.

## 1.0.16

- Hardened: every portal page's inline script now runs only under a fresh, unguessable permission
  (a "nonce") generated for that one request. Before, it was allowed to run because the page's security
  policy (CSP) trusted *any* inline script — which would equally have trusted one an attacker managed to
  slip into the page some other way. No visible change for anyone using the portal normally.

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
