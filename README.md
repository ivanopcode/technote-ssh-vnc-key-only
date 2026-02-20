# Technical Note: Remote Access via SSH Key-Only + VNC Tunnel on macOS Host

## Goal
Use a hardened remote access model on the target host where SSH is key-only and VNC is reachable only through an SSH tunnel.

## Problem Statement
- Remote desktop became unreliable after switching auth methods.
- Direct VNC access is not required on the public internet.
- SSH must remain the single remote control-plane for administration.

## Target State
1. SSH uses public key authentication only.
2. Public VNC/Screen Sharing ports (`5900/5901`) are not exposed publicly.
3. VNC sessions are opened through local port-forwarding (`ssh -L ...`).

## Server-Side SSH Configuration (macOS)
To enforce key-only authentication, use these directives:

```conf
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
PubkeyAuthentication yes
AuthenticationMethods publickey
PermitEmptyPasswords no
PermitRootLogin no
```

*Older macOS versions may rely on `ChallengeResponseAuthentication` instead of or in addition to `KbdInteractiveAuthentication`.*

**Where to apply this configuration:**

- **Method A (recommended for modern macOS):** create a file in `/etc/ssh/sshd_config.d/` (for example, `100-custom-auth.conf`).
  macOS updates often overwrite `/etc/ssh/sshd_config`, while included drop-ins usually remain.
- **Method B (traditional):** edit `/private/etc/ssh/sshd_config` directly.
  Recheck the file after major macOS upgrades.

Restart SSH after edits:

```bash
sudo launchctl kickstart -k system/com.openssh.sshd
```

## Firewall/Services
- Keep port `22` open for SSH only.
- Restrict inbound access to VNC ports (`5900`, `5901`) at the firewall/network edge.
- Confirm Screen Sharing service remains enabled for local/tunneled sessions.

## Daily Access Flow
1. Start a tunnel to the host:

```bash
ssh -N -L 5900:localhost:5900 relux
```

2. Open VNC client to local endpoint:
- Address: `127.0.0.1`
- Port: `5900`
- On macOS: `vnc://127.0.0.1`

3. Authenticate in VNC with the remote macOS user credentials.

## Helpful Validation Commands

### 1) Verify password login is rejected

```bash
ssh -v \
  -o PreferredAuthentications=password \
  -o PubkeyAuthentication=no \
  -o PasswordAuthentication=yes \
  -o NumberOfPasswordPrompts=0 \
  <user>@<macos-host-ip>
```

Expected result: `Permission denied (publickey).`

### 2) Verify tunnel and service health

Confirm tunnel endpoint is alive locally:

```bash
nc -vz 127.0.0.1 5900
```

This confirms the tunnel endpoint is present; it does **not** verify remote VNC authentication.

Confirm SSH reachability:

```bash
nc -vz <remote-host> 22
```

## Why this Works
- SSH auth is cryptographically bound to keys, removing password brute-force and shared-secret risks.
- VNC traffic is encrypted inside SSH instead of being exposed to LAN/WAN.
- Public attack surface is reduced by not exposing VNC ports.

## Notes
- A prior SSH session like `ssh relux` confirming login only proves host-level SSH reachability.
- If public VNC still appears reachable unexpectedly, check firewall rules and cloud security-group / reverse-proxy ACLs.
- For long-running sessions use background tunnel mode:

```bash
ssh -f -N -L 5900:localhost:5900 relux
```
