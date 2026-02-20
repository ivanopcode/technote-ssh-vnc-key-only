# Technical Note: Remote Access via SSH Key-Only + VNC Tunnel on macOS Host

## Goal
Use a hardened remote access model on the board host where SSH is key-only and VNC is reachable only through an SSH tunnel.

## Problem Statement
- Remote desktop became unreliable after switching auth methods.
- Direct VNC access is not required on the public internet.
- SSH must remain the single remote control control-plane for administration.

## Target State
1. SSH uses public key authentication only.
2. Public VNC/Screen Sharing ports (`5900/5901`) are not exposed publicly.
3. VNC sessions are opened through local port-forwarding (`ssh -L ...`).

## Server-Side SSH Configuration (macOS)
Add the following to `/etc/ssh/sshd_config` or a drop-in in `/etc/ssh/sshd_config.d/`:

```conf
PasswordAuthentication no
ChallengeResponseAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AuthenticationMethods publickey
PermitEmptyPasswords no
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
- Confirm SSH uses keys and rejects password prompts:

```bash
ssh -vvv -o PasswordAuthentication=no relux
```

- Confirm tunnel endpoint is alive:

```bash
nc -vz 127.0.0.1 5900
```

- Confirm service health as needed:

```bash
nc -vz <remote-host> 22
```

## Why this Works
- SSH auth is cryptographically bounded to keys, removing password brute-force and shared secret risk.
- VNC traffic is encrypted inside SSH instead of exposed to LAN/WAN.
- Public attack surface is reduced by not exposing VNC ports.

## Notes
- A prior SSH session like `ssh relux` confirming login only proves host-level SSH reachability.
- If public VNC still appears reachable unexpectedly, firewall rules and cloud security-group / reverse-proxy ACLs should be rechecked.
- For long-running sessions use background tunnel mode:

```bash
ssh -f -N -L 5900:localhost:5900 relux
```
