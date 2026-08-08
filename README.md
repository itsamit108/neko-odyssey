# Neko and Ubuntu web desktop on a cloud VM

A documentation-first, multi-cloud tutorial for running:

- a shared Firefox session with [Neko](https://github.com/m1k1o/neko);
- a persistent Ubuntu GNOME desktop with
  [KasmVNC](https://github.com/kasmtech/KasmVNC); or
- both on one Ubuntu 24.04 VM.

Everything needed to reproduce the setup is readable in [GUIDE.md](GUIDE.md): cloud
networking, commands, configuration templates, validation scripts, security, costs,
operations, recovery, and sanitized OCI end-to-end and Azure infrastructure-test case
studies. Nothing is hidden behind a generator or installer.

Current reviewed baseline (2026-08-08): Ubuntu 24.04 ARM64, current official Neko
Firefox image, KasmVNC 1.5.0, and Caddy. The generic templates still use reviewed
versions or digests. The live OCI combined deployment instead performs a controlled
internal update once per UTC month, with backup, health checks, and rollback.

## Verified live OCI deployment

The sanitized reference deployment currently uses:

| Part | Verified configuration |
|---|---|
| Compute | OCI Mumbai, non-Spot Ampere A1 Flex; 2 OCPUs and 12 GiB RAM |
| Storage | About 200 GB boot volume; complete Firefox profile persisted on the host |
| Neko | Modern mode, official Firefox image, 1368x768 at 60 FPS |
| Video | H.264/x264 ultrafast, zero latency, 4.5 Mbps, 2 threads, key interval 120 |
| WebRTC | ICE Lite; TCP and UDP mux on 59000 |
| Desktop | Persistent Ubuntu GNOME through KasmVNC 1.5.0 at 1368x768 and 60 FPS |
| Public edge | Caddy automatic HTTPS and explicit security headers; internal ports stay private |
| Runtime safety | `restart: unless-stopped`, 2 GiB shared memory, Docker logs limited to 10 MiB x 3 |
| Updates | Internal systemd maintenance once per UTC month, deferred while either service is active |
| VM shutdown | None; KasmVNC's idle disconnect does not stop the OCI VM |

The live updater checks daily but changes the system at most once per UTC month. It
updates normal Ubuntu packages and snaps, verifies the official KasmVNC ARM64 package,
updates Neko only after a Firefox-profile backup, validates both HTTPS services, and
rolls Neko back on failure. Ubuntu security updates continue through the standard daily
APT timer. It does not reboot automatically.

## Choose one mode

| Mode | Result | Public application ports | Starting size | Tutorial |
|---|---|---|---|---|
| Neko only | Shared synchronized browser with audio and controlled input | TCP 80/443/59000; UDP 59000 | 2 vCPU, 4-8 GiB | [Open](GUIDE.md#tutorials-neko-only-deployment) |
| Desktop only | Persistent Ubuntu GNOME desktop in a browser | TCP 80/443 | 2-4 vCPU, 8-12 GiB | [Open](GUIDE.md#tutorials-desktop-only-deployment) |
| Combined | Both services on one VM and two hostnames | TCP 80/443/59000; UDP 59000 | 4 vCPU, 12-16 GiB | [Open](GUIDE.md#tutorials-combined-deployment) |

These are recommended starting sizes. An Azure 2-vCPU/1-GiB Neko-only infrastructure
test used 4 GiB of persistent swap and `1024x576@20`; it is not demonstrated browser
session capacity, not the normal minimum, and not suitable for the desktop modes.

TCP 22 is separate key-only SSH administration and should be restricted to a trusted
source CIDR. Never publish Neko 8080, KasmVNC 8444, raw VNC 5901, or RDP 3389.

## Follow the guide

1. Choose and prepare [OCI, AWS, GCP, Azure, or another VPS](GUIDE.md#cloud-providers).
2. Follow exactly one mode tutorial from the table above.
3. Copy the labeled configuration blocks from the guide to their stated paths.
4. Complete the local, external-network, browser, and reboot checks.
5. Use the [operations](GUIDE.md#operations) and
   [reference](GUIDE.md#reference) sections for updates, backups, cleanup, and recovery.

The normal path is short; deeper explanations and the complete command-by-command
runbook remain in the same guide for lossless reference.

## Security and support

The graphical Linux account is locked and separate from the SSH administrator. Runtime
passwords, private keys, rendered cloud state, backups, and live endpoints must never
enter Git. Direct application passwords do not provide MFA; sensitive deployments
should add a compatible identity-aware proxy or VPN.

The combined design was tested end to end on Ubuntu 24.04 ARM64 in OCI and
maintenance-reverified on 2026-08-08. Neko-only was
also deployed and infrastructure-tested on Ubuntu 24.04 AMD64 in Azure, including
trusted TLS, the TCP/UDP boundary, cleanup, persistent swap, and reboot recovery. The
remaining provider/mode paths are grounded in official documentation and equivalent
networking, not claimed as separately executed deployments. Recheck prices, quotas,
images, and regional availability before provisioning.

## Repository contents

```text
README.md   quick start and mode choice
GUIDE.md    complete readable tutorial, templates, and reference
LICENSE     MIT license for this project material
.gitignore  prevents local secrets and rendered state from entering Git
```

Project material is licensed under the [MIT License](LICENSE). Neko, KasmVNC, Caddy,
Docker, Ubuntu, and cloud services retain their own licenses and terms.
