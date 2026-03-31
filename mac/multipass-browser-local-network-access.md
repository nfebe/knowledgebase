# Why Browsers Can't Reach Your Local VM on macOS (But curl and Safari Can)

A deep dive into a maddening macOS Sequoia networking issue that cost hours
of debugging — and how Apple silently locks out every non-Apple app from
your own local network.

## The Setup

We were testing [FlatRun](https://github.com/flatrun), a container orchestration
tool, using a Multipass VM on macOS. The installer sets up Docker, nginx, a
systemd service, and a setup wizard UI on a fresh Ubuntu 22.04 VM. Simple enough.

The VM was created with bridged networking:

```bash
multipass launch 22.04 \
    --name flatrun-test \
    --cpus 2 \
    --memory 2G \
    --disk 20G \
    --network en0
```

This gave the VM two IPs:
- `192.168.4.8` — QEMU internal network
- `192.168.2.176` — bridged to the host's Wi-Fi subnet via `en0`

The installer ran clean. nginx was listening on `0.0.0.0:80`. The agent was
running. Everything looked perfect.

## The Problem

```bash
# This works fine
$ curl -I http://192.168.2.176
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)

# This also works
$ ping 192.168.2.176
64 bytes from 192.168.2.176: icmp_seq=0 ttl=64 time=0.891 ms

# Python can reach it too
$ python3 -c "import urllib.request; print(urllib.request.urlopen('http://192.168.2.176').status)"
200
```

But open `http://192.168.2.176` in Chrome or Firefox? Nothing.
ERR_CONNECTION_REFUSED. NS_ERROR_CONNECTION_REFUSED.

**Safari worked fine.** Because of course it did — it's Apple's own browser.

## The Investigation

### Round 1: Check the Obvious

First instinct — something's wrong with the VM.

```bash
$ multipass exec flatrun-test -- sudo ss -tlnp | grep :80
LISTEN 0 511 0.0.0.0:80 0.0.0.0:* users:(("nginx",pid=...))
```

nginx is listening on all interfaces. No UFW rules. Empty iptables. The VM
side is clean.

### Round 2: Is It a Firewall?

macOS firewall? Off. Checked System Settings > Network > Firewall — disabled.

What about the packet filter?

```bash
$ sudo pfctl -s rules
# (no blocking rules found)
```

Clean.

### Round 3: The VPN Red Herring

We discovered TunnelBear had a **system network extension** still registered
on the machine — showing as `activated enabled` even though the app was
disconnected and the browser extension was off.

```bash
$ systemextensionsctl list
1 extension(s)
--- com.apple.system_extension.network_extension
enabled active  teamID  com.tunnelbear.mac.TunnelBear.PacketTunnel
```

Disabled it. Toggled it. Verified it showed `activated disabled`. Retested.

Still broken. Thanks for nothing, TunnelBear.

### Round 4: Proxy Settings?

VPN tools sometimes leave proxy configurations behind. Checked everything:

```bash
$ networksetup -getwebproxy Wi-Fi
Enabled: No
$ networksetup -getsecurewebproxy Wi-Fi
Enabled: No
$ networksetup -getsocksfirewallproxy Wi-Fi
Enabled: No
$ networksetup -getautoproxyurl Wi-Fi
URL: (null)
Enabled: No
```

No proxies. No PAC files. Nothing.

### Round 5: Is It the Browser or the OS?

This was the key diagnostic. We started a Python HTTP server on the **host
machine** itself:

```bash
$ python3 -m http.server 9998 --bind 192.168.2.101
```

Opened `http://192.168.2.101:9998` in Chrome — **worked instantly**.

So Chrome could reach the host's own LAN IP, but not the VM's LAN IP on the
**exact same subnet**. Both 192.168.2.x addresses, one works, one doesn't.

Meanwhile Safari hit both without issue. The pattern was becoming clear: Apple
software works, third-party software doesn't.

### Round 6: The QEMU Bridge Theory

Multipass on Apple Silicon uses QEMU. Bridged networking via `--network en0`
creates a virtual interface that gets a DHCP lease from the router. We
wondered if the router's ARP table wasn't properly registering the VM's MAC
address, or if macOS was filtering traffic to "unknown" MAC addresses on the
bridge.

Checked from the VM:

```bash
$ ip addr show enp0s2
inet 192.168.2.176/24 brd 192.168.2.255 scope global dynamic enp0s2
```

Valid lease. Proper subnet. The router saw it. curl on the host saw it.
Safari saw it. Chrome and Firefox didn't.

### Round 7: The Answer — macOS Locks Out Non-Apple Apps From Your Own Network

After exhaustive research, we found it.

**macOS Sequoia (15.x) introduced a "Local Network" privacy permission** under
System Settings > Privacy & Security > Local Network. This permission controls
whether applications can communicate with devices on private IP ranges
(192.168.x.x, 10.x.x.x, 172.16-31.x.x).

Here's what's exempt:
- **CLI tools** — `curl`, `ping`, `python3`, anything running in Terminal
- **Safari** — Apple's own browser, naturally

Here's what's blocked by default:
- **Chrome** — needs explicit permission
- **Firefox** — needs explicit permission
- **Every other third-party GUI app** — needs explicit permission

Read that again. Apple silently blocks every non-Apple application from
accessing your own local network, on your own machine, while giving their own
software a free pass. Safari connects to your VM instantly. Chrome gets a
brick wall. Same URL, same network, same machine.

This explains *everything*:
- curl works → CLI, exempt
- ping works → CLI, exempt
- python works → CLI, exempt
- Safari works → Apple's own app, exempt
- Chrome fails → third-party GUI app, silently blocked
- Firefox fails → third-party GUI app, silently blocked

## The Fix

1. Open **System Settings > Privacy & Security > Local Network**
2. Find Chrome, Firefox, and any other third-party browser in the list
3. Toggle them **ON**

That's it. Ten seconds once you know where to look.

## Why macOS Makes This So Painful

1. **No prompt appears.** Unlike Camera or Microphone permissions, macOS
   doesn't always prompt you when a browser tries to access a local network
   address. It just silently blocks the connection. No dialog. No notification.
   No breadcrumb. Just a dead connection that looks like a server problem.

2. **The error messages are actively misleading.** Chrome reports "connection
   refused" — implying the server is rejecting your connection. The server is
   fine. macOS is intercepting the connection before it ever leaves the machine
   and telling you nothing about it.

3. **Safari working is the ultimate misdirection.** When Safari works but
   Chrome doesn't, your brain goes to "this must be a Chrome issue" — maybe
   extensions, maybe HSTS, maybe some Chrome flag. You don't think "Apple is
   giving Safari privileged network access that it denies to competitors." But
   that's exactly what's happening.

4. **CLI tools working sends you in the wrong direction.** curl works, so
   the server must be fine. The browser fails, so the browser must be broken.
   You spend hours debugging the browser, the VM, the network, the firewall —
   everything except the one macOS setting that's actually causing the problem.

5. **It's a new behavior with no migration path.** If you've been doing local
   dev for years on macOS, this permission didn't exist before Sequoia. Apple
   broke decades of expected networking behavior in an OS update and told
   nobody.

6. **VPN extensions make the investigation worse.** If you have or had a VPN
   tool like TunnelBear installed, the leftover system network extension becomes
   the prime suspect. You'll spend significant time investigating the VPN when
   the real issue is a completely separate macOS privacy gate.

7. **The setting is buried.** System Settings > Privacy & Security > Local
   Network. Not under Network. Not under Firewall. It's filed under "Privacy"
   because Apple considers your browser talking to 192.168.x.x on your own
   machine a privacy concern — but only if that browser isn't Safari.

## Things We Tried That Didn't Help

- Disabling TunnelBear's system network extension
- Checking/clearing proxy settings
- Checking macOS firewall (already off)
- Checking VM-side firewall (UFW, iptables — all clean)
- Adding `--network en0` for bridged networking
- Chrome flags (`chrome://flags/#local-network-access-check`)
- Inspecting nginx configs and listening ports
- Checking packet filter rules
- Questioning our life choices

## The Bigger Picture

This is part of a pattern with macOS. Apple keeps adding "security" and
"privacy" features that:

- Silently break things that used to work
- Exempt Apple's own software from the restrictions
- Produce misleading error messages that point away from the real cause
- Bury the settings in places developers won't think to look
- Treat professionals using their own machines like threats to be contained

Local Network access control is a feature that makes sense for random apps
phoning home to IoT devices on your network without your knowledge. It makes
zero sense as a silent default-deny for browsers — the apps people use
specifically to access network resources — with no prompt and misleading
errors. And exempting Safari while blocking Chrome and Firefox isn't security,
it's competitive advantage dressed up as user protection.

## Lessons Learned

1. **Check Local Network permissions first** when non-Apple apps can't reach
   local IPs but CLI tools and Safari can. On macOS Sequoia+, this should be
   step one.

2. **Safari working is a clue, not a solution.** If Safari reaches a local IP
   but Chrome/Firefox can't, the problem is almost certainly the Local Network
   permission — not the browsers, not the server, not the network.

3. **Don't chase VPN ghosts.** If you have a VPN extension/app, verify the
   issue on a clean browser profile first:
   ```bash
   open -a "Google Chrome" --args --user-data-dir=/tmp/chrome-test
   ```

4. **The "curl works but browser doesn't" pattern** on macOS almost always
   points to either proxy settings or Local Network permissions. Check both
   immediately and save yourself the hours we lost.

## Quick Reference

| | |
|---|---|
| **Symptom** | Chrome/Firefox can't reach local/VM IPs, but `curl`, `ping`, and Safari work fine |
| **OS** | macOS Sequoia (15.x) or later |
| **Cause** | Apple blocks third-party apps from local network by default, exempts its own software |
| **Fix** | System Settings > Privacy & Security > Local Network > Enable for affected browsers |
| **Time to fix** | 10 seconds, once you know where to look |
| **Time to figure out** | Hours of your life you won't get back |
