# Setup Notes

Setup-specific context for this NanoClaw install. See [README.md](README.md) for general docs.

## Environment

- **Host:** macOS 26.2 (Tahoe), Apple Silicon
- **Container runtime:** Apple Container v0.10.0 (`brew install container`)
- **Node.js:** 25.2.1 (via Homebrew)
- **VPN:** Mullvad (active during normal use, must be disconnected for container builds)

## Setup Progress

Completed:
- [x] Git remotes (origin = TG-Techie/nanoclaw, upstream = qwibitai/nanoclaw)
- [x] Bootstrap (Node.js, npm dependencies, native modules)
- [x] Apple Container conversion (merged `upstream/skill/apple-container`)
- [x] Container image built (`nanoclaw-agent:latest`)

Remaining (resume with `/setup please pickup at SETUP_NOTES.md`):
- [ ] Container runtime verification (test run of the built image)
- [ ] Claude authentication (`.env` — OAuth token or API key)
- [ ] Telegram channel setup (`/add-telegram`)
- [ ] Mount allowlist configuration
- [ ] Service installation and start (launchd)
- [ ] End-to-end verification

## Key State on Disk

| Path | Status | Notes |
|------|--------|-------|
| `nanoclaw-agent:latest` | Built | Container image, ready to test |
| `.env` | Does not exist | Needs Claude credentials (step 4 of setup skill) |
| `store/nanoclaw.db` | Empty | No registered groups yet |
| `store/auth/` | Does not exist | No channel auth yet |

## Channel Plan

- **Telegram** — primary channel (selected by user)
- Future: user intends to build a custom client channel

## Apple Container DNS Bug

The builder VM's default nameserver (`192.168.64.1`, the vmnet gateway) does not forward DNS. Two independent issues compound:

1. **Explicit DNS is always required** — The vmnet gateway doesn't forward DNS, so builds fail with `Temporary failure resolving` even on a clean system with no VPN and nothing on port 53.
2. **Commercial VPN blocks external DNS** — VPN firewall blocks outbound DNS to non-tunnel servers (e.g. `8.8.8.8`), so even with DNS configured, builds fail while VPN is connected.

Tested configurations:

| Builder `--dns` flag | resolv.conf | VPN | Result |
|---|---|---|---|
| None | Default (`192.168.64.1`) | Off | **Fails** |
| None | Patched to `8.8.8.8` | Off | **Works** |
| None | Patched to `8.8.8.8` | On | **Fails** |
| `--dns 8.8.8.8` | N/A (set by flag) | Off | **Works** |
| `--dns 8.8.8.8` | N/A (set by flag) | On | **Fails** |

**To build** (both conditions required):
```bash
# 1. Disconnect VPN
# 2. Configure builder DNS (pick one):
#    Option A: --dns flag (cleanest, requires builder restart)
container builder stop && container builder rm && container builder start --dns 8.8.8.8
#    Option B: Patch resolv.conf (no restart needed)
container exec buildkit /bin/sh -c 'echo "nameserver 8.8.8.8" > /etc/resolv.conf'
# 3. Build
./container/build.sh
# 4. Reconnect VPN when done
```

**For runtime containers** (may also need `--dns`):
```bash
container run --dns 8.8.8.8 ...
```

Tracked upstream:
- [apple/container#402](https://github.com/apple/container/issues/402) — Port 53 conflict with local DNS services
- [apple/container#656](https://github.com/apple/container/issues/656) — Build-time DNS failure
- [apple/container#1287](https://github.com/apple/container/issues/1287), [#1307](https://github.com/apple/container/issues/1307) — Related reports
