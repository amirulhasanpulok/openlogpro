# Updates and rollback

How to move an installed server to a newer version, how the licence decides which versions you may install, and how to
go back to an earlier version if a new one misbehaves. This guide describes version 1.0.8.

**Contents**

1. [How updates work](#1-how-updates-work)
2. [Before you update](#2-before-you-update)
3. [Update from the portal](#3-update-from-the-portal)
4. [Update from the command line](#4-update-from-the-command-line)
5. [Which versions your licence covers](#5-which-versions-your-licence-covers)
6. [After the update](#6-after-the-update)
7. [Going back (rollback)](#7-going-back-rollback)
8. [Reporting a problem with a version](#8-reporting-a-problem-with-a-version)

## 1. How updates work

- Releases are published in this repository. Each release is **tested on a clean Ubuntu machine before it is offered**:
  a fresh install, the update from the previous release, and going back again.
- The portal **looks** for a newer version in the background (the answer is cached for ten minutes) and shows it under
  **Server > Updates**, with what changed. **It never installs anything by itself.** An update starts only when an
  administrator presses **Update now** and confirms with their password, or runs the command line update.
- An update downloads the release, checks its SHA-256 and your licence, backs up the settings, installs it, restarts the
  portal for a minute or two (log collection carries on), and verifies the server. **If the new version does not start,
  the version that was running is put back automatically.**
- Your data is kept: database changes are only ever additive, so an earlier version can still read what a newer one wrote.
- The last three releases stay installed on the server, which is what makes a rollback possible.

## 2. Before you update

| | |
|---|---|
| Read | What is new, on **Server > Updates** or in the [release notes](../CHANGELOG.md) |
| Back up | `sudo systemctl start openlog-backup.service` (the update also does this itself), and a VM snapshot if the machine is a VM |
| Time | Choose a quiet hour: the portal is unavailable for part of the update; routers keep sending, the log receiver keeps running |
| Licence | Check that your licence covers the new version ([section 5](#5-which-versions-your-licence-covers)) |
| One at a time | Do not start an update while another change (domain, network, storage, log port) is running; the portal refuses |

## 3. Update from the portal

1. **Server > Updates.** The page shows the installed version, the newest version, what is new and whether your licence
   covers it. **Check now** asks again immediately.
2. Press **Update to X.Y.Z**, enter your password, and confirm.
3. A progress window follows each step. The portal restarts during the update; the window waits and continues when it is
   back, and the page reloads when the update is done.

If the window reports a failure, nothing is lost: the version that was running has been put back. Open **Report a
problem** ([section 8](#8-reporting-a-problem-with-a-version)).

## 4. Update from the command line

```bash
sudo /usr/local/lib/openlog/update-log-server
```

It does the same steps and prints them. Use it when the portal cannot be reached. Run it again at any time: when the
server already runs the newest version it says so and changes nothing.

## 5. Which versions your licence covers

A licence is checked **against the date a version was built**:

| Licence | Versions you may install |
|---|---|
| Subscription | Versions built while the subscription is valid, plus 14 days of grace |
| Perpetual | Versions built within the update period printed on the licence (up to 5 years from the day it was issued) |
| None (evaluation) | Every version, within the evaluation limits (3 devices, 3 users, 30 days of logs) |

A version outside your period is shown as **not covered** and cannot be started from the portal. If one is installed
anyway, the server **keeps working**, but cannot add devices or users until the licence is renewed. Nothing is stopped or
deleted because of a licence. The Plan page (**Server > Plan**) shows your dates.

## 6. After the update

```bash
sudo /usr/local/lib/openlog/verify-log-server
```

It must end with `VERIFICATION OK`. Then check that **Server > Updates** shows the new version as running, that the
device list shows your routers as **Receiving logs**, and that Search Log still finds recent events.

## 7. Going back (rollback)

If the new version misbehaves, you can go back to an earlier one that is still installed on the server.

### From the portal

**Server > Updates > Installed versions** lists the releases on the server (the running one is marked). Press **Go back
to this** next to the one you want and confirm with your password. The server backs up the settings, switches to that
release, installs and verifies it, and restarts the portal for a minute or two. If that release does not start, the
release that was running is put back by itself. The page reloads when it is done.

### From the command line

```bash
sudo /usr/local/lib/openlog/rollback-log-server            # lists the installed releases
sudo /usr/local/lib/openlog/rollback-log-server 1.0.0      # goes to that one
```

### What to know

- **Your data is not touched.** Database changes are additive, so the earlier version reads the same data. Events that
  arrived while the newer version was running are kept.
- The **newest version stays on offer** under Updates. Do not press **Update now** again until the problem is fixed
  in a newer release.
- You can switch **forward** again to a newer release that is still installed the same way.
- Only the last three releases are kept; to go back further, install that release explicitly with
  `--release-tag` ([Installation guide, section 3.3](INSTALL.md#33-options)).
- A rollback keeps your licence, devices, users, log port, domain and HTTPS.

## 8. Reporting a problem with a version

Before you go back, use **Report a problem** (bottom of the portal sidebar) so the versions and the state of the server
are recorded with a reference number. Download the report or e-mail it to your support contact and quote the reference
number. If the portal cannot be opened, send the output of `sudo /usr/local/lib/openlog/verify-log-server` and
`sudo tail -n 80 /var/lib/openlog-sysops/update.log`.
