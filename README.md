# Neko and Ubuntu web desktop on a cloud VM

A documentation-first, multi-cloud tutorial for running:

- a shared Firefox session with [Neko](https://github.com/m1k1o/neko);
- a persistent Ubuntu GNOME desktop with
  [KasmVNC](https://github.com/kasmtech/KasmVNC); or
- both on one Ubuntu 24.04 VM.

Everything needed to reproduce the setup is readable in [GUIDE.md](GUIDE.md): cloud
networking, commands, configuration templates, validation scripts, security, costs,
operations, recovery, and the tested OCI ARM64 case study. Nothing is hidden behind a
generator or installer.

## Choose one mode

| Mode | Result | Public application ports | Starting size | Tutorial |
|---|---|---|---|---|
| Neko only | Shared synchronized browser with audio and controlled input | TCP 80/443/59000; UDP 59000 | 2 vCPU, 4-8 GiB | [Open](GUIDE.md#tutorials-neko-only-deployment) |
| Desktop only | Persistent Ubuntu GNOME desktop in a browser | TCP 80/443 | 2-4 vCPU, 8-12 GiB | [Open](GUIDE.md#tutorials-desktop-only-deployment) |
| Combined | Both services on one VM and two hostnames | TCP 80/443/59000; UDP 59000 | 4 vCPU, 12-16 GiB | [Open](GUIDE.md#tutorials-combined-deployment) |

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

The combined design was tested end to end on Ubuntu 24.04 ARM64 in OCI. Other provider
paths are grounded in official provider documentation and equivalent networking, not
claimed as separately executed deployments. Recheck prices, quotas, images, and regional
availability before provisioning.

## Repository contents

```text
README.md   quick start and mode choice
GUIDE.md    complete readable tutorial, templates, and reference
LICENSE     MIT license for this project material
.gitignore  prevents local secrets and rendered state from entering Git
```

Project material is licensed under the [MIT License](LICENSE). Neko, KasmVNC, Caddy,
Docker, Ubuntu, and cloud services retain their own licenses and terms.
