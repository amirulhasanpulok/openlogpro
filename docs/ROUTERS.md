# Connect MikroTik routers

Each router needs three things: a **read-only API account** so the portal can read its identity and PPPoE sessions,
the **device entry** in the portal, and the **logging commands** that send its NAT and PPPoE logs to your Openlog server.
This guide covers RouterOS 7 and RouterOS 6 (version 1.0.5 of the portal writes the right commands for both).

**Contents**

1. [How a router's logs are accepted](#1-how-a-routers-logs-are-accepted)
2. [Create a read-only API account](#2-create-a-read-only-api-account)
3. [Allow the API for the log server only](#3-allow-the-api-for-the-log-server-only)
4. [Add the device in the portal](#4-add-the-device-in-the-portal)
5. [Send the logs](#5-send-the-logs)
6. [Check that it works](#6-check-that-it-works)
7. [Changing or removing things later](#7-changing-or-removing-things-later)

## 1. How a router's logs are accepted

A log message is kept only when **all** of these are true:

1. the router was added in the portal (**Devices > Add device**) and its API login was verified;
2. the device is **enabled**;
3. the router's name inside the message (its `/system identity`) is the identity **saved** for that device.

The sender's IP address plays no part. A message from an unknown router, a disabled or deleted device, a router whose
identity differs, or with no identity at all is dropped before it reaches the evidence store. So:

- keep every router's identity **unique**, and use only letters, digits, dot, dash and underscore in it;
- if a router is renamed, its logs are refused until an administrator opens **Devices > Edit** and presses **Save device**
  to accept the new name (the device shows **Identity changed**, and a Telegram alert can tell you).

## 2. Create a read-only API account

On the router (WinBox > New Terminal):

```routeros
/user group add name=evidence-read policy=read,api
/user add name=log group=evidence-read password="A-STRONG-UNIQUE-PASSWORD"
```

Use a different password on every router, and never a full or write-capable account for evidence collection. If the
group or user already exists, look at it first instead of adding a duplicate:

```routeros
/user group print detail where name="evidence-read"
/user print detail where name="log"
```

## 3. Allow the API for the log server only

The portal connects to the router's plain RouterOS API (TCP). Pick the port (MikroTik's default is **8728**) and use
the same one when you add the device.

```routeros
/ip service set api disabled=no port=8728 address=<LOG_SERVER_IP>/32
```

If the router's input firewall ends in a drop rule, accept the connection **above** that rule:

```routeros
/ip firewall filter add chain=input action=accept src-address=<LOG_SERVER_IP> protocol=tcp dst-port=8728 comment="Evidence server API"
```

Replace `<LOG_SERVER_IP>` with your Openlog server's address. Do not open the API to a whole network.

## 4. Add the device in the portal

1. **Devices > Add device.**
2. Enter a name, the router's management address and the API port, the API user (`log`) and its password, and the
   role: a **NAT** router (writes the NAT logs), an **Access (PPPoE)** router (writes the PPPoE sessions), or a router
   that does both.
3. Save. The portal logs in, reads the router's identity and shows the device. If it cannot, it says why (wrong
   password, port blocked, API not enabled).

The same dialog has **Log setup commands**: the commands in [section 5](#5-send-the-logs), already filled in with this
server's address and log port, with a **Copy** button.

## 5. Send the logs

Run these in the router's terminal, in order. Choose the block for your **RouterOS version** (`/system resource print`
shows it). In the portal, **Log setup commands** has a **RouterOS 7 / RouterOS 6** choice and writes exactly these.

Replace `<LOG_SERVER_IP>` and `<LOG_PORT>` (514 unless you changed it under **Server > Network**).

### 5.1 The logging action

**RouterOS 7**

```routeros
/system logging action add name=evidencelog target=remote remote=<LOG_SERVER_IP> remote-port=<LOG_PORT> remote-protocol=udp remote-log-format=syslog syslog-time-format=bsd-syslog syslog-facility=local6
```

**RouterOS 6**

```routeros
/system logging action add name=evidencelog target=remote remote=<LOG_SERVER_IP> remote-port=<LOG_PORT> bsd-syslog=yes syslog-facility=local6
```

The name has **letters and digits only**: RouterOS 7 refuses a hyphen ("action name can contain only letters and
numbers"). `remote-log-format=syslog` (RouterOS 7) and `bsd-syslog=yes` (RouterOS 6) are what make the router put its
name in every message; without it the portal drops the logs. RouterOS 7 has no `bsd-syslog` parameter and answers
`bad parameter` to it; if you see that, you are using the RouterOS 6 command on RouterOS 7, or the other way round.

If an action called `evidencelog` already exists, remove it and add it again:

```routeros
/system logging action remove [find where name="evidencelog"]
```

### 5.2 What to send

A NAT router sends its firewall logs, an access router its PPPoE logs, a router that does both sends both:

```routeros
/system logging add topics=firewall,!debug action=evidencelog
/system logging add topics=pppoe,!debug action=evidencelog
```

### 5.3 Log NAT connections when they close (NAT routers)

This writes one log line for each TCP connection as it ends (FIN), for traffic from your subscribers' private range.
Replace `10.176.0.0/16` with the private range your subscribers use (the portal's dialog has a field for it).

```routeros
/ip firewall mangle add chain=prerouting action=log tcp-flags=fin connection-state=established protocol=tcp src-address=10.176.0.0/16 comment=nat-evidence
```

- Make sure the range covers **only** subscriber private addresses.
- This can create a large amount of log volume: check the router's CPU and the server's disk first, and start with one
  router.
- If the router has an older evidence rule (comment `NAT evidence logging`), remove it first so connections are not
  logged twice: `/ip firewall mangle remove [find comment="NAT evidence logging"]`.

### 5.4 Let the logs through

The log port (UDP and TCP, 514 by default) must be reachable from the router to the log server, and routers' addresses
should be the only ones allowed to reach it ([Installation guide, section 9](INSTALL.md#9-run-it-in-production)).

## 6. Check that it works

On the router:

```routeros
/system logging action print where name=evidencelog
/system logging print where action=evidencelog
```

The action must show your server's address and port, and the topics must list `firewall` and/or `pppoe`.

In the portal, within about a minute:

- **Devices**: the device changes to **Receiving logs**.
- **Log Stream**: new NAT events appear as subscribers browse.
- **Search Log**: search by PPPoE user, address or time.

If it does not, open the device: the portal gives a plain-words diagnosis (router offline, API blocked, identity
differs, logs not arriving, the action missing or changed). More in
[Troubleshooting](TROUBLESHOOTING.md#a-device-shows-no-logs).

## 7. Changing or removing things later

| Task | How |
|---|---|
| Change the log port | **Server > Network > Log receiving port.** The old port stays open next to the new one until you press **Stop listening**, so routers not yet updated are not cut off. Then update `remote-port` on each router |
| Rename a router | Rename it on the router, then **Devices > Edit > Save device** to accept the new identity |
| Stop collecting from a router | **Devices**: disable the device (its logs are refused at once), or delete it |
| Remove the evidence rule | `/ip firewall mangle remove [find comment=nat-evidence]` |
| Remove the logging | `/system logging remove [find action=evidencelog]`, then `/system logging action remove [find name=evidencelog]` |
