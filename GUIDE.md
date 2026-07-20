# Complete guide

This standalone, documentation-first guide covers Neko, an Ubuntu GNOME web desktop,
and a combined deployment on OCI, AWS, GCP, Azure, or a generic VPS. It includes all
copyable runtime templates and optional helper sources. Start with the [README](README.md)
for project orientation. The project is distributed under the [license](LICENSE).

**Contents**

- [Tutorials](#tutorials)
  - [Start here](#tutorials-start-here)
  - [Neko-only deployment](#tutorials-neko-only-deployment)
  - [Desktop-only deployment](#tutorials-desktop-only-deployment)
  - [Combined deployment](#tutorials-combined-deployment)
- [Cloud providers](#cloud-providers)
  - [OCI](#cloud-providers-oracle-cloud-infrastructure)
  - [AWS](#cloud-providers-amazon-web-services)
  - [GCP](#cloud-providers-google-cloud)
  - [Azure](#cloud-providers-microsoft-azure)
  - [Generic VPS](#cloud-providers-generic-vps-or-another-cloud)
- [Design](#design)
  - [Architecture](#design-architecture)
  - [Security model](#design-security-model)
  - [Sizing and cost](#design-sizing-and-cost)
- [Operations](#operations)
  - [Updates and rollback](#operations-update-and-roll-back)
  - [Backups](#operations-back-up-and-restore)
  - [Troubleshooting](#operations-troubleshooting)
- [Reference](#reference)
  - [Compatibility](#reference-compatibility-and-support-matrix)
  - [Complete runbook](#reference-complete-multi-cloud-neko-and-ubuntu-browser-desktop-reference)
  - [Copyable templates](#reference-copyable-deployment-templates)
  - [Optional helper scripts](#reference-optional-original-helper-scripts)
  - [Security policy](#reference-security-policy)
  - [Optional GitHub Actions workflow](#reference-optional-github-actions-workflow)

## Tutorials


### Tutorials: Start here

Use this sequence for a new installation:

1. [Choose one mode](#tutorials-start-1-choose-one-mode).
2. [Choose one provider](#tutorials-start-2-choose-one-provider).
3. [Deploy the selected mode](#tutorials-start-3-deploy-the-selected-mode).
4. [Verify the result](#tutorials-start-4-verify-the-result).

Do not merge the single-mode and combined instructions. Each tutorial and matching
template set in this guide is independently complete.

#### Tutorials: Start 1. Choose one mode

| Mode | Use it when | Public application ports | Suggested starting size | Tutorial and assets |
|---|---|---|---|---|
| Neko-only | You need one shared synchronized browser with Neko audio and controlled input | TCP 80, 443, 59000; UDP 59000 | 2 vCPU, 4-8 GiB | [Tutorial](#tutorials-neko-only-deployment) / [Neko templates](#reference-template-neko-only-runtime-files) |
| Desktop-only | You need a persistent Ubuntu GNOME desktop but not Neko | TCP 80, 443 | 2-4 vCPU, 8-12 GiB | [Tutorial](#tutorials-desktop-only-deployment) / [desktop templates](#reference-template-desktop-only-runtime-files) |
| Combined | You need both on one VM and accept shared resources and one failure domain | TCP 80, 443, 59000; UDP 59000 | 4 vCPU, 12-16 GiB | [Tutorial](#tutorials-combined-deployment) / [combined templates](#reference-template-combined-runtime-files) |

TCP 22 is separate key-only SSH administration and must be restricted to a trusted
administrator CIDR. The sizing figures are starting points, not provider free-tier
guarantees. Compare resource and architecture trade-offs in
[sizing and cost](#design-sizing-and-cost) and
[browser/CPU compatibility](#design-browser-and-cpu-architecture).

#### Tutorials: Start 2. Choose one provider

| Provider | Preparation guide |
|---|---|
| Oracle Cloud Infrastructure | [OCI](#cloud-providers-oracle-cloud-infrastructure) |
| Amazon Web Services | [AWS](#cloud-providers-amazon-web-services) |
| Google Cloud | [GCP](#cloud-providers-google-cloud) |
| Microsoft Azure | [Azure](#cloud-providers-microsoft-azure) |
| Another cloud or VPS | [Generic VPS](#cloud-providers-generic-vps-or-another-cloud) |

The provider guide establishes the public network, stable IPv4 address, firewall, DNS,
recovery access, and cost controls. The selected tutorial then configures Ubuntu and the
applications. The [cloud overview](#cloud-providers) contains the common port table.

Current pricing, free allowances, public-IP charges, storage, egress, and regional
capacity can change independently of this project. Recheck the official sources linked
from the provider page immediately before provisioning.

#### Tutorials: Start 3. Deploy the selected mode

Follow the chosen tutorial in order. Each covers:

1. the expected architecture and public port contract;
2. VM sizing, provider networking, a stable address, and DNS;
3. Ubuntu patching and Docker installation;
4. installation of only the selected mode's templates from this guide;
5. protected runtime secrets and credentials;
6. host services, firewall, TLS, and boot persistence; and
7. local, external, browser, and reboot acceptance tests.

Tutorial command labels have these meanings:

- **Local**: your trusted workstation before connecting to the VM.
- **Cloud**: the provider console, CLI, or infrastructure-as-code project.
- **VM**: an SSH shell on Ubuntu 24.04.
- **External**: another machine or network used to test the public boundary.

Local Bash blocks run in Linux, macOS, WSL, or Git Bash. Windows external checks use
PowerShell. VM blocks require Bash on Ubuntu.

Replace `PUBLIC_IP`, `NEKO_HOST`, `DESKTOP_HOST`, `ADMIN_PUBLIC_IP/32`,
`ADMIN_USER`, `PATH_TO_PRIVATE_KEY`, and `REPOSITORY_URL` with deployment values. The
documentation uses examples only; it contains no live endpoint or credential.

Use the [full runbook](#reference-complete-multi-cloud-neko-and-ubuntu-browser-desktop-reference) when you need exact cloud CLI flows,
existing-Docker migration, provider recovery, backup internals, or the deployment
inventory template. Its build sequence is the combined-mode superset; for either
single-service mode, the selected short tutorial controls which assets and ports apply.

#### Tutorials: Start 4. Verify the result

Check the reviewed four-file repository for whitespace errors before applying changes:

```bash
git diff --check
```

The selected tutorial then runs:

- [the preflight helper](#reference-helper-script-preflight) saved as `/usr/local/sbin/neko-cloud-preflight` before the first start;
- [the post-deployment helper](#reference-helper-script-post-deployment-validation) saved as `/usr/local/sbin/neko-cloud-validate` with `neko`, `desktop`, or `combined` after startup;
- external TCP checks for required and prohibited ports;
- real Neko media/control tests when Neko is enabled;
- real GNOME login, input, resize, persistence, and reboot tests when the desktop is
  enabled.

The intended boundary is:

- public: TCP 80/443, plus TCP/UDP 59000 only for Neko modes;
- administrator-only: TCP 22;
- never public: Neko 8080, KasmVNC 8444, raw VNC 5901, and RDP 3389.

Caddy is the only public HTTP/HTTPS listener. The `desktop` Linux account is locked,
separate from the SSH administrator, and outside sudo, Docker, LXD, and SSH access. On
OCI Ubuntu platform images, do not apply the generic UFW procedure; preserve Oracle's
required rules using the [OCI instructions](#cloud-providers-oracle-cloud-infrastructure).

Stop if validation contradicts any boundary. Use [troubleshooting](#operations-troubleshooting)
instead of opening an internal port.

#### Tutorials: Start: After deployment

| Task | Guide |
|---|---|
| Routine checks and browser access | [Operate](#operations-operate-the-deployment) |
| Controlled image/package changes | [Update and roll back](#operations-update-and-roll-back) |
| Neko and KasmVNC password changes | [Rotate credentials](#operations-rotate-credentials) |
| Encrypted backup and tested recovery | [Back up and restore](#operations-back-up-and-restore) |
| Safe host cleanup or provider teardown | [Clean up](#operations-safe-cleanup-and-cloud-teardown) |
| Failures and recovery paths | [Troubleshoot](#operations-troubleshooting) |

#### Tutorials: Start: Design, evidence, and reference

- [Architecture and trust boundaries](#design-architecture)
- [Security model and password-only trade-offs](#design-security-model)
- [Sizing, cost, and ARM64/AMD64](#design-sizing-and-cost)
- [Browser and CPU architecture choices](#design-browser-and-cpu-architecture)
- [Compatibility and pinned versions](#reference-compatibility-and-support-matrix)
- [Complete multi-cloud reference](#reference-complete-multi-cloud-neko-and-ubuntu-browser-desktop-reference)
- [Sanitized OCI ARM64 combined-mode case study](#reference-sanitized-oci-arm64-combined-deployment-case-study)
- [Sanitized Azure AMD64 Neko-only case study](#reference-sanitized-azure-amd64-neko-only-deployment-case-study)

The combined architecture was tested end to end on Ubuntu 24.04 ARM64 in OCI. Neko-only
was also deployed and infrastructure-tested on Ubuntu 24.04 AMD64 in Azure. Remaining
provider/mode paths are source-grounded equivalents, not claims of completed deployments
on every cloud. Read the repository [security policy](#reference-security-policy) before
publishing an endpoint and [contribution guide](#reference-contributing-and-validation)
before changing templates or version pins.

---

### Tutorials: Neko-only deployment

#### Tutorials: Neko 1. Outcome

Deploy one synchronized Firefox browser at `https://NEKO_HOST`: Neko runs in Docker,
Caddy terminates TLS, and WebRTC uses one UDP/TCP mux port. This mode does not install
GNOME, KasmVNC, XRDP, or a general-purpose desktop. Firefox runs **inside the VM**;
the viewing laptop may use Chrome, Edge, Firefox, or another compatible WebRTC client.
Use the [combined tutorial](#tutorials-combined-deployment) to add Ubuntu GNOME, and never mix files from
different mode directories.

```text
local browser
  |-- HTTPS 443 --------------------> Caddy
  |                                    `--> Neko on Docker network:8080
  `-- WebRTC UDP/TCP 59000 ---------> Neko

SSH administrator -- TCP 22 from one trusted CIDR only
```

#### Tutorials: Neko 2. Prerequisites

Start with:

- Ubuntu Server 24.04 LTS, AMD64 or ARM64;
- 2 vCPU and 4 GiB RAM minimum for light use;
- 2-4 vCPU and 8 GiB RAM recommended for video-heavy browsing;
- at least 50 GiB of encrypted persistent disk;
- one stable public IPv4 address;
- one DNS `A` record, such as `neko.example.com`;
- a trusted workstation with an SSH client and the VM's private key.

Real-time resolution, frame rate, video, and viewer count drive CPU and egress; the
supplied profile starts at 1280x720. Confirm provider-specific free-tier limits in
[sizing and cost](#design-sizing-and-cost), and check
[compatibility](#reference-compatibility-and-support-matrix) before changing the AMD64/ARM64 Firefox
image or version.

A 2-vCPU/1-GiB Azure VM was able to complete constrained Neko-only infrastructure tests
with 4 GiB of persistent swap, `shm_size: '1gb'`, and
`NEKO_DESKTOP_SCREEN: '1024x576@20'`. This is an infrastructure-test configuration
with active swapping and little headroom, not demonstrated browser-session capacity or
a replacement for the 4-GiB minimum above. Never use it for desktop-only, combined,
concurrent, or video-heavy workloads.

##### Tutorials: Neko: Provision the VM and network

**Context: Cloud console, cloud CLI, or infrastructure-as-code**

1. Create a dedicated network or place the VM in a public subnet with a route to an
   internet gateway.
2. Create an Ubuntu 24.04 VM with the size above.
3. Assign a stable/reserved public IPv4 address. A changing address breaks Neko's
   advertised WebRTC address, DNS, and certificates.
4. Create a dedicated instance firewall/security group with exactly the rules in
   section 3. Set source ports to `Any`; that table describes destination ports.
5. Attach that firewall/security group to the VM's primary network interface.
6. Confirm no broader subnet rule already exposes an internal port.
7. Add cost-budget and resource-usage alerts.

Use the provider-specific instructions for [OCI](#cloud-providers-oracle-cloud-infrastructure),
[AWS](#cloud-providers-amazon-web-services), [GCP](#cloud-providers-google-cloud), [Azure](#cloud-providers-microsoft-azure), or a
[generic VPS](#cloud-providers-generic-vps-or-another-cloud). On OCI, security lists and network security
groups are additive; a restrictive NSG cannot cancel a broad security-list rule.

#### Tutorials: Neko 3. Variables and ports

Use this public ingress contract:

| Protocol | Port | Source | Purpose |
| --- | ---: | --- | --- |
| TCP | 22 | `ADMIN_PUBLIC_IP/32` | Key-only SSH administration |
| TCP | 80 | Internet | ACME HTTP validation and HTTPS redirect |
| TCP | 443 | Internet | Neko web UI and signaling through Caddy |
| UDP | 59000 | Internet | Preferred Neko WebRTC media path |
| TCP | 59000 | Internet | Neko WebRTC fallback |

Do **not** publish TCP 8080. Also keep 3389, 5901, and 8444 closed. Cloud firewall,
host firewall, and application bindings are independent layers; verify all three.

##### Tutorials: Neko: DNS

**Context: Cloud or DNS provider**

Create this record:

```text
Type: A
Name: neko.example.com
Value: PUBLIC_IP
TTL: 300 (during setup)
```

Use a domain you control for a stable deployment. A public-IP-derived DNS service can
be useful for a disposable test, but it does not make an ephemeral address stable.
Remove stray `AAAA` records unless IPv6 is intentionally routed and firewalled.

**Context: Local workstation**

```bash
dig +short A neko.example.com
```

On Windows PowerShell, use
`Resolve-DnsName neko.example.com -Type A | Select-Object -ExpandProperty IPAddress`.

The result must be exactly the VM's public IPv4 address before Caddy starts certificate
issuance.

#### Tutorials: Neko 4. Install

##### Tutorials: Neko: Fetch and inspect the project

**Context: Local workstation**

```bash
git clone "REPOSITORY_URL" neko-cloud-guide
cd neko-cloud-guide
git rev-parse HEAD
git log -1 --show-signature --format=fuller -- GUIDE.md
```

Replace `REPOSITORY_URL`, prefer a signed release or reviewed commit, and record its
ID. Preflight checks installed templates; it neither provisions networking nor creates
secrets.

**Context: VM over SSH**

```bash
ssh -i "PATH_TO_PRIVATE_KEY" "ADMIN_USER@PUBLIC_IP"
```

Keep this SSH session open until a second key-only session succeeds after firewall and
SSH changes.

##### Tutorials: Neko: Prepare Ubuntu

**Context: VM**

Record the starting state, then patch and reboot:

```bash
set -euo pipefail
cat /etc/os-release
dpkg --print-architecture
free -h
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
ip route
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get dist-upgrade -y
sudo apt-get install -y ca-certificates curl git gnupg jq openssl unattended-upgrades
sudo reboot
```

Reconnect and confirm Ubuntu 24.04, time synchronization, free disk, and no failed
units:

```bash
timedatectl
df -hT
free -h
systemctl --failed --no-pager
```

Do not change cloud-init, netplan, or the provider agent merely to install Neko.

##### Tutorials: Neko: Install Docker Engine and Compose

**Context: VM**

First inventory any existing runtime. Stop and plan a migration if it already manages
workloads:

```bash
command -v docker || true
sudo docker ps -a 2>/dev/null || true
dpkg -l | grep -E 'docker|containerd|podman|runc' || true
```

On a fresh host, install Docker from Docker's official Ubuntu repository:

```bash
set -euo pipefail
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
. /etc/os-release
CODENAME=${UBUNTU_CODENAME:-$VERSION_CODENAME}
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $CODENAME stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker.service containerd.service
sudo docker run --rm hello-world
sudo docker compose version
```

Use `sudo docker`; membership in the `docker` group is root-equivalent. If Docker or a
custom `/etc/docker/daemon.json` already exists, use the inventory and merge procedure
in [full runbook sections 12-13](#reference-13-install-docker-engine-and-compose)
instead of overwriting it.

##### Tutorials: Neko: Install the Neko-mode assets

Use the same reviewed guide revision recorded earlier. Open the matching
[copyable template appendix](#reference-copyable-deployment-templates), then save each
shown block directly to its labeled runtime path. The commands below create and open
those paths; paste only the selected mode's blocks. If the VM cannot open the guide,
copy the reviewed text from the trusted workstation. Never mix mode templates.

**Context: VM**

```bash
set -euo pipefail
sudo install -d -m 0755 -o root -g root /opt/neko-cloud
sudoedit /opt/neko-cloud/compose.yaml
sudoedit /opt/neko-cloud/Caddyfile
sudoedit /opt/neko-cloud/.env
sudo chown root:root /opt/neko-cloud/compose.yaml /opt/neko-cloud/Caddyfile /opt/neko-cloud/.env
sudo chmod 0644 /opt/neko-cloud/compose.yaml /opt/neko-cloud/Caddyfile
sudo chmod 0600 /opt/neko-cloud/.env
```

#### Tutorials: Neko 5. Configure

##### Tutorials: Neko: Set the runtime variables and credentials

Generate two different random passwords on a trusted terminal and save them in a
password manager:

```bash
openssl rand -hex 24
openssl rand -hex 24
```

Edit the runtime environment:

```bash
sudoedit /opt/neko-cloud/.env
```

Set all four values and remove every placeholder:

```dotenv
PUBLIC_IP=PUBLIC_IP
NEKO_HOST=neko.example.com
NEKO_USER_PASSWORD=UNIQUE_REGULAR_USER_PASSWORD
NEKO_ADMIN_PASSWORD=DIFFERENT_UNIQUE_ADMIN_PASSWORD
```

Do not copy those example literals. Do not put the runtime `.env` in Git, shell
history, screenshots, chat, a public image, or an unencrypted backup. Confirm ownership
and mode without printing its contents:

```bash
sudo test "$(sudo stat -c '%a %U:%G' /opt/neko-cloud/.env)" = '600 root:root'
if sudo grep -Eq '=(PUBLIC_IP|UNIQUE_|DIFFERENT_|CHANGE_ME|replace-with)' \
  /opt/neko-cloud/.env; then
  echo 'STOP: resolve every placeholder before continuing' >&2
fi
sudo /usr/local/sbin/neko-cloud-preflight neko /opt/neko-cloud
```

Validate Compose and pull the pinned images. Do not start Caddy until the host-firewall
subsection below permits certificate validation:

```bash
set -euo pipefail
cd /opt/neko-cloud
sudo docker compose config --quiet
sudo docker compose pull
```

Treat the Neko template appendix as reviewed source and `/opt/neko-cloud` as the installed copy.
Never commit `/opt/neko-cloud/.env`.

##### Tutorials: Neko: Configure the host firewall

The provider firewall configured under Prerequisites remains the primary public
boundary. On ordinary Ubuntu images that use UFW, allow only this mode's ports.

**Context: VM, non-OCI images only**

```bash
set -euo pipefail
ADMIN_CIDR='ADMIN_PUBLIC_IP/32'
case "$ADMIN_CIDR" in *ADMIN*|'0.0.0.0/0') echo 'Set a trusted admin CIDR' >&2; exit 1;; esac
sudo apt-get install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from "$ADMIN_CIDR" to any port 22 proto tcp comment 'SSH admin'
sudo ufw allow 80/tcp comment 'Caddy ACME'
sudo ufw allow 443/tcp comment 'Caddy HTTPS'
sudo ufw allow 59000/tcp comment 'Neko WebRTC fallback'
sudo ufw allow 59000/udp comment 'Neko WebRTC media'
sudo ufw --force enable
sudo ufw status numbered
```

Before enabling UFW, keep the current SSH connection open and confirm a second SSH
connection works. Docker-published ports can bypass normal UFW chains, so never rely on
UFW instead of the cloud firewall.

**OCI exception:** Oracle warns that UFW can remove required rules from OCI Ubuntu
platform images. Do not run the block above on OCI. Follow the persistent iptables
procedure in the [OCI guide](#cloud-providers-oracle-cloud-infrastructure) and preserve Oracle's link-local/iSCSI
rules.

#### Tutorials: Neko 6. Start

**Context: VM**

```bash
set -euo pipefail
cd /opt/neko-cloud
sudo docker compose up -d
```

Start Caddy only after the cloud and host firewall rules above are effective. Then
continue immediately with the checks below.

#### Tutorials: Neko 7. Verify

##### Tutorials: Neko: Local checks on the VM

**Context: VM**

```bash
set -euo pipefail
cd /opt/neko-cloud
sudo docker compose ps
sudo docker compose logs --tail=100 neko caddy
sudo ss -lntup | grep -E ':(80|443|59000)\b'
sudo swapon --show
sudo docker inspect --format '{{.Name}} restarts={{.RestartCount}} oom={{.State.OOMKilled}}' \
  $(sudo docker compose ps -q)
sudo docker stats --no-stream

if curl -fsS --connect-timeout 2 http://127.0.0.1:8080/ >/dev/null; then
  echo 'ERROR: host port 8080 is unexpectedly published' >&2
  exit 1
fi
sudo docker compose exec caddy wget -qO- http://neko:8080/ >/dev/null
sudo /usr/local/sbin/neko-cloud-validate neko /opt/neko-cloud
```

Expected state:

- `neko` and `caddy` are up without restart loops;
- host TCP 80, TCP 443, and TCP/UDP 59000 are listening;
- host TCP 8080 is unreachable;
- Caddy can reach `neko:8080` on the private Compose network;
- the Caddy log shows successful certificate issuance for `NEKO_HOST`;
- no container reports an unexpected restart or OOM kill, and configured swap is active.

The validation command is mode-specific. Do not validate a Neko runtime with the
desktop or combined mode contract.

##### Tutorials: Neko: Test the public boundary

Run these checks from the local workstation or a second network, not from the VM.

**Context: External Windows PowerShell**

```powershell
$Ip = "PUBLIC_IP"
$HostName = "neko.example.com"
$Ports = 22,80,443,59000,5901,8080,8444,3389
foreach ($Port in $Ports) {
  $Open = Test-NetConnection -ComputerName $Ip -Port $Port -InformationLevel Quiet
  [pscustomobject]@{ Port = $Port; Open = $Open }
}
curl.exe -I "http://$HostName"
curl.exe -I "https://$HostName"
```

Expected TCP results from the administrator's IP:

- open: 22, 80, 443, 59000;
- closed or filtered: 5901, 8080, 8444, 3389;
- from any other source, 22 is also closed or filtered.

A TCP probe does not test UDP 59000. A real media session from another network is the
decisive test.

If `tcpdump` is already installed, one harmless datagram can prove that the cloud and
host firewalls deliver UDP 59000 to the VM. Start the capture on the VM:

```bash
sudo timeout 15 tcpdump -nn -i any 'udp dst port 59000' -c 1
```

Then send one packet from external Windows PowerShell:

```powershell
$Udp = [System.Net.Sockets.UdpClient]::new()
try {
  $Bytes = [Text.Encoding]::ASCII.GetBytes('neko-udp-path-check')
  [void]$Udp.Send($Bytes, $Bytes.Length, 'PUBLIC_IP', 59000)
} finally {
  $Udp.Dispose()
}
```

A captured packet proves transport reachability only. It does not prove ICE negotiation,
decoded media, audio, or browser control; the real cross-network Neko session below
remains mandatory.

##### Tutorials: Neko: Complete the functional acceptance test

**Context: External web browser**

1. Open `https://neko.example.com`; verify a publicly trusted certificate and no
   mixed-content warning.
2. Sign in with a display name and the regular-user password. Confirm picture and
   audio arrive.
3. Sign out, then sign in with the separate administrator password. Claim control and
   test mouse, keyboard, navigation, tabs, sound, and the intended clipboard policy.
   Seeing the stream does not automatically grant input: use Neko's mouse/lock control
   button and the administrator policy before diagnosing clicks as broken.
4. Connect a second viewer. Confirm both see the same synchronized browser session.
5. Test from a different internet connection. Confirm UDP media works; on a network
   that blocks UDP, verify TCP 59000 fallback.
6. Restart only the Neko container and confirm the browser profile is intentionally
   stateless unless you deliberately added an encrypted, backed-up profile volume.
7. Reboot the VM and repeat the HTTPS, login, control, and audio tests. Also require
   `systemctl --failed` to be empty, configured swap to appear in `swapon --show`, and
   Neko/Caddy to return without restart loops or OOM kills.

On a borrowed laptop, use a private window, never save the administrator password,
sign out, and close the window. Private mode reduces residue but cannot stop malware or
key logging.

#### Tutorials: Neko 8. Next steps

##### Tutorials: Neko: Operate and recover

**Context: VM**

```bash
cd /opt/neko-cloud
sudo docker compose ps
sudo docker compose logs --since=30m neko caddy
sudo docker compose restart neko
df -hT
free -h
```

For safe image updates, digest pinning, rollback, credential rotation, backups, and
teardown, use:

- [Operate the services](#operations-operate-the-deployment)
- [Update and roll back](#operations-update-and-roll-back)
- [Rotate credentials](#operations-rotate-credentials)
- [Back up and restore](#operations-back-up-and-restore)
- [Clean up or tear down](#operations-safe-cleanup-and-cloud-teardown)
- [Troubleshooting](#operations-troubleshooting)

The [full runbook](#reference-complete-multi-cloud-neko-and-ubuntu-browser-desktop-reference) is the deep reference for provider
network audits, Docker conflicts, TLS failures, WebRTC diagnosis, and recovery.

---

### Tutorials: Desktop-only deployment

#### Tutorials: Desktop 1. Outcome

Deploy one persistent Ubuntu GNOME desktop at `https://DESKTOP_HOST`: KasmVNC supplies
the virtual X11 display and browser client, while Caddy supplies TLS. This headless
cloud session runs GNOME/Yaru/Ubuntu Dock without a physical login screen; GDM and
Wayland stay disabled. It deploys neither Neko nor public VNC/RDP. Use the
[combined tutorial](#tutorials-combined-deployment) if you need both products.

```text
local browser -- HTTPS 443 --> Caddy container
                                 `--> private Unix socket
                                        `--> socat --> 127.0.0.1:8444 KasmVNC
                                                        `--> persistent GNOME session

SSH administrator -- TCP 22 from one trusted CIDR only
desktop service account -- no SSH, sudo, Docker, LXD, or Linux password login
```

#### Tutorials: Desktop 2. Prerequisites

Start with:

- Ubuntu Server 24.04 LTS, AMD64 or ARM64;
- 2 vCPU and 4 GiB RAM minimum for evaluation or one very light session;
- 4 vCPU and 12 GiB RAM recommended for a responsive browser and several GUI apps;
- at least 50 GiB of encrypted persistent disk;
- one stable public IPv4 address;
- one DNS `A` record, such as `desktop.example.com`;
- a trusted workstation with an SSH client and the VM's private key.

This normal Ubuntu GNOME appearance costs more memory and disk than XFCE and provides
one persistent exclusive session, not one desktop per person. It does not promise host
audio; use Neko for synchronized streaming audio. Before changing versions or CPU,
check [compatibility](#reference-compatibility-and-support-matrix),
[browser and CPU architecture](#design-browser-and-cpu-architecture), and
[sizing and cost](#design-sizing-and-cost).

##### Tutorials: Desktop: Provision the VM and network

**Context: Cloud console, cloud CLI, or infrastructure-as-code**

1. Create a public subnet with a route to an internet gateway.
2. Create the Ubuntu 24.04 VM with the size above.
3. Assign a stable/reserved public IPv4 address.
4. Create a dedicated instance firewall/security group with exactly the three rules in
   section 3. Source ports remain `Any`; restrict destination TCP 22 to the
   administrator's current public `/32`.
5. Attach the firewall/security group to the VM's primary network interface.
6. Check effective subnet rules as well; a broad subnet rule can defeat a narrow
   instance rule.
7. Enable cost and resource alerts, and identify serial-console or disk-recovery access
   before installing a desktop stack.

Use the provider-specific instructions for [OCI](#cloud-providers-oracle-cloud-infrastructure),
[AWS](#cloud-providers-amazon-web-services), [GCP](#cloud-providers-google-cloud), [Azure](#cloud-providers-microsoft-azure), or a
[generic VPS](#cloud-providers-generic-vps-or-another-cloud).

#### Tutorials: Desktop 3. Variables and ports

Use this public ingress contract:

| Protocol | Port | Source | Purpose |
| --- | ---: | --- | --- |
| TCP | 22 | `ADMIN_PUBLIC_IP/32` | Key-only SSH administration |
| TCP | 80 | Internet | ACME HTTP validation and HTTPS redirect |
| TCP | 443 | Internet | KasmVNC web client and WebSocket through Caddy |

Keep TCP/UDP 8444 private, and keep 3389, 5901, 8080, and 59000 closed. The KasmVNC
listener binds to loopback; Caddy reaches it through `/run/kasmvnc/kasm.sock`.

##### Tutorials: Desktop: DNS

**Context: Cloud or DNS provider**

```text
Type: A
Name: desktop.example.com
Value: PUBLIC_IP
TTL: 300 (during setup)
```

Remove an unintended `AAAA` record unless IPv6 is fully configured. Wait until public
DNS resolves to exactly the stable address.

**Context: Local workstation**

```bash
dig +short A desktop.example.com
```

On Windows PowerShell, use
`Resolve-DnsName desktop.example.com -Type A | Select-Object -ExpandProperty IPAddress`.

#### Tutorials: Desktop 4. Install

##### Tutorials: Desktop: Fetch and inspect the project

**Context: Local workstation**

```bash
git clone "REPOSITORY_URL" neko-cloud-guide
cd neko-cloud-guide
git rev-parse HEAD
git log -1 --show-signature --format=fuller -- GUIDE.md
```

Replace `REPOSITORY_URL`, review the selected revision, and retain its commit ID.

**Context: VM over SSH**

```bash
ssh -i "PATH_TO_PRIVATE_KEY" "ADMIN_USER@PUBLIC_IP"
```

Keep this connection open while changing access rules. Test a second SSH connection
before closing it.

##### Tutorials: Desktop: Patch Ubuntu and create the isolated desktop user

**Context: VM**

```bash
set -euo pipefail
cat /etc/os-release
dpkg --print-architecture
free -h
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get dist-upgrade -y
sudo apt-get install -y ca-certificates curl git gnupg jq openssl socat \
  dbus-x11 xauth x11-xserver-utils xdg-user-dirs unattended-upgrades
sudo reboot
```

Reconnect, then create a fresh account dedicated to the web desktop. Stop and inspect
instead of silently reusing a pre-existing account with the same name:

```bash
set -euo pipefail
DESKTOP_USER=desktop
DESKTOP_HOME=/home/desktop
if id "$DESKTOP_USER" >/dev/null 2>&1; then
  echo 'STOP: the desktop account already exists; verify it manually' >&2
  exit 1
fi
sudo adduser --disabled-password --gecos '' "$DESKTOP_USER"
sudo passwd -l "$DESKTOP_USER"
sudo chmod 0750 "$DESKTOP_HOME"
id "$DESKTOP_USER"
sudo -l -U "$DESKTOP_USER" 2>&1 || true
sudo test ! -s "$DESKTOP_HOME/.ssh/authorized_keys"
```

The account must have only its own primary group and must not be granted sudo, Docker,
LXD, or SSH keys. Its Linux password remains locked; KasmVNC authentication is separate.

Prevent direct SSH for the desktop account while retaining key-only administrator
access:

```bash
sudo install -d -m 0755 /etc/ssh/sshd_config.d
sudo tee /etc/ssh/sshd_config.d/40-cloud-desktop.conf >/dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
X11Forwarding no
DenyUsers desktop
EOF
sudo sshd -t
sudo systemctl reload ssh.service
```

Open a second administrator SSH session now. If it fails, restore the SSH configuration
from the first session before proceeding.

##### Tutorials: Desktop: Install Docker for Caddy

The desktop is host-native; Docker runs only Caddy. Inventory existing runtimes first:

**Context: VM**

```bash
command -v docker || true
sudo docker ps -a 2>/dev/null || true
dpkg -l | grep -E 'docker|containerd|podman|runc' || true
```

If the VM already runs containers, stop and merge this project into the existing
runtime deliberately. On a fresh VM:

```bash
set -euo pipefail
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
. /etc/os-release
CODENAME=${UBUNTU_CODENAME:-$VERSION_CODENAME}
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $CODENAME stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker.service containerd.service
sudo docker run --rm hello-world
sudo docker compose version
```

Never add either account to the root-equivalent `docker` group. Use the
[full Docker section](#reference-13-install-docker-engine-and-compose)
for conflicts and log rotation.

##### Tutorials: Desktop: Install Ubuntu GNOME without a display manager

Take a cloud snapshot first if the VM contains anything important.

**Context: VM**

```bash
set -euo pipefail
sudo systemctl set-default multi-user.target
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y \
  ubuntu-desktop-minimal ubuntu-session gnome-shell gnome-terminal nautilus \
  yaru-theme-gtk yaru-theme-icon dbus-x11 x11-xserver-utils xauth xdg-user-dirs \
  gnome-remote-desktop
sudo systemctl disable --now gdm3.service 2>/dev/null || true
sudo systemctl mask gdm3.service
systemctl get-default
systemctl is-enabled gdm3.service || true
```

Expected: default target `multi-user.target`; GDM masked. KasmVNC supplies the virtual
X11 display, so a physical greeter and Wayland session are unnecessary. Do not alter
netplan because desktop packages installed NetworkManager components.

##### Tutorials: Desktop: Install the verified KasmVNC package

This project pins KasmVNC 1.4.0 for Ubuntu 24.04 Noble and verifies the release digest
for each supported architecture.

**Context: VM**

```bash
set -euo pipefail
ARCH=$(dpkg --print-architecture)
case "$ARCH" in
  arm64)
    KASM_URL='https://github.com/kasmtech/KasmVNC/releases/download/v1.4.0/kasmvncserver_noble_1.4.0_arm64.deb'
    KASM_SHA256='120d9462cb5e917cad91a23f6cb0b780c06f701def40e900b29f996979200638'
    ;;
  amd64)
    KASM_URL='https://github.com/kasmtech/KasmVNC/releases/download/v1.4.0/kasmvncserver_noble_1.4.0_amd64.deb'
    KASM_SHA256='12bac6014149c5fdee75f0d403785aaa3e5dd4ea222de73253a5d4181bc9567e'
    ;;
  *) echo "Unsupported architecture: $ARCH" >&2; exit 1 ;;
esac
KASM_DEB=$(mktemp --suffix=.deb)
trap 'rm -f -- "$KASM_DEB"' EXIT HUP INT TERM
curl -fL "$KASM_URL" -o "$KASM_DEB"
printf '%s  %s\n' "$KASM_SHA256" "$KASM_DEB" | sha256sum -c -
sudo apt-get install -y "$KASM_DEB"
rm -f -- "$KASM_DEB"
trap - EXIT HUP INT TERM
dpkg-query -W kasmvncserver
```

If the pinned release is later changed, obtain both package and digest from the
official release and update [compatibility](#reference-compatibility-and-support-matrix) in the same
review. Do not install an unverified package copied from an unrelated tutorial.

##### Tutorials: Desktop: Install the desktop-mode assets

Use the same reviewed guide revision recorded earlier. Open the matching
[copyable template appendix](#reference-copyable-deployment-templates), then save each
shown block directly to its labeled runtime path. The commands below create and open
those paths; paste only the selected mode's blocks. If the VM cannot open the guide,
copy the reviewed text from the trusted workstation. Never mix mode templates.

**Context: VM**

```bash
set -euo pipefail
sudo install -d -m 0755 -o root -g root /opt/neko-cloud
sudoedit /opt/neko-cloud/compose.yaml
sudoedit /opt/neko-cloud/Caddyfile
sudoedit /opt/neko-cloud/.env
sudo chown root:root /opt/neko-cloud/compose.yaml /opt/neko-cloud/Caddyfile /opt/neko-cloud/.env
sudo chmod 0644 /opt/neko-cloud/compose.yaml /opt/neko-cloud/Caddyfile
sudo chmod 0600 /opt/neko-cloud/.env

sudo install -d -m 0700 -o desktop -g desktop /home/desktop/.vnc
sudo install -d -m 0700 -o desktop -g desktop /home/desktop/.config/systemd/user
sudoedit /home/desktop/.vnc/kasmvnc.yaml
sudoedit /home/desktop/.vnc/xstartup
sudoedit /home/desktop/.config/systemd/user/kasmvncserver@.service
sudoedit /etc/systemd/system/kasmvnc-socat.service
sudo chown desktop:desktop /home/desktop/.vnc/kasmvnc.yaml /home/desktop/.vnc/xstartup /home/desktop/.config/systemd/user/kasmvncserver@.service
sudo chmod 0600 /home/desktop/.vnc/kasmvnc.yaml
sudo chmod 0755 /home/desktop/.vnc/xstartup
sudo chmod 0644 /home/desktop/.config/systemd/user/kasmvncserver@.service
sudo chown root:root /etc/systemd/system/kasmvnc-socat.service
sudo chmod 0644 /etc/systemd/system/kasmvnc-socat.service
```

#### Tutorials: Desktop 5. Configure

##### Tutorials: Desktop: Set the runtime variables

Edit the runtime environment:

```bash
sudoedit /opt/neko-cloud/.env
```

Set both values and remove every placeholder:

```dotenv
PUBLIC_IP=PUBLIC_IP
DESKTOP_HOST=desktop.example.com
```

Then validate it without printing the file:

```bash
sudo test "$(sudo stat -c '%a %U:%G' /opt/neko-cloud/.env)" = '600 root:root'
if sudo grep -Eq '=(PUBLIC_IP|CHANGE_ME|replace-with)' /opt/neko-cloud/.env; then
  echo 'STOP: resolve every placeholder before continuing' >&2
fi
sudo /usr/local/sbin/neko-cloud-preflight desktop /opt/neko-cloud
```

The desktop template appendix uses service account `desktop`, home `/home/desktop`,
and portable systemd `%t` rather than a numeric UID.

##### Tutorials: Desktop: Create KasmVNC identity and credentials

Create a private placeholder certificate for KasmVNC's loopback endpoint. Public TLS
still terminates at Caddy.

**Context: VM**

```bash
sudo -iu desktop bash <<'EOF'
set -euo pipefail
install -d -m 0700 "$HOME/.vnc"
openssl req -x509 -nodes -newkey rsa:2048 -days 3650 \
  -subj '/CN=localhost' \
  -keyout "$HOME/.vnc/kasmvnc.key" \
  -out "$HOME/.vnc/kasmvnc.crt"
chmod 0600 "$HOME/.vnc/kasmvnc.key" "$HOME/.vnc/kasmvnc.crt"
EOF
```

Now enter the desktop user's shell and create one write-capable Kasm login. Choose a
new login name and a unique password when prompted:

```bash
sudo -iu desktop
export XDG_RUNTIME_DIR=/run/user/$(id -u)
export DBUS_SESSION_BUS_ADDRESS=unix:path=$XDG_RUNTIME_DIR/bus
KASM_LOGIN='CHOOSE_A_KASM_LOGIN_NAME'
vncpasswd -u "$KASM_LOGIN" -w
chmod 0600 "$HOME/.kasmpasswd"
exit
```

`-w` grants keyboard and mouse control. Do not add owner permission unless web-based
Kasm user management is intentionally required. `.kasmpasswd` is obfuscated rather
than strongly encrypted, so treat it like plaintext: never commit or casually back it
up.

##### Tutorials: Desktop: Configure the persistent GNOME session

Enable a lingering user manager so the desktop survives SSH logout and starts at boot:

**Context: VM**

```bash
set -euo pipefail
DESKTOP_UID=$(id -u desktop)
sudo loginctl enable-linger desktop
sudo systemctl start "user@$DESKTOP_UID.service"
sudo -u desktop env \
  HOME=/home/desktop USER=desktop LOGNAME=desktop \
  XDG_RUNTIME_DIR="/run/user/$DESKTOP_UID" \
  DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$DESKTOP_UID/bus" \
  systemctl --user daemon-reload
sudo -u desktop env \
  HOME=/home/desktop USER=desktop LOGNAME=desktop \
  XDG_RUNTIME_DIR="/run/user/$DESKTOP_UID" \
  DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$DESKTOP_UID/bus" \
  systemctl --user enable --now 'kasmvncserver@:1.service'
```

For this headless, non-sudo session, disable the idle lock and suspend that otherwise
leave a persistent remote desktop unusable:

```bash
DESKTOP_UID=$(id -u desktop)
sudo -u desktop env \
  HOME=/home/desktop USER=desktop LOGNAME=desktop \
  XDG_RUNTIME_DIR="/run/user/$DESKTOP_UID" \
  DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$DESKTOP_UID/bus" \
  gsettings set org.gnome.desktop.session idle-delay 'uint32 0'
sudo -u desktop env \
  HOME=/home/desktop USER=desktop LOGNAME=desktop \
  XDG_RUNTIME_DIR="/run/user/$DESKTOP_UID" \
  DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$DESKTOP_UID/bus" \
  gsettings set org.gnome.desktop.screensaver lock-enabled false
sudo -u desktop env \
  HOME=/home/desktop USER=desktop LOGNAME=desktop \
  XDG_RUNTIME_DIR="/run/user/$DESKTOP_UID" \
  DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$DESKTOP_UID/bus" \
  gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
```

The tested Ubuntu 24.04 GNOME session also keeps the system handover service active;
this does **not** authorize RDP:

```bash
sudo systemctl unmask gnome-remote-desktop.service
sudo systemctl enable --now gnome-remote-desktop.service
if sudo ss -lntup | grep -qE ':(3389)\b'; then
  echo 'ERROR: unexpected RDP listener; investigate before continuing' >&2
  exit 1
fi
```

Do not set GNOME RDP credentials and do not open TCP 3389.

#### Tutorials: Desktop 6. Start

##### Tutorials: Desktop: Start the private desktop bridge and prepare Caddy

**Context: VM**

```bash
set -euo pipefail
sudo systemd-analyze verify /etc/systemd/system/kasmvnc-socat.service
sudo systemctl daemon-reload
sudo systemctl enable --now kasmvnc-socat.service
sudo systemctl status kasmvnc-socat.service --no-pager
sudo stat -c '%a %U:%G %F %n' /run/kasmvnc/kasm.sock
sudo curl --unix-socket /run/kasmvnc/kasm.sock -I http://localhost/

cd /opt/neko-cloud
sudo docker compose config --quiet
sudo docker compose pull
```

The Unix-socket curl should return HTTP `401 Unauthorized`; that is success because
KasmVNC is reachable and demanding its own credential. `Connection refused` means the
desktop service is down. Never make port 8444 public to fix a bridge problem. Caddy is
pulled but deliberately not started until the host firewall is ready.

##### Tutorials: Desktop: Apply the host firewall before Caddy

**Context: VM, non-OCI images only**

```bash
set -euo pipefail
ADMIN_CIDR='ADMIN_PUBLIC_IP/32'
case "$ADMIN_CIDR" in *ADMIN*|'0.0.0.0/0') echo 'Set a trusted admin CIDR' >&2; exit 1;; esac
sudo apt-get install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from "$ADMIN_CIDR" to any port 22 proto tcp comment 'SSH admin'
sudo ufw allow 80/tcp comment 'Caddy ACME'
sudo ufw allow 443/tcp comment 'Caddy HTTPS'
sudo ufw --force enable
sudo ufw status numbered
```

Keep the current SSH session open and test a second before enabling UFW. Do not add
59000, 8080, 8444, 5901, or 3389 for desktop-only mode. The cloud firewall remains
mandatory because Docker and UFW interact at different chains.

**OCI exception:** do not use UFW on OCI Ubuntu platform images. Follow the
[OCI host-firewall instructions](#cloud-providers-oracle-cloud-infrastructure) without flushing Oracle's required
link-local/iSCSI rules.

##### Tutorials: Desktop: Start Caddy

**Context: VM**

```bash
set -euo pipefail
cd /opt/neko-cloud
sudo docker compose up -d
```

#### Tutorials: Desktop 7. Verify

##### Tutorials: Desktop: Local checks on the VM

**Context: VM**

```bash
set -euo pipefail
cd /opt/neko-cloud
DESKTOP_UID=$(id -u desktop)
systemctl is-active docker.service kasmvnc-socat.service gnome-remote-desktop.service
sudo -u desktop env \
  HOME=/home/desktop USER=desktop LOGNAME=desktop \
  XDG_RUNTIME_DIR="/run/user/$DESKTOP_UID" \
  DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$DESKTOP_UID/bus" \
  systemctl --user is-active 'kasmvncserver@:1.service'
sudo ss -lntup
sudo ss -lnt | grep '127.0.0.1:8444'
sudo test -S /run/kasmvnc/kasm.sock
sudo docker compose ps
sudo docker compose logs --tail=100 caddy
sudo /usr/local/sbin/neko-cloud-validate desktop /opt/neko-cloud
```

Expected state:

- KasmVNC, the bridge, Caddy, and the GNOME handover service are active;
- KasmVNC TCP 8444 binds only to `127.0.0.1`;
- `/run/kasmvnc/kasm.sock` exists;
- Caddy is up and has a trusted certificate for `DESKTOP_HOST`;
- there is no public/raw listener on 3389 or 5901.

The validation command is mode-specific. Do not validate this runtime with the Neko or
combined mode contract.

##### Tutorials: Desktop: Test the public boundary

**Context: External Windows PowerShell**

```powershell
$Ip = "PUBLIC_IP"
$HostName = "desktop.example.com"
$Ports = 22,80,443,59000,5901,8080,8444,3389
foreach ($Port in $Ports) {
  $Open = Test-NetConnection -ComputerName $Ip -Port $Port -InformationLevel Quiet
  [pscustomobject]@{ Port = $Port; Open = $Open }
}
curl.exe -I "http://$HostName"
curl.exe -I "https://$HostName"
```

Expected TCP results from the administrator's IP:

- open: 22, 80, 443;
- closed or filtered: 59000, 5901, 8080, 8444, 3389;
- from any other source, 22 is also closed or filtered;
- HTTPS initially returns `401`, proving KasmVNC authentication is present.

##### Tutorials: Desktop: Complete the functional acceptance test

**Context: External web browser**

1. Open `https://desktop.example.com` in Chrome, Edge, or another supported client.
2. Verify the certificate is publicly trusted, then enter the Kasm login and password.
3. Confirm the Ubuntu Dock, Activities overview, Files, Settings, and Terminal look and
   behave like Ubuntu GNOME rather than XFCE.
4. Test keyboard, mouse, clipboard policy, full screen, and display resize.
5. Open an application, close the local browser tab without signing out, reconnect,
   and confirm the application remains open.
6. Connect from a second browser. It should replace the old exclusive viewer rather
   than create another GNOME session.
7. Confirm desktop audio is not an acceptance requirement for this standalone path.
8. Reboot the VM. Confirm all units return and repeat the HTTPS, login, persistence,
   input, and resize tests.

On a borrowed laptop, use a private window, never save the credential, sign out, and
close it. Private browsing cannot prevent an untrusted endpoint from recording input.

#### Tutorials: Desktop 8. Next steps

##### Tutorials: Desktop: Operate and recover

**Context: VM**

```bash
DESKTOP_UID=$(id -u desktop)
sudo -u desktop env \
  HOME=/home/desktop USER=desktop LOGNAME=desktop \
  XDG_RUNTIME_DIR="/run/user/$DESKTOP_UID" \
  DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$DESKTOP_UID/bus" \
  systemctl --user status 'kasmvncserver@:1.service' --no-pager
sudo journalctl -u kasmvnc-socat.service -n 100 --no-pager
cd /opt/neko-cloud && sudo docker compose logs --since=30m caddy
df -hT
free -h
```

Continue with [daily operation](#operations-operate-the-deployment),
[updates and rollback](#operations-update-and-roll-back),
[credential rotation](#operations-rotate-credentials),
[backup and restore](#operations-back-up-and-restore),
[cleanup](#operations-safe-cleanup-and-cloud-teardown), or
[troubleshooting](#operations-troubleshooting). The
[full runbook](#reference-complete-multi-cloud-neko-and-ubuntu-browser-desktop-reference) contains detailed GNOME, D-Bus, KasmVNC,
socket-bridge, TLS, and recovery diagnostics.

---

### Tutorials: Combined deployment

#### Tutorials: Combined 1. Outcome

Deploy two independent browser endpoints on one VM:

- `https://NEKO_HOST` for a synchronized Firefox session with Neko audio/video;
- `https://DESKTOP_HOST` for a persistent Ubuntu GNOME desktop through KasmVNC.

One Caddy container terminates TLS for both names. Neko stays on its private Docker
network except for WebRTC port 59000; KasmVNC stays on host loopback and reaches Caddy
through a private Unix socket. Neko streams Firefox, while the viewing laptop may use
Chrome, Edge, Firefox, or another compatible WebRTC client. Choose [Neko only](#tutorials-neko-only-deployment)
or [desktop only](#tutorials-desktop-only-deployment) when appropriate. For this mode, install only
the combined templates into `/opt/neko-cloud`; deployment modes are mutually exclusive.

```text
                         +--> Neko container:8080 (private Docker network)
browser -- HTTPS 443 --> Caddy
                         `--> /run/kasmvnc/kasm.sock
                                `--> socat --> 127.0.0.1:8444 KasmVNC --> GNOME

browser -- UDP/TCP 59000 -----------------------------> Neko WebRTC
administrator -- TCP 22 from trusted CIDR only ------> SSH
```

#### Tutorials: Combined 2. Prerequisites

Start with:

- Ubuntu Server 24.04 LTS, AMD64 or ARM64;
- 2 vCPU and 8 GiB RAM as a constrained evaluation minimum;
- 4 vCPU and 16 GiB RAM recommended for simultaneous Neko video and GNOME work;
- at least 50 GiB encrypted persistent disk, preferably 80 GiB with snapshots and GUI apps;
- one stable public IPv4 address;
- two distinct DNS `A` records pointing to that address;
- a trusted workstation, an SSH client, and the VM's private key.

Colocation can be cheaper and simpler, but creates one failure, maintenance, and
resource-contention domain; use two VMs for different trust groups, independent
maintenance, stronger isolation, or predictable performance. Firefox and pinned
KasmVNC support AMD64 and ARM64, but the
Neko Chrome image is not an ARM64 replacement. Review
[compatibility](#reference-compatibility-and-support-matrix),
[browser and CPU architecture](#design-browser-and-cpu-architecture), and
[sizing and cost](#design-sizing-and-cost).

##### Tutorials: Combined: Provision the cloud VM and network

**Context: Cloud console, cloud CLI, or infrastructure-as-code**

1. Create a dedicated public network/subnet with a route to an internet gateway.
2. Create the Ubuntu 24.04 VM at the chosen size.
3. Assign a stable/reserved public IPv4 address.
4. Create an instance firewall/security group with exactly the combined-mode rules in
   section 3. Source ports remain `Any`; only destination ports are fixed.
5. Attach it to the VM's primary network interface.
6. Audit effective subnet and instance rules. Remove broad rules for internal ports;
   an additional restrictive rule does not override an existing allow rule.
7. Configure cost, CPU, memory, disk, and egress alerts.
8. Identify serial-console or disk-recovery access before installing GNOME.

Follow the cloud-specific page for [OCI](#cloud-providers-oracle-cloud-infrastructure), [AWS](#cloud-providers-amazon-web-services),
[GCP](#cloud-providers-google-cloud), [Azure](#cloud-providers-microsoft-azure), or a
[generic VPS](#cloud-providers-generic-vps-or-another-cloud). Free quotas and stopped-instance charges differ;
Ubuntu's lack of an OS licence fee does not make the infrastructure free.

#### Tutorials: Combined 3. Variables and ports

Use this public ingress contract:

| Protocol | Port | Source | Purpose |
| --- | ---: | --- | --- |
| TCP | 22 | `ADMIN_PUBLIC_IP/32` | Key-only SSH administration |
| TCP | 80 | Internet | ACME HTTP validation and HTTPS redirect |
| TCP | 443 | Internet | Both web clients and WebSockets through Caddy |
| UDP | 59000 | Internet | Preferred Neko WebRTC media path |
| TCP | 59000 | Internet | Neko WebRTC fallback |

Keep 8080, 8444, 5901, and 3389 closed. KasmVNC does not need a public VNC/RDP/media
port. Neko cannot work reliably through the reverse proxy alone because its media path
uses port 59000 directly.

##### Tutorials: Combined: DNS names

**Context: Cloud or DNS provider**

```text
Type: A                         Type: A
Name: neko.example.com         Name: desktop.example.com
Value: PUBLIC_IP               Value: PUBLIC_IP
TTL: 300                       TTL: 300
```

Use two distinct hostnames. Remove unintended `AAAA` records unless IPv6 is fully
routed and firewalled.

**Context: Local workstation**

```bash
dig +short A neko.example.com
dig +short A desktop.example.com
```

On Windows PowerShell, run `Resolve-DnsName HOSTNAME -Type A` once for each hostname.

Each result must be exactly the stable public IPv4 address before Caddy starts.

#### Tutorials: Combined 4. Install

##### Tutorials: Combined: Fetch and inspect the project

**Context: Local workstation**

```bash
git clone "REPOSITORY_URL" neko-cloud-guide
cd neko-cloud-guide
git rev-parse HEAD
git log -1 --show-signature --format=fuller -- GUIDE.md
```

Replace `REPOSITORY_URL`. Use a reviewed commit or release and record its ID.

**Context: VM over SSH**

```bash
ssh -i "PATH_TO_PRIVATE_KEY" "ADMIN_USER@PUBLIC_IP"
```

Keep this initial connection open until a second key-only session works after all
firewall and SSH changes.

##### Tutorials: Combined: Prepare Ubuntu and the isolated desktop account

**Context: VM**

```bash
set -euo pipefail
cat /etc/os-release
dpkg --print-architecture
free -h
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
ip route
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get dist-upgrade -y
sudo apt-get install -y ca-certificates curl git gnupg jq openssl socat \
  dbus-x11 xauth x11-xserver-utils xdg-user-dirs unattended-upgrades
sudo reboot
```

Reconnect and verify host health:

```bash
timedatectl
df -hT
free -h
systemctl --failed --no-pager
```

Create a locked, non-sudo account for the internet-controlled graphical session. On a
fresh build, abort instead of adopting a pre-existing same-named account:

```bash
set -euo pipefail
DESKTOP_USER=desktop
DESKTOP_HOME=/home/desktop
if id "$DESKTOP_USER" >/dev/null 2>&1; then
  echo 'STOP: the desktop account already exists; verify it manually' >&2
  exit 1
fi
sudo adduser --disabled-password --gecos '' "$DESKTOP_USER"
sudo passwd -l "$DESKTOP_USER"
sudo chmod 0750 "$DESKTOP_HOME"
id "$DESKTOP_USER"
sudo -l -U "$DESKTOP_USER" 2>&1 || true
sudo test ! -s "$DESKTOP_HOME/.ssh/authorized_keys"
```

Do not add `desktop` to `sudo`, `docker`, `lxd`, `adm`, or an SSH authorization file.
Block it from SSH explicitly:

```bash
sudo install -d -m 0755 /etc/ssh/sshd_config.d
sudo tee /etc/ssh/sshd_config.d/40-cloud-desktop.conf >/dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
X11Forwarding no
DenyUsers desktop
EOF
sudo sshd -t
sudo systemctl reload ssh.service
```

Open and verify a second administrator SSH session before continuing.

##### Tutorials: Combined: Install Docker Engine and Compose

Inventory the host first. Do not overwrite an existing container platform:

**Context: VM**

```bash
command -v docker || true
sudo docker ps -a 2>/dev/null || true
dpkg -l | grep -E 'docker|containerd|podman|runc' || true
```

For a fresh host, install from Docker's official Ubuntu repository:

```bash
set -euo pipefail
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
. /etc/os-release
CODENAME=${UBUNTU_CODENAME:-$VERSION_CODENAME}
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $CODENAME stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker.service containerd.service
sudo docker run --rm hello-world
sudo docker compose version
```

Use `sudo docker`; its group is root-equivalent. If a runtime or
`/etc/docker/daemon.json` exists, follow the
[full Docker section](#reference-13-install-docker-engine-and-compose).

##### Tutorials: Combined: Install Ubuntu GNOME and KasmVNC

Take a provider snapshot if the VM is not disposable. Install GNOME without a physical
display manager:

**Context: VM**

```bash
set -euo pipefail
sudo systemctl set-default multi-user.target
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y \
  ubuntu-desktop-minimal ubuntu-session gnome-shell gnome-terminal nautilus \
  yaru-theme-gtk yaru-theme-icon dbus-x11 x11-xserver-utils xauth xdg-user-dirs \
  gnome-remote-desktop
sudo systemctl disable --now gdm3.service 2>/dev/null || true
sudo systemctl mask gdm3.service
systemctl get-default
```

Expected: `multi-user.target`, with GDM masked. Do not modify netplan merely because
desktop packages installed NetworkManager components.

Install and verify the pinned KasmVNC 1.4.0 Noble package:

```bash
set -euo pipefail
ARCH=$(dpkg --print-architecture)
case "$ARCH" in
  arm64)
    KASM_URL='https://github.com/kasmtech/KasmVNC/releases/download/v1.4.0/kasmvncserver_noble_1.4.0_arm64.deb'
    KASM_SHA256='120d9462cb5e917cad91a23f6cb0b780c06f701def40e900b29f996979200638'
    ;;
  amd64)
    KASM_URL='https://github.com/kasmtech/KasmVNC/releases/download/v1.4.0/kasmvncserver_noble_1.4.0_amd64.deb'
    KASM_SHA256='12bac6014149c5fdee75f0d403785aaa3e5dd4ea222de73253a5d4181bc9567e'
    ;;
  *) echo "Unsupported architecture: $ARCH" >&2; exit 1 ;;
esac
KASM_DEB=$(mktemp --suffix=.deb)
trap 'rm -f -- "$KASM_DEB"' EXIT HUP INT TERM
curl -fL "$KASM_URL" -o "$KASM_DEB"
printf '%s  %s\n' "$KASM_SHA256" "$KASM_DEB" | sha256sum -c -
sudo apt-get install -y "$KASM_DEB"
rm -f -- "$KASM_DEB"
trap - EXIT HUP INT TERM
dpkg-query -W kasmvncserver
```

##### Tutorials: Combined: Install the combined-mode assets

Use the same reviewed guide revision recorded earlier. Open the matching
[copyable template appendix](#reference-copyable-deployment-templates), then save each
shown block directly to its labeled runtime path. The commands below create and open
those paths; paste only the selected mode's blocks. If the VM cannot open the guide,
copy the reviewed text from the trusted workstation. Never mix mode templates.

**Context: VM**

```bash
set -euo pipefail
sudo install -d -m 0755 -o root -g root /opt/neko-cloud
sudoedit /opt/neko-cloud/compose.yaml
sudoedit /opt/neko-cloud/Caddyfile
sudoedit /opt/neko-cloud/.env
sudo chown root:root /opt/neko-cloud/compose.yaml /opt/neko-cloud/Caddyfile /opt/neko-cloud/.env
sudo chmod 0644 /opt/neko-cloud/compose.yaml /opt/neko-cloud/Caddyfile
sudo chmod 0600 /opt/neko-cloud/.env

sudo install -d -m 0700 -o desktop -g desktop /home/desktop/.vnc
sudo install -d -m 0700 -o desktop -g desktop /home/desktop/.config/systemd/user
sudoedit /home/desktop/.vnc/kasmvnc.yaml
sudoedit /home/desktop/.vnc/xstartup
sudoedit /home/desktop/.config/systemd/user/kasmvncserver@.service
sudoedit /etc/systemd/system/kasmvnc-socat.service
sudo chown desktop:desktop /home/desktop/.vnc/kasmvnc.yaml /home/desktop/.vnc/xstartup /home/desktop/.config/systemd/user/kasmvncserver@.service
sudo chmod 0600 /home/desktop/.vnc/kasmvnc.yaml
sudo chmod 0755 /home/desktop/.vnc/xstartup
sudo chmod 0644 /home/desktop/.config/systemd/user/kasmvncserver@.service
sudo chown root:root /etc/systemd/system/kasmvnc-socat.service
sudo chmod 0644 /etc/systemd/system/kasmvnc-socat.service
```

#### Tutorials: Combined 5. Configure

##### Tutorials: Combined: Set the runtime variables and credentials

Generate two different Neko passwords on a trusted terminal and save them in a password
manager:

```bash
openssl rand -hex 24
openssl rand -hex 24
```

Edit `/opt/neko-cloud/.env`:

```bash
sudoedit /opt/neko-cloud/.env
```

Set all five values and remove every placeholder:

```dotenv
PUBLIC_IP=PUBLIC_IP
NEKO_HOST=neko.example.com
DESKTOP_HOST=desktop.example.com
NEKO_USER_PASSWORD=UNIQUE_REGULAR_USER_PASSWORD
NEKO_ADMIN_PASSWORD=DIFFERENT_UNIQUE_ADMIN_PASSWORD
```

Protect and check the file without printing its secrets:

```bash
sudo test "$(sudo stat -c '%a %U:%G' /opt/neko-cloud/.env)" = '600 root:root'
if sudo grep -Eq '=(PUBLIC_IP|UNIQUE_|DIFFERENT_|CHANGE_ME|replace-with)' \
  /opt/neko-cloud/.env; then
  echo 'STOP: resolve every placeholder before continuing' >&2
fi
sudo /usr/local/sbin/neko-cloud-preflight combined /opt/neko-cloud
```

Do not commit the runtime `.env` or include it in an unencrypted VM image or backup.

##### Tutorials: Combined: Create the private KasmVNC identity and login

**Context: VM**

```bash
sudo -iu desktop bash <<'EOF'
set -euo pipefail
install -d -m 0700 "$HOME/.vnc"
openssl req -x509 -nodes -newkey rsa:2048 -days 3650 \
  -subj '/CN=localhost' \
  -keyout "$HOME/.vnc/kasmvnc.key" \
  -out "$HOME/.vnc/kasmvnc.crt"
chmod 0600 "$HOME/.vnc/kasmvnc.key" "$HOME/.vnc/kasmvnc.crt"
EOF
```

Create a third, different credential for a write-capable Kasm login:

```bash
sudo -iu desktop
export XDG_RUNTIME_DIR=/run/user/$(id -u)
export DBUS_SESSION_BUS_ADDRESS=unix:path=$XDG_RUNTIME_DIR/bus
KASM_LOGIN='CHOOSE_A_KASM_LOGIN_NAME'
vncpasswd -u "$KASM_LOGIN" -w
chmod 0600 "$HOME/.kasmpasswd"
exit
```

Never reuse either Neko password. Treat `.kasmpasswd` as plaintext-equivalent because
KasmVNC obfuscates rather than strongly encrypts it.

##### Tutorials: Combined: Configure the persistent desktop and socket bridge

**Context: VM**

```bash
set -euo pipefail
DESKTOP_UID=$(id -u desktop)
sudo loginctl enable-linger desktop
sudo systemctl start "user@$DESKTOP_UID.service"
sudo -u desktop env \
  HOME=/home/desktop USER=desktop LOGNAME=desktop \
  XDG_RUNTIME_DIR="/run/user/$DESKTOP_UID" \
  DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$DESKTOP_UID/bus" \
  systemctl --user daemon-reload
sudo -u desktop env \
  HOME=/home/desktop USER=desktop LOGNAME=desktop \
  XDG_RUNTIME_DIR="/run/user/$DESKTOP_UID" \
  DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$DESKTOP_UID/bus" \
  systemctl --user enable --now 'kasmvncserver@:1.service'
```

Disable idle locking and suspend only for this locked, headless service account:

```bash
DESKTOP_UID=$(id -u desktop)
for setting in \
  "org.gnome.desktop.session idle-delay 'uint32 0'" \
  "org.gnome.desktop.screensaver lock-enabled false" \
  "org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'"; do
  sudo -u desktop env \
    HOME=/home/desktop USER=desktop LOGNAME=desktop \
    XDG_RUNTIME_DIR="/run/user/$DESKTOP_UID" \
    DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$DESKTOP_UID/bus" \
    sh -c "gsettings set $setting"
done
```

Keep the tested GNOME handover service, but do not configure or expose RDP:

```bash
sudo systemctl unmask gnome-remote-desktop.service
sudo systemctl enable --now gnome-remote-desktop.service
if sudo ss -lntup | grep -qE ':(3389)\b'; then
  echo 'ERROR: unexpected RDP listener; investigate before continuing' >&2
  exit 1
fi
```

Start and test the private Unix-socket bridge:

```bash
sudo systemd-analyze verify /etc/systemd/system/kasmvnc-socat.service
sudo systemctl daemon-reload
sudo systemctl enable --now kasmvnc-socat.service
sudo systemctl status kasmvnc-socat.service --no-pager
sudo curl --unix-socket /run/kasmvnc/kasm.sock -I http://localhost/
```

HTTP `401 Unauthorized` from that curl is expected. Never expose 8444 to work around a
bridge failure.

##### Tutorials: Combined: Validate configuration and pull Neko and Caddy

**Context: VM**

```bash
set -euo pipefail
cd /opt/neko-cloud
sudo docker compose config --quiet
sudo docker compose pull
```

The combined Caddyfile serves both hostnames. Neko port 8080 is only reachable on the
private Compose network; the desktop route uses the mounted KasmVNC Unix socket. The
images are pulled, but Caddy deliberately remains stopped until the host-firewall
subsection below is ready for certificate validation.

##### Tutorials: Combined: Configure the host firewall

The provider firewall is mandatory. For ordinary Ubuntu images that use UFW, mirror
only the combined port contract.

**Context: VM, non-OCI images only**

```bash
set -euo pipefail
ADMIN_CIDR='ADMIN_PUBLIC_IP/32'
case "$ADMIN_CIDR" in *ADMIN*|'0.0.0.0/0') echo 'Set a trusted admin CIDR' >&2; exit 1;; esac
sudo apt-get install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from "$ADMIN_CIDR" to any port 22 proto tcp comment 'SSH admin'
sudo ufw allow 80/tcp comment 'Caddy ACME'
sudo ufw allow 443/tcp comment 'Caddy HTTPS'
sudo ufw allow 59000/tcp comment 'Neko WebRTC fallback'
sudo ufw allow 59000/udp comment 'Neko WebRTC media'
sudo ufw --force enable
sudo ufw status numbered
```

Keep the current SSH session open until a second succeeds. Do not add 8080, 8444,
5901, or 3389. Docker-published ports make the cloud firewall essential even when UFW
is enabled.

**OCI exception:** Oracle warns against UFW on OCI Ubuntu platform images. Follow the
[OCI guide](#cloud-providers-oracle-cloud-infrastructure) and preserve its required link-local/iSCSI iptables rules.

#### Tutorials: Combined 6. Start

**Context: VM**

```bash
set -euo pipefail
cd /opt/neko-cloud
sudo docker compose up -d
```

Start Caddy and Neko only after the cloud and host firewall rules above are effective.
Then continue immediately with the checks below.

#### Tutorials: Combined 7. Verify

##### Tutorials: Combined: Local checks on the VM

**Context: VM**

```bash
set -euo pipefail
cd /opt/neko-cloud
DESKTOP_UID=$(id -u desktop)
systemctl is-active docker.service kasmvnc-socat.service gnome-remote-desktop.service
sudo -u desktop env \
  HOME=/home/desktop USER=desktop LOGNAME=desktop \
  XDG_RUNTIME_DIR="/run/user/$DESKTOP_UID" \
  DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$DESKTOP_UID/bus" \
  systemctl --user is-active 'kasmvncserver@:1.service'
sudo docker compose ps
sudo docker compose logs --tail=100 neko caddy
sudo ss -lntup
sudo ss -lnt | grep '127.0.0.1:8444'
sudo test -S /run/kasmvnc/kasm.sock

if curl -fsS --connect-timeout 2 http://127.0.0.1:8080/ >/dev/null; then
  echo 'ERROR: host port 8080 is unexpectedly published' >&2
  exit 1
fi
sudo docker compose exec caddy wget -qO- http://neko:8080/ >/dev/null
sudo /usr/local/sbin/neko-cloud-validate combined /opt/neko-cloud
```

Expected state:

- Neko, Caddy, KasmVNC, the bridge, and the GNOME handover service are active;
- TCP 8444 binds only to `127.0.0.1` and the Unix socket exists;
- host TCP 8080 is unreachable while Caddy can reach `neko:8080` privately;
- public listeners match TCP 80/443/59000 and UDP 59000;
- Caddy has trusted certificates for both hostnames.

The validation command is mode-specific. Do not validate this runtime with either
single-service mode contract.

##### Tutorials: Combined: Test the public boundary

**Context: External Windows PowerShell**

```powershell
$Ip = "PUBLIC_IP"
$NekoHost = "neko.example.com"
$DesktopHost = "desktop.example.com"
$Ports = 22,80,443,59000,5901,8080,8444,3389
foreach ($Port in $Ports) {
  $Open = Test-NetConnection -ComputerName $Ip -Port $Port -InformationLevel Quiet
  [pscustomobject]@{ Port = $Port; Open = $Open }
}
curl.exe -I "http://$NekoHost"
curl.exe -I "https://$NekoHost"
curl.exe -I "https://$DesktopHost"
```

Expected TCP results from the administrator's source address:

- open: 22, 80, 443, 59000;
- closed or filtered: 5901, 8080, 8444, 3389;
- from any other source, 22 is also closed or filtered;
- the Neko hostname returns a successful/redirect response;
- the desktop hostname returns `401` before KasmVNC credentials are supplied.

TCP testing does not verify UDP 59000. Test a real Neko media session from a separate
network.

##### Tutorials: Combined: Complete both functional acceptance tests

**Context: External web browser**

For Neko:

1. Open `https://neko.example.com` and verify the certificate.
2. Sign in with the regular password; confirm picture and audio.
3. Sign in with the administrator password; claim control and test keyboard, mouse,
   navigation, tabs, audio, and the intended clipboard policy.
   Viewing the stream alone does not grant input; use Neko's mouse/lock control button
   and administrator policy before treating clicks as broken.
4. Connect a second viewer and confirm both see one synchronized browser.
5. Test from another internet connection. Verify UDP media and TCP 59000 fallback.

For Ubuntu GNOME:

1. Open `https://desktop.example.com`, verify the certificate, and enter the separate
   Kasm login.
2. Confirm Ubuntu Dock, Activities, Files, Settings, Terminal, mouse, keyboard,
   clipboard policy, full screen, and display resize.
3. Close the local tab without signing out and reconnect; an open application should
   remain open.
4. Connect another viewer; it should replace the old exclusive viewer, not create a
   second desktop.
5. Do not treat desktop audio as promised by this standalone KasmVNC path.

For the combined system:

1. Run both services at the same time and observe `free -h`, CPU load, disk space, and
   interaction latency.
2. Restart Neko only; the GNOME session should remain available.
3. Restart the KasmVNC user service only; Neko should remain available.
4. Reboot the VM and repeat public-boundary and functional checks for both hostnames.
5. Inspect logs for restarts, out-of-memory events, authentication failures, and TLS
   errors.

On a borrowed computer, use private browsing, never save or share credentials, sign
out, and close the window. A compromised endpoint can still capture them.

#### Tutorials: Combined 8. Next steps

##### Tutorials: Combined: Operate, update, and recover

**Context: VM**

```bash
cd /opt/neko-cloud
sudo docker compose ps
sudo docker compose logs --since=30m neko caddy
sudo journalctl -u kasmvnc-socat.service --since=-30min --no-pager
DESKTOP_UID=$(id -u desktop)
sudo -u desktop env \
  HOME=/home/desktop USER=desktop LOGNAME=desktop \
  XDG_RUNTIME_DIR="/run/user/$DESKTOP_UID" \
  DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$DESKTOP_UID/bus" \
  journalctl --user -u 'kasmvncserver@:1.service' --since=-30min --no-pager
df -hT
free -h
```

Use the project procedures for [daily operation](#operations-operate-the-deployment),
[updates and rollback](#operations-update-and-roll-back),
[credential rotation](#operations-rotate-credentials),
[backup and restore](#operations-back-up-and-restore),
[cleanup and cloud teardown](#operations-safe-cleanup-and-cloud-teardown), and
[troubleshooting](#operations-troubleshooting).

The [full runbook](#reference-complete-multi-cloud-neko-and-ubuntu-browser-desktop-reference) remains the deep reference for exact
provider provisioning, current image identities, recovery, firewall audits, WebRTC,
GNOME/D-Bus, KasmVNC, Caddy, and backup internals.

## Cloud providers


Every tutorial needs one Ubuntu 24.04 VM with a stable public IPv4 address, outbound
internet access, DNS, and provider firewall rules. Pick a provider section:

- [Oracle Cloud Infrastructure](#cloud-providers-oracle-cloud-infrastructure)
- [Amazon Web Services](#cloud-providers-amazon-web-services)
- [Google Cloud](#cloud-providers-google-cloud)
- [Microsoft Azure](#cloud-providers-microsoft-azure)
- [Generic VPS or another cloud](#cloud-providers-generic-vps-or-another-cloud)

The exact CLI builds, safe state handling, and billing caveats live in the
[complete reference](#reference-6-common-cloud-prerequisites).

### Cloud providers: Mode-specific ingress

| Port | Neko-only | Desktop-only | Combined | Source |
|---|---:|---:|---:|---|
| TCP 22 | Yes | Yes | Yes | Administrator public CIDR only |
| TCP 80 | Yes | Yes | Yes | Public; ACME validation and redirect |
| TCP 443 | Yes | Yes | Yes | Public HTTPS/WebSocket |
| TCP 59000 | Yes | No | Yes | Public Neko WebRTC fallback |
| UDP 59000 | Yes | No | Yes | Public Neko WebRTC media |

Never expose Neko 8080, KasmVNC 8444, raw VNC 5901, or RDP 3389. Cloud firewall,
subnet rules, host firewall, container publishing, and listening addresses are separate
layers; verify all of them.

### Cloud providers: Common checklist

1. Select Ubuntu Server 24.04 LTS on AMD64 or ARM64.
2. Size for the chosen mode; see [sizing](#design-sizing-and-cost).
3. Use a 50 GiB or larger encrypted boot disk.
4. Reserve a static public IPv4 address before creating DNS.
5. Add the mode-specific ingress above and an unrestricted outbound path.
6. Create `A` records for the applicable hostnames.
7. Set a budget alert and learn which resources survive VM termination.
8. Keep provider console/serial recovery available until reboot testing passes.

### Cloud providers: Oracle Cloud Infrastructure

Use a Canonical Ubuntu 24.04 platform image. ARM64 `VM.Standard.A1.Flex` is suitable;
choose the current shape and allocation only after checking the tenancy's live limits and
cost estimate. Oracle's [Always Free documentation](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)
is the billing authority, not an old blog post.

#### Cloud providers / Oracle Cloud Infrastructure: Provisioning path

1. Create or select a VCN with a public subnet.
2. Attach and enable an internet gateway.
3. Ensure the subnet's effective route table has `0.0.0.0/0` to that gateway.
4. Create a dedicated application NSG with the rules in [the port table](#cloud-providers-mode-specific-ingress).
5. Launch Ubuntu 24.04 with an SSH public key and attach the primary VNIC to the NSG.
6. Assign a **reserved** public IP if the endpoint must survive rebuilds.
7. Verify that subnet security lists do not accidentally broaden the NSG policy.

OCI evaluates applicable security-list and NSG rules as a union: a restrictive NSG does
not cancel an overly broad security-list allow rule. See Oracle's
[security-rule comparison](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/securityrules.htm)
and [public-IP guide](https://docs.oracle.com/en-us/iaas/Content/Network/Tasks/managingpublicIPs.htm).

#### Cloud providers: Ubuntu firewall warning

Do **not** enable or reconfigure UFW on an OCI Ubuntu platform image. Oracle warns that
UFW changes can remove essential rules, including iSCSI/link-local access, and make an
instance fail to boot. Inspect and extend the existing persistent iptables policy without
flushing it. Follow Oracle's [Ubuntu/UFW known issue](https://docs.oracle.com/en-us/iaas/Content/Compute/known-issues.htm)
and [platform-image firewall guidance](https://docs.oracle.com/en-us/iaas/Content/Compute/References/images.htm).

Exact idempotent OCI CLI examples, ETag-safe route updates, NSG attachment, and reserved
IP handling are in [section 7 of the reference](#reference-7-oci-provisioning).

### Cloud providers: Amazon Web Services

Use Canonical Ubuntu Server 24.04 from its current regional AMI. Graviton/T4g is ARM64;
T3/T3a and many other families are AMD64. For combined use, start around two vCPU and
8 GiB RAM, then measure video-encoding load.

#### Cloud providers / Amazon Web Services: Provisioning path

1. Create a VPC, public subnet, internet gateway, and `0.0.0.0/0` route.
2. Create a security group with the rules in [the port table](#cloud-providers-mode-specific-ingress).
3. Import only an SSH **public** key; retain and verify the matching private key locally.
4. Launch Ubuntu with an encrypted gp3 boot volume and IMDSv2 required.
5. Allocate and associate an Elastic IP before creating DNS.
6. Configure budget alerts for compute, EBS, snapshots, data transfer, and public IPv4.

T-family instances are burstable. Sustained browser video encoding can exhaust baseline
credits or incur surplus-credit charges in Unlimited mode. Review AWS's
[T4g documentation](https://aws.amazon.com/ec2/instance-types/t4/) and
[Unlimited-mode behavior](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-performance-instances-unlimited-mode.html).
Public IPv4 and free-plan terms can change; consult [EC2 Free Tier](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-free-tier-usage.html)
and [VPC pricing](https://aws.amazon.com/vpc/pricing/).

Exact AWS CLI commands, current Canonical SSM AMI lookup, public-key correspondence
checks, and Elastic IP flow are in [section 8](#reference-8-aws-provisioning).

### Cloud providers: Google Cloud

Google Cloud's Free Tier `e2-micro` is too small for this workload. Choose a paid Ubuntu
24.04 VM sized for the selected mode. Use an AMD64 E2/N2 family or a supported ARM64
Tau T2A shape after confirming regional availability.

#### Cloud providers / Google Cloud: Provisioning path

1. Create a custom-mode VPC and regional subnet, or use a carefully reviewed existing one.
2. Reserve a regional external IPv4 address.
3. Create firewall rules targeted by network tag for the selected mode.
4. Create the VM with Shielded VM features where the architecture supports them.
5. Prefer OS Login; use IAP TCP forwarding for SSH when possible.
6. Point DNS only after the reserved address is attached.

If SSH uses Identity-Aware Proxy, allow the documented IAP source range only for TCP 22;
do not apply that range to public web/media rules. See
[IAP TCP forwarding](https://cloud.google.com/iap/docs/using-tcp-forwarding).
Review [Google Cloud Free Program](https://cloud.google.com/free/docs/free-cloud-features)
and the live pricing calculator before launch.

Exact `gcloud` network, firewall, ARM/AMD machine-image, static-IP, OS Login, and IAP
commands are in [section 9](#reference-9-gcp-provisioning).

### Cloud providers: Microsoft Azure

Use Canonical Ubuntu 24.04. AMD64 has the broadest VM-size availability. ARM64 requires
a compatible `p`-series size such as the current Bpsv2 family and may use different
security features, so validate the image/size combination in the target region.

#### Cloud providers / Microsoft Azure: Provisioning path

1. Create a resource group, VNet, and subnet.
2. Create a Standard SKU static public IPv4 address.
3. Create an NSG with the rules in [the port table](#cloud-providers-mode-specific-ingress).
4. Attach the NSG and address to a NIC.
5. Create the Ubuntu VM with an SSH public key and encrypted managed disk.
6. Verify outbound access explicitly; Azure's default outbound behavior has changed for
   newly created networks.
7. Add budget alerts and deletion locks only after understanding recovery/teardown.

Review the [Bpsv2 size documentation](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/general-purpose/bpsv2-series)
and current region pricing before launch. Exact Azure CLI commands and ARM64 image
selection are in [section 10](#reference-10-azure-provisioning).

### Cloud providers: Generic VPS or another cloud

The project does not depend on provider-specific agents. Any provider can work when it
offers Ubuntu 24.04, a public IPv4 address, inbound TCP/UDP filtering, outbound internet,
and enough CPU/RAM.

#### Cloud providers / Generic VPS or another cloud: Provisioning path

1. Create an Ubuntu 24.04 AMD64 or ARM64 VM.
2. Reserve a stable public IPv4 address if the provider supports it.
3. Configure the rules in [the mode-specific port table](#cloud-providers-mode-specific-ingress).
4. Confirm the provider has an internet route and no second firewall product blocking it.
5. Use provider DNS or external authoritative DNS for the required `A` records.
6. Confirm snapshot, backup, public-IP, stopped-instance, and outbound-transfer pricing.
7. Keep a web/serial console available until SSH and reboot recovery are proven.

On non-OCI Ubuntu, use the provider's supported host-firewall method. Docker-published
ports can bypass assumptions about UFW; validate the final nftables/iptables and Docker
rules rather than relying on one tool's status line.

Continue with [common Ubuntu preparation](#reference-12-prepare-ubuntu)
or one of the [short tutorials](#tutorials).


## Design


### Design: Architecture

The repository supports two applications and three deployment modes.

| Mode | Application | Runtime |
|---|---|---|
| Neko-only | Shared remote browser | Neko container plus Caddy container |
| Desktop-only | Persistent Ubuntu GNOME desktop | Host KasmVNC/GNOME plus Caddy container |
| Combined | Both hostnames on one VM | Both of the above |

#### Design: Combined request flow

```text
                         Internet
                            |
                      TCP 80 / 443
                            |
                     Caddy TLS gateway
                     /               \
         <NEKO_FQDN> /                 \ <DESKTOP_FQDN>
                   /                   \
        Docker network                  Unix socket
        neko:8080                       /run/kasmvnc/kasm.sock
             |                              |
     Neko browser container              socat bridge
             |                              |
       TCP+UDP 59000                 127.0.0.1:8444
        WebRTC media                       |
                                     KasmVNC :1
                                          |
                                  Ubuntu GNOME X11
                                  non-sudo desktop user
```

Caddy is the only public HTTP endpoint. Neko 8080 remains on a private Docker network.
KasmVNC 8444 binds to loopback and reaches Caddy through a root-owned Unix socket bridge.
Raw VNC and RDP are not part of this design.

#### Design: Trust boundaries

- **Cloud boundary:** routing, public/static IP, provider firewall, billing, snapshots.
- **Host boundary:** SSH, host firewall, systemd, isolated desktop account, filesystem.
- **Container boundary:** Docker daemon, Neko, Caddy, private frontend network.
- **Browser boundary:** credentials/cookies on the client laptop and interactive control.

The Docker daemon and root can inspect Neko environment secrets. Anyone who obtains the
Kasm password can act as the desktop OS user, so that user must not have sudo, Docker,
LXD, privileged certificate, or administrative group membership.

#### Design: Why GNOME is not a physical laptop console

The desktop mode runs the genuine Ubuntu GNOME session, Yaru theme, Ubuntu Dock, Files,
Settings, and host applications. KasmVNC provides a virtual X11 display instead of a
physical GDM/Wayland console. GDM stays masked to avoid a second invisible graphical
session. This is why it looks like Ubuntu Desktop while remaining browser-accessible.

### Design: Security model

This project deliberately exposes a small public surface and keeps application backends
private. It is a baseline, not a substitute for an organization's security review.

#### Design: Required controls

1. Restrict SSH to a trusted administrator CIDR, IAP/bastion, or provider session tool.
2. Publish only the ports required by the selected mode.
3. Use HTTPS hostnames and verify certificate hostname validation.
4. Generate unique Neko regular/admin and Kasm passwords; never reuse an OS password.
5. Run GNOME/KasmVNC as a locked, non-sudo `desktop` account.
6. Keep `.env` and `.kasmpasswd` mode `0600`; treat both as plaintext-equivalent.
7. Patch Ubuntu and review pinned application updates on a defined schedule.
8. Keep encrypted off-VM backups and test restoration.
9. Verify internal ports externally after every firewall or Compose change.

#### Design: Password-only access

The direct Caddy configuration supports application-password-only access. It does not
provide MFA. That can be convenient when accessing your own VM from another laptop, but
password theft becomes the primary risk. Use a password manager, avoid saving credentials
on borrowed machines, and close every Guest/Private window afterward.

For sensitive workloads, add an identity-aware proxy or VPN with MFA. Ensure it supports
WebSockets and Neko's separate WebRTC media path. An HTTP proxy alone does not replace
TCP/UDP 59000 or automatically make Neko work from networks that block both paths.

#### Design: Client-device reality

Guest/Incognito mode limits normal history and cookie persistence. It cannot protect
against a keylogger, malicious extension, screen recorder, compromised browser, or the
device owner. Do not access sensitive accounts from an untrusted computer.

See the [security policy](#reference-security-policy) for disclosure guidance and the
[complete reference](#reference-5-security-model) for SSH, firewall,
credential, backup, and recovery details.

### Design: Sizing and cost

These are starting points, not guarantees. Video resolution, frame rate, visited pages,
extensions, concurrent viewers, and software encoding can change CPU/RAM demand sharply.

| Mode | Minimum evaluation size | Comfortable starting size | Public media ports |
|---|---|---|---|
| Neko-only | 2 vCPU / 2-4 GiB | 2-4 vCPU / 4-8 GiB | TCP+UDP 59000 |
| Desktop-only | 2 vCPU / 4 GiB | 2-4 vCPU / 8 GiB | None beyond HTTPS |
| Combined | 2 vCPU / 8 GiB | 4 vCPU / 12-16 GiB | TCP+UDP 59000 |

Use at least 50 GiB encrypted storage; increase it for downloads, browser profiles,
snapshots, or desktop applications. Add swap only after measuring memory pressure and
understanding the provider disk's performance and cost.

The tested 1-GiB Azure Neko-only exception used `1024x576@20`, 1 GiB of container shared
memory, and 4 GiB of persistent disk-backed swap. It proves a constrained service can
start; it does not turn disk into RAM or establish a durable production minimum.

#### Design: Why â€œfree VMâ€ does not mean free deployment

Provider allowances change and are shared across an account. Charges can come from:

- public IPv4 addresses;
- boot/data volumes and snapshots;
- outbound data transfer;
- burst CPU surplus credits;
- DNS zones, backups, or monitoring;
- resources left after VM deletion;
- usage beyond a time-limited credit or free quota.

Review [cloud preparation](#cloud-providers), use the provider's current cost calculator,
create budget alerts, and test teardown. Alerts are not hard spending caps.

#### Design: ARM64 versus AMD64

ARM64 can be cost-effective and is supported by the pinned Neko Firefox and KasmVNC
paths. Google Chrome's Neko image is AMD64-only; Firefox or Chromium is the portable ARM
choice. Image digests are architecture-specific. See
[browser and CPU architecture](#design-browser-and-cpu-architecture).

### Design: Browser and CPU architecture

Two browsers are involved:

1. The **client browser** on your laptop displays Neko or KasmVNC.
2. The **remote browser** runs inside Neko on the VM.

Changing the client from Chrome to Firefox does not change the Neko container image.

#### Design: Neko image choice

| Remote Neko image | AMD64 | ARM64 | Notes |
|---|---:|---:|---|
| Firefox | Yes | Yes | Default here; ARM builds do not provide DRM |
| Chromium | Yes | Yes | Requires larger shared memory; no DRM on ARM |
| Google Chrome | Yes | No | Cannot run natively on ARM64 OCI A1/T2A/T4g |
| Brave/Vivaldi | Image-dependent | Image-dependent | Verify codecs, release, and architecture |

The upstream [Neko Docker image matrix](https://neko.m1k1o.net/docs/v3/installation/docker-images)
is authoritative. The templates default to Firefox because it covers both CPU families.

#### Design: Client browser choice

A current Chromium-family client (Chrome or Edge) offers the most predictable KasmVNC
experience. KasmVNC documents limitations in Firefox and a Safari Basic-Authentication/
WebSocket issue for direct deployments. Review its
[client-side requirements](https://www.kasmweb.com/kasmvnc/docs/latest/clientside.html).

#### Design: Changing architecture

When moving between AMD64 and ARM64:

- select a provider image and VM shape for the target architecture;
- use the matching KasmVNC `.deb` and verified checksum;
- pull the target platform's Neko/Caddy images;
- resolve and record new platform-specific digests;
- restore only architecture-neutral configuration and user data;
- rerun the complete acceptance tests.


## Operations


### Operations: Operate the deployment

Run these commands on the Ubuntu VM. Set the path to the deployed variant.

```bash
DEPLOY_DIR=/opt/neko-cloud
cd "$DEPLOY_DIR"
sudo docker compose --env-file .env ps
sudo docker compose --env-file .env logs --tail=100
sudo systemctl --failed --no-pager
sudo ss -lntup
```

For desktop-only or combined mode:

```bash
DESKTOP_USER=desktop
DESKTOP_UID=$(id -u "$DESKTOP_USER")
sudo -u "$DESKTOP_USER" \
  XDG_RUNTIME_DIR="/run/user/$DESKTOP_UID" \
  DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$DESKTOP_UID/bus" \
  systemctl --user status 'kasmvncserver@:1.service' --no-pager
sudo systemctl status kasmvnc-socat.service --no-pager
```

#### Operations: Browser access

- Neko: `https://<NEKO_FQDN>`
- Desktop: `https://<DESKTOP_FQDN>`

On a borrowed but trusted laptop, prefer Chrome Guest mode or a private window, do not
save credentials, and close all related windows at the end. The desktop defaults to one
exclusive viewer; a new viewer can replace the old viewer while the GNOME session stays
alive.

#### Operations: Monthly checklist

- Review failed logins and bounded container/system logs.
- Install Ubuntu security updates during a rollback-ready window.
- Review upstream Neko, KasmVNC, Caddy, Docker, and Ubuntu security releases.
- Confirm budget alerts, disk space, static-IP attachment, DNS, and certificate renewal.
- Test the expected URLs from a second network and recheck closed internal ports.
- Make an encrypted off-VM backup and periodically rehearse restoration.

### Operations: Update and roll back

Treat every image, package, or desktop change as a controlled maintenance event.

#### Operations: Container stack

1. Snapshot or back up configuration and record current image digests.
2. Read Neko and Caddy release notes.
3. Change the pinned reference in `/opt/neko-cloud/compose.yaml` and the compatibility
   table on a branch.
4. Validate before touching the VM:

   ```bash
   git diff --check
   ```

5. On the VM, open the selected template appendix and save its reviewed Compose/Caddy files without
   overwriting the runtime `.env`. Replace `MODE` with the selected deployment mode:

   ```bash
   set -euo pipefail
   MODE=MODE
   case "$MODE" in neko|desktop|combined) ;; *) echo 'Set MODE' >&2; exit 1;; esac
   cd "$HOME/neko-cloud-guide"
   if [ -n "$(git status --porcelain)" ]; then
     git status --short
     echo 'STOP: review or preserve local changes before updating' >&2
     exit 1
   fi
   git pull --ff-only
   git rev-parse HEAD
   sudoedit /opt/neko-cloud/compose.yaml
   sudoedit /opt/neko-cloud/Caddyfile
   sudo /usr/local/sbin/neko-cloud-preflight "$MODE" /opt/neko-cloud
   ```

   Stop if the checkout has unexpected local changes. Record the commit ID with the
   maintenance record.

6. Pull without replacing the running containers:

   ```bash
   cd /opt/neko-cloud
   sudo docker compose --env-file .env pull
   sudo docker compose --env-file .env images
   ```

7. Recreate and run the applicable acceptance tests:

   ```bash
   sudo docker compose --env-file .env up -d
   sudo docker compose --env-file .env ps
   sudo docker compose --env-file .env logs --tail=100
   ```

8. Run the mode-specific post-deployment validator from the checked-out repository;
   replace `MODE` with `neko`, `desktop`, or `combined`:

   ```bash
   sudo /usr/local/sbin/neko-cloud-validate MODE /opt/neko-cloud
   ```

To roll back, restore the prior Compose/Caddy files or image digests and run `up -d`
again. Do not delete old images until rollback has been proven unnecessary.

#### Operations: Host desktop

Take a provider snapshot before KasmVNC, GNOME, display, PAM, or systemd changes. Keep
SSH/serial recovery open, update packages, reboot, and validate the persistent session.
If GNOME was migrated from XFCE, retain the tested XFCE startup/unit until the GNOME
snapshot and restore path are verified.

The [complete reference update procedure](#reference-192-safe-updates-and-rollback)
includes digest recording, Ubuntu recovery, and rollback checks.

### Operations: Back up and restore

A VM snapshot is useful but is not a complete backup strategy. Keep an encrypted copy in
a different failure domain and test restoration on a disposable VM.

#### Operations: Separate configuration from secrets

Back up non-secret configuration separately from:

- `/opt/neko-cloud/.env`;
- `/home/<DESKTOP_USER>/.kasmpasswd`;
- KasmVNC private key material;
- cloud/SSH credentials.

Encrypt the secret archive before it leaves a memory-backed temporary directory. Never
place plaintext credentials in `/tmp`, an ordinary tarball, a Git repository, CI logs, or
cloud-init output. Remove plaintext staging immediately on both success and failure.

#### Operations: Minimum restore order

1. Provision a clean Ubuntu 24.04 VM and attach a stable address/firewall.
2. Install the pinned Docker/KasmVNC/desktop prerequisites.
3. Recreate the locked non-sudo desktop account with a new machine-local UID.
4. Restore non-secret files with explicit ownership and modes.
5. Decrypt secrets in memory-backed storage and install them mode `0600`.
6. Replace the public IP/DNS values for the new host.
7. Start KasmVNC, the socket bridge, Neko, and Caddy in dependency order.
8. Rotate restored credentials, reboot, and run every acceptance test.

The deep reference contains exact allow-listed archive membership, regular-file checks,
encrypted secret handling, and activation commands in
[backup and restore](#reference-194-backups-and-restore-order).

### Operations: Rotate credentials

Rotate any password or key that was pasted into chat, shown in a screenshot, reused,
stored on an untrusted laptop, or exposed through logs/backups.

#### Operations: Neko passwords

On the VM:

```bash
cd /opt/neko-cloud
sudoedit .env
sudo chmod 0600 .env
sudo docker compose --env-file .env up -d --force-recreate neko
```

Set independent generated values for the regular and administrator roles. Do not print
the rendered Compose configuration in a shared terminal because it can expose values.

#### Operations: KasmVNC web user

```bash
DESKTOP_USER=desktop
KASM_WEB_USER='existing-login-name'
sudo -iu "$DESKTOP_USER" vncpasswd -u "$KASM_WEB_USER" -w
sudo chmod 0600 "/home/$DESKTOP_USER/.kasmpasswd"
sudo chown "$DESKTOP_USER:$DESKTOP_USER" "/home/$DESKTOP_USER/.kasmpasswd"
```

The command prompts for and replaces that user's password while retaining write
permission. Do **not** add `-n` during rotation: in KasmVNC it means â€œdo not update the
passwordâ€ and changes permissions only. Reuse the existing login name for an ordinary
password rotation. Renaming a login is a separate migration: deleting an entry requires
an owner-authorized KasmVNC user, so do not add a new name and assume the old one can be
removed with the command above. The Kasm password file is reversibly obfuscated, not
strongly encrypted; treat it as plaintext-equivalent.

Restart the desktop user service through its running user manager, then verify login in
a new private browser window. Rotating Neko, Kasm, Linux, SSH, and cloud credentials are
separate operations.

### Operations: Safe cleanup and cloud teardown

#### Operations: Host cleanup

Start with evidence:

```bash
df -hT
sudo du -xhd1 /var /opt 2>/dev/null | sort -h
systemctl --failed --no-pager
sudo systemctl list-unit-files --state=enabled --no-pager
sudo docker system df
sudo apt-get -s autoremove
```

Clear only known caches first. Review `apt-get -s autoremove` before accepting it. Do not
blindly purge GNOME, XFCE rollback, Docker, provider agents, firewall persistence, network
services, snapshots, or backups. On OCI, never flush Oracle's link-local/iSCSI rules and
do not enable UFW.

Optional printing, mDNS, modem, Bluetooth, RPC, SSSD, or VPN services may be disabled
only after proving the VM does not use them. A service installed by a desktop metapackage
is not automatically safe to remove.

#### Operations: Complete provider teardown

After exporting and testing backups, inventory and deliberately delete or retain:

- VM and boot/data volumes;
- snapshots, images, and backup-vault copies;
- reserved/Elastic/static public IPs;
- DNS records and hosted zones;
- load balancers, NAT gateways, VPNs, bastions, or IAP-related resources;
- firewall groups/NSGs, NICs, subnets, routes, gateways, and VPC/VCN/VNet;
- log buckets, monitoring, secrets, service accounts, and budget alerts.

Stopping a VM is not the same as terminating it, and terminating a VM does not guarantee
that disks, addresses, snapshots, DNS, or networking stop billing. Finish by checking the
provider's resource inventory and cost view after billing data has caught up.

### Operations: Troubleshooting

Work from inside out: application, private proxy path, public gateway, host firewall,
cloud rules/routes, DNS, then client network.

#### Operations: First evidence

```bash
systemctl --failed --no-pager
sudo ss -lntup
sudo docker compose --project-directory /opt/neko-cloud --env-file /opt/neko-cloud/.env ps
sudo docker compose --project-directory /opt/neko-cloud --env-file /opt/neko-cloud/.env logs --tail=200
sudo journalctl -b -p warning --no-pager
```

| Symptom | Likely layer | First check |
|---|---|---|
| SSH and both URLs fail | VM/routing/public IP | Provider state, serial console, route, address attachment |
| HTTP works, HTTPS fails | DNS/ACME/firewall | `A` records, TCP 80/443, Caddy logs |
| Local Caddy/backend works but ACME times out | Cloud ingress/effective NSG | NIC and subnet NSGs, then external TCP 80/443 |
| Neko page loads, media fails | WebRTC path | `nat1to1`, TCP+UDP 59000, client network |
| Neko says token not found | Auth/client compatibility | Pinned version, legacy/cookie settings, fresh private window |
| Video works but clicks do not | Neko control ownership | Request/release control in Neko UI |
| Desktop returns 502 | Kasm/socket bridge | Kasm user unit, socat service, socket ownership |
| GNOME is blank or exits | User D-Bus/session env | `%t`, user lingering, xstartup, user journal |
| Reboot takes about two minutes | Network wait | Prove active renderer before changing any wait unit |
| VM becomes unresponsive | CPU/RAM/disk | Provider metrics, serial console, OOM/disk logs |

#### Operations: DNS and TLS

Replace the two example hostnames with the names enabled by the selected mode:

```bash
NEKO_HOST='neko.example.com'
DESKTOP_HOST='desktop.example.com'
getent ahostsv4 "$NEKO_HOST"
getent ahostsv4 "$DESKTOP_HOST"
curl -fsSI "https://$NEKO_HOST"
openssl s_client -connect "$NEKO_HOST:443" -servername "$NEKO_HOST" \
  -verify_hostname "$NEKO_HOST" </dev/null
```

#### Operations: Internal ports accidentally public

Inspect Compose port publishing, listening addresses, cloud rules, and Docker-managed
firewall rules. Remove the exposure and recreate the affected service; do not try to
hide a published Docker port with UFW alone.

The [complete troubleshooting reference](#reference-21-troubleshooting)
contains detailed recovery sequences for Caddy, Neko, WebRTC, KasmVNC, GNOME, DNS, TLS,
reboot failures, and unresponsive cloud instances.


## Reference


### Reference: Compatibility and support matrix

This reference applies to the three independent deployment modes in this repository:
Neko only, Ubuntu desktop only, and the combined single-VM stack. It records the tested
baseline and the boundaries that matter when choosing a cloud, CPU architecture, remote
client, or browser image. It is not a promise that every future upstream release remains
compatible. Re-check the linked upstream sources before changing a pin.

Baseline reviewed: **2026-07-19**.

#### Reference: Tested version baseline

| Component | Repository pin | AMD64 | ARM64 | Notes |
|---|---:|:---:|:---:|---|
| Host OS | Ubuntu Server 24.04 LTS | Yes | Yes | A minimal server image is expected; GNOME is added only for desktop modes. |
| Docker Engine | Current Ubuntu 24.04 packages from Docker's official repository | Yes | Yes | The Compose v2 plugin is required. Do not use an unmaintained standalone Compose v1 binary. |
| Neko Firefox | `ghcr.io/m1k1o/neko/firefox:3.1.4` | Yes | Yes | Default and tested Neko image. The templates keep its browser profile ephemeral. |
| Caddy | `caddy:2.11.4-alpine` | Yes | Yes | Current patched stable line at review time; persistent volumes retain ACME state. |
| KasmVNC | `1.4.0-1` Noble package | Yes | Yes | Install the architecture-matched `.deb` and verify its SHA-256 before installation. |
| Desktop | Ubuntu GNOME 46/X11 | Yes | Yes | KasmVNC supplies the X11 display. GDM stays masked on the headless VM. |

The official Caddy image is multi-platform. The repository pins a patch release rather
than a moving `latest` or `2-alpine` tag. Caddy's
[official Docker image](https://hub.docker.com/_/caddy) and
[2.11.4 release](https://github.com/caddyserver/caddy/releases/tag/v2.11.4) are the
authoritative sources for that pin. The image can later be replaced with an immutable
platform-specific digest after it has been pulled and tested on the target architecture.

#### Reference: Deployment-mode matrix

| Capability | Neko only | Desktop only | Combined |
|---|:---:|:---:|:---:|
| Shared controlled browser | Yes | No | Yes |
| Full Ubuntu GNOME desktop | No | Yes | Yes |
| Neko container | Yes | No | Yes |
| Host KasmVNC/GNOME session | No | Yes | Yes |
| Caddy HTTPS gateway | Yes | Yes | Yes |
| Required DNS names | One | One | Two distinct names |
| Neko media ports 59000 TCP/UDP | Yes | No | Yes |
| Suggested minimum | 2 vCPU, 4 GiB | 2 vCPU, 8 GiB | 2 vCPU, 12 GiB |

The resource figures are project recommendations for an interactive small deployment,
not vendor guarantees. Browser workload, tab count, video resolution, codecs, concurrent
viewers, and host contention can materially increase CPU and memory use. Avoid burstable
shapes that are already CPU-credit constrained for sustained video workloads.

#### Reference: Neko browser-image compatibility

The browser running *inside* Neko is separate from the browser used on the viewer's
laptop.

| Neko image family | AMD64 host | ARM64 host | Guidance |
|---|:---:|:---:|---|
| Firefox | Yes | Yes | Repository default; simplest cross-architecture choice. |
| Chromium | Yes | Yes | Usable, but allocate 2 GiB shared memory; DRM remains limited on ARM64. |
| Google Chrome | Yes | No | Use only on AMD64; it cannot run natively on an ARM64 OCI A1/Graviton/T2A host. |

Neko's [official Docker image matrix](https://neko.m1k1o.net/docs/v3/installation/docker-images)
is authoritative for supported architectures and DRM notes. The pinned 3.1.4 template
also carries `NEKO_LEGACY=true` and disables cookie sessions because that bundled client
expects the bearer token returned by `/api/login`. Treat those as version-specific
compatibility settings and re-audit authentication before upgrading Neko.

The templates intentionally do not mount a persistent browser profile. Container
recreation can remove tabs, extensions, cookies, downloads, and history inside Neko.
Add persistence only after confirming the exact profile path for the selected image and
designing encrypted backup and lifecycle handling.

#### Reference: KasmVNC package verification

The desktop templates target the dedicated, locked Linux user `desktop` with home
`/home/desktop`. They are not compatible with a numeric UID embedded in a user service;
`kasmvncserver@.service` uses systemd `%t` for the runtime directory.

The verified Noble packages are:

| Architecture | Release asset | SHA-256 |
|---|---|---|
| ARM64 | `kasmvncserver_noble_1.4.0_arm64.deb` | `120d9462cb5e917cad91a23f6cb0b780c06f701def40e900b29f996979200638` |
| AMD64 | `kasmvncserver_noble_1.4.0_amd64.deb` | `12bac6014149c5fdee75f0d403785aaa3e5dd4ea222de73253a5d4181bc9567e` |

Use the [KasmVNC installation matrix](https://www.kasmweb.com/kasmvnc/docs/latest/install.html)
and [KasmVNC 1.4.0 release](https://github.com/kasmtech/KasmVNC/releases/tag/v1.4.0)
to verify that these remain the correct Noble assets. Do not install an AMD64 package on
ARM64 or vice versa.

KasmVNC's `.kasmpasswd` is obfuscated rather than strongly encrypted. It is
plaintext-equivalent secret material: use a unique credential, mode `0600`, exclude it
from Git, and never reuse a Linux, cloud, Neko, or SSH password.

#### Reference: Cloud and CPU compatibility

The application layer is cloud-neutral. A target is compatible when it provides:

- Ubuntu Server 24.04 for the selected CPU architecture;
- a globally reachable static or reserved IPv4 address;
- inbound TCP 80 and 443, plus TCP/UDP 59000 for modes containing Neko;
- outbound HTTPS/DNS/NTP for packages, images, ACME, and normal browsing;
- at least the mode's recommended memory and enough boot storage for images, packages,
  logs, and desktop caches;
- provider firewall rules and a host firewall that agree with the port matrix below.

| Provider family | AMD64 examples | ARM64 examples | Compatibility notes |
|---|---|---|---|
| Oracle Cloud | AMD flexible shapes | Ampere A1 Flex | A1 is a strong fit for the ARM64 Firefox build. Always Free allowances are tenancy-wide and can change. |
| AWS EC2 | Current x86_64 general-purpose shapes | Graviton/T4g and later Arm shapes | Match the Canonical AMI architecture to the instance architecture. Public IPv4 and burst-credit charges are separate considerations. |
| Google Compute Engine | Current x86 general-purpose shapes | Tau T2A and later Arm shapes | The e2-micro Free Tier shape is too small for this interactive stack. Verify regional Arm image/shape availability. |
| Microsoft Azure | Current x64 general-purpose sizes | Current Arm64 general-purpose sizes | Confirm Ubuntu 24.04 image support for the chosen size and the subscription's outbound-connectivity design. |
| Other VPS/bare metal | x86-64 | ARMv8/AArch64 | Works when the OS, public routing, virtualization policy, and port requirements above are satisfied. |

Do not infer cost eligibility from technical compatibility. Free tiers, public IPv4
charges, storage, egress, snapshots, backups, and burst surcharges are provider/account
specific and change independently of this stack.

#### Reference: Viewer-browser compatibility

| Local viewer | Neko | KasmVNC desktop | Recommendation |
|---|:---:|:---:|---|
| Current Chrome | Yes | Yes | Preferred for the desktop; Guest mode is suitable on a borrowed laptop. |
| Current Edge | Yes | Yes | Preferred alternative; use InPrivate on a borrowed laptop. |
| Current Firefox | Yes | Limited | Neko is fine; KasmVNC documents client-side feature limitations. |
| Safari | Usually | Problematic directly | Basic Authentication may not be propagated reliably to WebSockets; do not make it the primary tested client. |
| Mobile browsers | Limited | Limited | Useful for emergency viewing, not the primary keyboard/mouse workflow. |

See KasmVNC's [client/browser requirements](https://www.kasmweb.com/kasmvnc/docs/master/clientside.html).
On an untrusted or borrowed computer, use a private/guest browser session, do not save
passwords, sign out, close every private window, and revoke/rotate credentials if the
machine may have been compromised. Private browsing does not defend against malware or
keyloggers on the laptop.

#### Reference: Network and protocol compatibility

| Port/protocol | Neko only | Desktop only | Combined | Exposure |
|---|:---:|:---:|:---:|---|
| 22/TCP | Optional administration | Optional administration | Optional administration | Restrict to trusted admin IPs or a private access path. |
| 80/TCP | Required | Required | Required | Public; ACME HTTP validation and redirect. |
| 443/TCP | Required | Required | Required | Public HTTPS and WebSockets. |
| 59000/UDP | Required | Closed | Required | Public Neko media path. |
| 59000/TCP | Required fallback | Closed | Required fallback | Public Neko media fallback. |
| 8080/TCP | Closed | Closed | Closed | Neko backend exists only in its Compose network. |
| 8444/TCP/UDP | Closed publicly | Closed publicly | Closed publicly | KasmVNC TCP is loopback-only; Caddy uses a Unix-socket bridge. |
| 5901/TCP | Closed | Closed | Closed | Raw VNC is not part of this design. |
| 3389/TCP | Closed | Closed | Closed | GNOME RDP is not exposed even when its handover service is active. |
| 443/UDP | Closed | Closed | Closed | Caddy is restricted to HTTP/1.1 and HTTP/2; HTTP/3 is disabled. |

Direct Neko media is not network-independent. Guest Wi-Fi or enterprise networks that
block both UDP and TCP 59000 can prevent media even though HTTPS loads. A TURN/TLS relay
on commonly permitted ports is a separate design and is not included here.

For desktop modes, public TLS terminates in Caddy. KasmVNC's HTTP WebSocket listener is
bound to `127.0.0.1:8444`, then bridged to `/run/kasmvnc/kasm.sock`; never publish 8444
through Compose or a cloud firewall. Standalone KasmVNC in this design does not promise
host audio forwarding. Neko audio is a separate WebRTC path.

#### Reference: DNS and TLS compatibility

Use ordinary DNS A records whenever possible:

- `NEKO_HOST` points to `PUBLIC_IP` for Neko modes;
- `DESKTOP_HOST` points to the same address for desktop modes;
- combined mode requires two different names;
- TCP 80 and 443 must reach Caddy during initial certificate issuance and renewal.

The Caddyfiles use environment placeholders supplied by Compose, so they contain no
deployment-specific IP address or hostname. Caddy stores ACME account and certificate
state in named volumes. Back up those volumes if preserving rate-limit history and ACME
state matters, but do not commit their contents.

Wildcard names such as `sslip.io` can be useful for a temporary test, but production
replicas should use a domain under the operator's control. IP-address certificates now
exist in parts of the public ACME ecosystem, but DNS names remain the portable default
across cloud providers and clients.

#### Reference: Upgrade compatibility policy

Treat every version change as a new deployment candidate:

1. Read the upstream release notes and image/package support matrix.
2. Pull only the intended tag, record its architecture-specific digest, and scan it.
3. Re-run Compose and Caddy validation without printing resolved secrets.
4. Test login roles, keyboard/mouse control, resize, clipboard, audio where applicable,
   Neko UDP and TCP media, reconnect behavior, and certificate renewal.
5. Reboot and run `sudo /usr/local/sbin/neko-cloud-validate MODE /opt/neko-cloud`, replacing
   `MODE` with the installed deployment mode, before promoting the version.
6. Keep the previous image digest and a provider snapshot until acceptance succeeds.

Moving tags are convenient for discovery but not reproducible deployment inputs. Patch
pins in this repository are review points; immutable image digests are the strongest
production pin once architecture-specific testing has completed.

#### Reference: Primary upstream references

- [Ubuntu 24.04 LTS releases](https://releases.ubuntu.com/noble/)
- [Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Docker Compose plugin](https://docs.docker.com/compose/install/linux/)
- [Neko installation](https://neko.m1k1o.net/docs/v3/installation)
- [Neko Docker images](https://neko.m1k1o.net/docs/v3/installation/docker-images)
- [Neko configuration](https://neko.m1k1o.net/docs/v3/configuration)
- [Neko reverse proxy setup](https://neko.m1k1o.net/docs/v3/reverse-proxy-setup)
- [Caddy official image](https://hub.docker.com/_/caddy)
- [Caddy 2.11.4](https://github.com/caddyserver/caddy/releases/tag/v2.11.4)
- [KasmVNC installation/support matrix](https://www.kasmweb.com/kasmvnc/docs/latest/install.html)
- [KasmVNC server configuration](https://www.kasmweb.com/kasmvnc/docs/master/serverside.html)
- [KasmVNC client requirements](https://www.kasmweb.com/kasmvnc/docs/master/clientside.html)
- [KasmVNC 1.4.0](https://github.com/kasmtech/KasmVNC/releases/tag/v1.4.0)

### Reference: Complete multi-cloud Neko and Ubuntu browser-desktop reference

Last verified: **2026-07-19**

This is the deep build, recovery, security, cost, and operations reference for a
portable Neko shared browser and Ubuntu GNOME/KasmVNC web desktop. It covers Oracle
Cloud Infrastructure (OCI), Amazon Web Services (AWS), Google Cloud Platform (GCP),
Microsoft Azure, and generic Linux VM/VPS providers. The shorter tutorials elsewhere
in this repository offer three supported modes: Neko-only, desktop-only, and combined.
The command sequence in this deep reference builds the combined mode as the superset.
For either single-service mode, use its short tutorial and port contract instead of
copying combined-mode firewall or service steps.

Commands in this reference use `neko-cloud` as a **replaceable example resource prefix**.
It is not a required host, account, or project name. Choose a lowercase project slug
and replace that prefix consistently in resource names and private inventory filenames.
The canonical application runtime directory remains `/opt/neko-cloud` unless you also
update every supplied script and procedure that references it.

The guide deliberately does **not** contain live passwords, private SSH keys, cloud
OCIDs, subscription IDs, or account numbers. Values such as `<PUBLIC_IP>`,
`<NEKO_FQDN>`, `<DESKTOP_FQDN>`, and `<ADMIN_PUBLIC_IP>/32` are required
substitutions, not omitted steps. Never commit `/opt/neko-cloud/.env`,
`~/.kasmpasswd`, a private key, or a cloud credential file.

#### Reference: Contents

1. [What this reference builds](#reference-1-what-this-reference-builds)
2. [Decisions and common misunderstandings](#reference-2-decisions-and-common-misunderstandings)
3. [Architecture and port contract](#reference-3-architecture-and-port-contract)
4. [Capacity, architecture, and free-tier reality](#reference-4-capacity-architecture-and-free-tier-reality)
5. [Security model](#reference-5-security-model)
6. [Common cloud prerequisites](#reference-6-common-cloud-prerequisites)
7. [OCI provisioning](#reference-7-oci-provisioning)
8. [AWS provisioning](#reference-8-aws-provisioning)
9. [GCP provisioning](#reference-9-gcp-provisioning)
10. [Azure provisioning](#reference-10-azure-provisioning)
11. [Generic VPS or other cloud](#reference-11-generic-vps-or-other-cloud)
12. [Prepare Ubuntu](#reference-12-prepare-ubuntu)
13. [Install Docker Engine and Compose](#reference-13-install-docker-engine-and-compose)
14. [Deploy Neko and Caddy](#reference-14-deploy-neko-and-caddy)
15. [Install Ubuntu GNOME and KasmVNC](#reference-15-install-ubuntu-gnome-and-kasmvnc)
16. [Connect KasmVNC privately to Caddy](#reference-16-connect-kasmvnc-privately-to-caddy)
17. [Start everything and validate end to end](#reference-17-start-everything-and-validate-end-to-end)
18. [How to use it from any laptop](#reference-18-how-to-use-it-from-any-laptop)
19. [Routine operations, updates, backups, and rollback](#reference-19-routine-operations-updates-backups-and-rollback)
20. [Safe cleanup](#reference-20-safe-cleanup)
21. [Troubleshooting](#reference-21-troubleshooting)
22. [Deployment inventory template](#reference-22-deployment-inventory-template)
23. [Authoritative references](#reference-23-authoritative-references)
24. [Replication completion checklist](#reference-24-replication-completion-checklist)

#### Reference: 1. What this reference builds

There are two separate remote applications on one Ubuntu 24.04 VM:

1. **Neko shared browser** â€” a Firefox browser running in the pinned
   `ghcr.io/m1k1o/neko/firefox:3.1.4` container. Neko sends video/audio and accepts
   keyboard/mouse input through a normal local web browser.
2. **Full Ubuntu desktop** â€” Ubuntu's GNOME Shell, Yaru theme, Ubuntu Dock, Files,
   Settings, and normal host applications, rendered in a KasmVNC virtual X11 display.
   It is also used through a normal web browser.

The resulting endpoints are:

- Neko: `https://<NEKO_FQDN>`
- Ubuntu GNOME: `https://<DESKTOP_FQDN>`
- VM: Ubuntu 24.04 on supported ARM64 or AMD64 cloud compute

Caddy terminates public TLS for both sites. Neko's web service on port 8080 and
KasmVNC's web service on port 8444 are never directly exposed.

The upstream projects describe Neko as a self-hosted virtual browser delivered through
Docker and KasmVNC as web-native remote access. See the
[Neko introduction](https://neko.m1k1o.net/docs/v3/introduction) and
[KasmVNC overview](https://www.kasmweb.com/kasmvnc/docs/latest/index.html).

#### Reference: 2. Decisions and common misunderstandings

##### Reference: Ubuntu Server versus Ubuntu Desktop

Cloud providers normally launch an **Ubuntu Server** image. It has the same Ubuntu base
as Ubuntu Desktop but normally lacks GNOME, GDM, and desktop applications. A VM does not
need to be destroyed and recreated merely to add a GUI.

These checks are useful:

```bash
cat /etc/os-release
dpkg --print-architecture
systemctl get-default
dpkg-query -W ubuntu-desktop-minimal gnome-shell xfce4 2>/dev/null
systemctl status display-manager.service --no-pager
```

`echo $XDG_CURRENT_DESKTOP` can be blank over SSH even when a desktop is installed,
because SSH is not the graphical session. Package and service checks are more reliable.

Keep the existing VM unless it has the wrong CPU architecture, an unusable disk/network
layout, corruption, or no data/configuration worth preserving. Snapshot or back up an
important boot volume before a desktop migration.

##### Reference: Why Firefox is the portable Neko default

The browser used to *view* Neko can be local Chrome, Edge, Firefox, or another modern
browser. The browser *inside* the Neko video is a separate remote browser.

For ARM64 deployments such as OCI A1:

| Neko image | AMD64 | ARM64 | Important consequence |
|---|---:|---:|---|
| Firefox | Yes | Yes | Portable default in this project; ARM build has no DRM |
| Chromium | Yes | Yes | Usable, no DRM on ARM; Chromium images require more shared memory |
| Google Chrome | Yes | **No** | Cannot run natively on OCI A1 ARM64 |
| Brave/Vivaldi | Yes | Yes | ARM availability exists, but validate required codecs/DRM |

This matrix comes from Neko's
[official Docker image and architecture documentation](https://neko.m1k1o.net/docs/v3/installation/docker-images).
The same page says Chromium-based images need `shm_size: 2g` and run Chromium with
`--no-sandbox` in the container. Firefox has fewer container requirements and is the
safer fit for this small ARM VM. On an AMD64 VM, the Neko Google Chrome image can be
selected deliberately, but it is not a drop-in option on ARM64.

##### Reference: GNOME versus XFCE

- **GNOME Ubuntu session** gives the familiar Ubuntu laptop appearance. It consumes
  more RAM/CPU and needs careful session setup when run without a physical display.
- **XFCE** is lighter and simpler. It was installed first and remains as an emergency
  rollback, but it does not look like standard Ubuntu Desktop.

The desktop and combined modes run the real Ubuntu GNOME X11 session in KasmVNC. It is not a
physical GDM/Wayland console. GDM is masked so a second invisible graphical session does
not waste resources or compete with KasmVNC.

##### Reference: KasmVNC versus XRDP

XRDP requires an RDP client and normally opens TCP 3389. This project's browser-first
flow uses KasmVNC behind Caddy instead.
TCP 3389 remains closed. Do not install or expose XRDP unless RDP is a separate,
explicit requirement.

##### Reference: Password-only baseline and stronger access options

The baseline can operate with application passwords and HTTPS alone; neither Neko nor
this standalone KasmVNC configuration adds MFA by itself. Password-only public access is
a usability tradeoff, not the recommended maximum-security posture. At minimum use
unique generated passwords, one active Kasm session, restricted SSH, rapid patching,
and closed internal ports. For higher-risk use, put an identity-aware proxy, VPN, or
provider-native access layer in front and follow its MFA/passkey guidance.

#### Reference: 3. Architecture and port contract

```text
                         PUBLIC INTERNET
                                |
                       TCP 80 and TCP 443
                                |
                         Caddy container
                         /              \
        Neko hostname /                  \ desktop hostname
                     /                    \
          Docker network                 Unix socket
        neko:8080 (private)        /run/kasmvnc/kasm.sock
                 |                         |
        Neko control/WSS                  socat
                 |                         |
      TCP+UDP 59000 WebRTC          127.0.0.1:8444
        directly to Neko                    |
                                      KasmVNC Xvnc
                                            |
                                  Ubuntu GNOME X11 session
```

Required inbound rules:

| Protocol | Port | Source | Purpose |
|---|---:|---|---|
| TCP | 22 | `<ADMIN_PUBLIC_IP>/32` | SSH administration |
| TCP | 80 | `0.0.0.0/0` | HTTP redirect and ACME HTTP validation |
| TCP | 443 | `0.0.0.0/0` | HTTPS/WSS for Neko and KasmVNC |
| TCP | 59000 | `0.0.0.0/0` | Neko WebRTC TCP fallback/multiplexing |
| UDP | 59000 | `0.0.0.0/0` | Neko WebRTC media/multiplexing |

Do **not** expose:

| Port | Reason |
|---:|---|
| 8080 | Neko HTTP backend; Caddy reaches it on the Docker network |
| 8444 TCP/UDP | KasmVNC backend; Caddy reaches TCP over a private Unix-socket bridge |
| 5901 | Raw/local VNC-compatible listener, if present |
| 3389 | RDP is not used |
| 631, 111, 1194 | Printing, rpcbind/NFS, and OpenVPN are not configured |

Caddy supports WebSocket proxying automatically and can obtain public certificates when
the hostname resolves to the VM and ports 80/443 are reachable; see Caddy's
[reverse proxy](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy) and
[automatic HTTPS](https://caddyserver.com/docs/automatic-https) documentation.

#### Reference: 4. Capacity, architecture, and free-tier reality

##### Reference: Practical capacity

Neko's official quick start lists 4 cores and 3 GB RAM as â€œgood performanceâ€ for
1280x720@30 and explains that browser rendering plus real-time encoding is expensive.
See [Neko Quick Start](https://neko.m1k1o.net/docs/v3/quick-start).

This combined Neko + full GNOME deployment should use:

- **Absolute experimental floor:** 2 vCPU, 4 GB RAM.
- **Practical minimum:** 2 vCPU, 8 GB RAM.
- **Current working OCI allocation:** 2 OCPU, 12 GB RAM.
- **Smoother concurrent use:** 4 vCPU, 16 GB RAM or more.
- **Disk:** 50 GB minimum; 100â€“200 GB if desktop applications and downloads are kept.

CPU is usually the limiting factor during video encoding. Lower the Neko/Kasm resolution
or frame rate before assuming that additional RAM fixes latency.

Both `linux/arm64` and `linux/amd64` work for this design. The VM image, cloud
instance type, KasmVNC package, and Neko image must all support the same architecture.

##### Reference: Current free-offer comparison

Cloud offers change. Treat this table as a **2026-07-19 snapshot**, verify it in the
provider console before provisioning, and set a budget alert.

| Provider | Current free/free-ish option | Suitable for the combined mode? | Main caveat |
|---|---|---:|---|
| OCI | A1 entitlement: 1,500 OCPU-hours + 9,000 GB-hours/month, equivalent to **2 OCPU/12 GB total** | Constrained but workable | Home region only; idle instances may be reclaimed; capacity can be unavailable |
| AWS | New free plan uses credits; separate T4g trial offers up to 750 t4g.small hours/month through 2026-12-31 | Marginal at 2 GB | Compute trial does not automatically cover public IPv4, EBS, DNS, snapshots, or egress |
| GCP | One e2-micro-equivalent monthly allowance in selected US regions, plus limited disk/egress | No | e2-micro is too small; ARM T2A/N4A/C4A is paid |
| Azure | 750 hours/month B2pts v2 ARM for 12 months for eligible new accounts | No | Free ARM SKU has only 1 GB RAM; disk, IP, bandwidth, and overage rules are separate |

###### Reference: OCI correction to old guidance

Historic articles often say A1 Always Free provides 4 OCPUs and 24 GB. Oracle's current
page instead states 1,500 OCPU-hours and 9,000 GB-hours, equivalent to **2 OCPUs and
12 GB total across the tenancy**. One continuous 2-OCPU/12-GB VM consumes that entire
continuous A1 allowance. Oracle also documents:

- 200 GB total Always Free boot + block storage in the home region.
- Five total Always Free volume backups.
- A default 50 GB boot volume; a 200 GB boot volume consumes the full storage allowance.
- Up to two A1 instances sharing 2 OCPUs and 12 GB.
- 10 TB/month Always Free outbound data.
- Possible reclamation when CPU p95, network, and A1 memory are all below 20% during a
  seven-day period.

The authoritative source is Oracle's
[Always Free Resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)
page. Always check **Limits, Quotas and Usage**, **Cost Analysis**, and the
[OCI cost estimator](https://www.oracle.com/cloud/costestimator.html). A budget alert is
not a hard spending cap.

Ubuntu and the installed desktop packages do not add a separate Windows-style OS
license, but compute, storage, backups, public IPs, DNS, and network traffic can still be
billed. OCI A1 is Arm and does not support Windows; choose an Ubuntu AArch64 image.

###### Reference: Other-provider free-offer sources

- AWS accounts created on/after 2025-07-15 use a credit-based six-month Free Plan;
  consult [EC2 Free Tier usage](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-free-tier-usage.html),
  [AWS Free Tier terms](https://aws.amazon.com/free/terms/), and the
  [T4g page](https://aws.amazon.com/ec2/instance-types/t4/).
- GCP's current allowance is documented in
  [Free Google Cloud features](https://docs.cloud.google.com/free/docs/free-cloud-features).
- Azure's new-customer offers are documented on
  [Azure Free](https://azure.microsoft.com/en-us/free/) and
  [Azure free services](https://azure.microsoft.com/en-us/pricing/free-services/).

#### Reference: 5. Security model

##### Reference: Credentials

- Use SSH public-key authentication. Do not send or copy the private key to the VM.
- Generate different random Neko user and administrator passwords.
- Choose a dedicated KasmVNC web username and unique password, granting write permission
  for keyboard/mouse control. Do not grant Kasm owner/user-management permission unless
  that administrative API is deliberately required.
- Neko secrets live in `/opt/neko-cloud/.env` with mode `0600`.
- KasmVNC credentials live in `~/.kasmpasswd` with mode `0600`.

KasmVNC explicitly documents that its password file is obfuscated rather than strongly
encrypted; filesystem access to it must be treated as password disclosure. See
[vncpasswd(1)](https://kasmweb.com/kasmvnc/docs/master/man/vncpasswd.html).

Treat any password pasted into chat, shown in a screenshot, or entered on an untrusted
device as exposed and rotate it. Rotation commands appear in the operations section.

##### Reference: Network layers

Apply least privilege at all applicable layers:

1. Provider firewall: OCI NSG/security list, AWS security group, GCP VPC firewall, or
   Azure NSG.
2. Host filtering: preserve provider-required rules; do not blindly flush iptables.
3. Service binding: keep KasmVNC on loopback and Neko 8080 only on the Docker network.
4. Reverse proxy: expose only Caddy 80/443 and Neko's 59000 TCP/UDP media port.

Docker-published ports are reachable on every host interface by default. Docker creates
NAT/FORWARD rules, and its documentation warns that published traffic is diverted
before the INPUT chains used by UFW. Therefore, â€œUFW denies itâ€ is not protection for a
published container port. Publish only the four intentional mappings and enforce the
provider firewall. See Docker's
[port publishing](https://docs.docker.com/engine/network/port-publishing/) and
[firewall behavior](https://docs.docker.com/engine/network/packet-filtering-firewalls/).

If a provider assigns public IPv6, configure equivalent IPv6 rules deliberately or
remove public IPv6. An IPv4-only firewall does not protect an IPv6 listener.

##### Reference: SSH

Use `<ADMIN_PUBLIC_IP>/32`, not `0.0.0.0/0`, for TCP 22. When traveling, update the
rule to the current trusted address or use a provider bastion/session service. Never
open password SSH merely for convenience.

An initial OCI security list may allow SSH from `0.0.0.0/0`. That works but is broader
than recommended; replace it with the administrator CIDR after confirming a second SSH
session.

##### Reference: Browser access on another person's laptop

Prefer a current Chromium-based browser (Chrome or Edge); KasmVNC documents feature
limitations in Firefox and problems with direct Safari use because Basic Authentication
is not reliably propagated to WebSockets. Use Chrome Guest mode or a private/incognito
window, do not save the password, and close all such windows when finished. This limits
ordinary browser history/cookie retention, but it cannot defeat a keylogger, screen
recorder, malicious extension, or compromised operating system. Do not use an untrusted
machine for sensitive accounts.

##### Reference: DNS and TLS

A real domain with `A` records such as `neko.example.com` and
`desktop.example.com` is the durable option. For disposable evaluation, sslip.io
resolves an IPv4 address embedded in a hostname. For example, IP `203.0.113.10`
can use `203-0-113-10.sslip.io`. The
[sslip.io documentation](https://sslip.io/) explains the mapping and notes that
wildcard certificates are not supported.

This runbook's default Caddy configuration requires those DNS hostnames. Since January
2026, Let's Encrypt also offers opt-in, approximately six-day public IP-address
certificates, but they require compatible ACME client/profile support and much more
frequent automated renewal; that is a separate design, not an instruction to replace
these hostnames. See Let's Encrypt's
[IP-certificate general-availability announcement](https://letsencrypt.org/2026/01/15/6day-and-ip-general-availability.html).

Use a reserved/static public IP before relying on DNS. If the IP changes:

1. Update DNS or the sslip.io hostnames.
2. Update `NEKO_WEBRTC_NAT1TO1`.
3. Update the Caddy hostnames.
4. Recreate the containers and verify WebRTC/TLS.

#### Reference: 6. Common cloud prerequisites

Before installing software, every provider must have:

- Ubuntu Server 24.04 LTS, ARM64 or AMD64.
- A compatible VM shape, preferably 2 vCPU/8 GB or larger.
- A 50 GB or larger persistent boot disk.
- A stable public IPv4 address for straightforward WebRTC and DNS.
- A public subnet or equivalent internet-routable NIC.
- Outbound DNS, NTP, HTTPS, Ubuntu APT, GHCR, Docker, and ACME access.
- Inbound rules matching the port contract and nothing more.
- An SSH public key and restricted TCP 22 source.
- A budget/cost alert and a known backup policy.

Record these values:

```text
CLOUD_PROVIDER=
REGION=
VM_NAME=
ADMIN_USER=ubuntu|azureuser|provider-specific
DESKTOP_USER=desktop
KASM_WEB_USER=desktopuser
ADMIN_CIDR=x.x.x.x/32
PUBLIC_IP=
NEKO_HOST=
DESKTOP_HOST=
ARCH=arm64|amd64
```

Determine the administrator's current public IPv4 from a trusted machine, not from the
VM:

```bash
curl -4 https://ifconfig.me
```

Confirm that local policy permits use of an external IP-echo service, or obtain the
address from the router/ISP instead.

##### Reference: 6.1 Preserve provisioning state across Cloud Shell sessions

Within each provider section, consecutive fenced CLI blocks assume the same
`set -euo pipefail` shell and the variables produced by earlier blocks. Cloud Shell tabs
are disposable; an interrupted create sequence must not be resumed with empty variables
or by blindly rerunning non-idempotent `create` commands.

Persist only non-secret inputs and returned resource IDs in a provider-specific file
such as `$HOME/neko-cloud-aws.env`, `$HOME/neko-cloud-gcp.env`, or
`$HOME/neko-cloud-azure.env`, set `umask 077`, write values with Bash `printf '%q'`, and
`chmod 0600` the result. Never store passwords, private keys, access tokens, or service
account JSON there. At minimum retain:

- AWS: account/region/AZ, VPC, subnet, internet gateway, route table, security group,
  Elastic IP allocation/association, AMI, key-pair name, and instance IDs.
- GCP: project/region/zone, network/subnet/firewall names, static address, machine/image,
  OS Login principal, and instance name/ID.
- Azure: subscription/location, resource group, VNet/subnet, NSG, public IP, NIC, image
  version, and VM name/ID.

After an interruption, use the provider's `describe`/`show`/`list` commands or console to
rediscover the exact existing resources, verify tags/names/IDs and attachments, rebuild
the state file, and continue at the first incomplete operation. Review duplicates and
partial resources before deleting anything. The OCI section supplies a concrete atomic
state-file implementation to prevent that common interrupted-shell failure mode.

#### Reference: 7. OCI provisioning

##### Reference: 7.1 Choose the image, shape, and storage

Create the VM in the tenancy's **home region** if it must consume Always Free A1
entitlement. Select the Canonical Ubuntu 24.04 platform image whose architecture is
`aarch64`, then select `VM.Standard.A1.Flex` and configure **2 OCPU / 12 GB RAM**.
Use a 50 GB or larger boot volume. Increase it only for measured application/data needs;
on OCI, remember that the documented 200 GB allowance is tenancy-wide storage.

The old 4 OCPU/24 GB advice is no longer the documented allowance. Oracle's current
[Always Free resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)
page defines 1,500 A1 OCPU-hours and 9,000 GB-hours each month, equivalent to 2 OCPU and
12 GB used continuously across the tenancy. It also documents 200 GB total boot/block
storage across the account, idle-instance reclamation criteria, and possible capacity
shortages. Confirm that the instance and volume show **Always Free eligible** before
launching; the console, [cost estimator](https://www.oracle.com/cloud/costestimator.html),
and [Cost Analysis](https://docs.oracle.com/en-us/iaas/Content/Billing/Concepts/costanalysisoverview.htm)
are authoritative for the tenancy.

Upload only the **public** SSH key at launch. On Windows, keep the private key outside
the repository and restrict its ACL:

```powershell
$Key = 'C:\path\to\neko-cloud.key'
$PublicIp = '<PUBLIC_IP>'
icacls.exe $Key /inheritance:r
icacls.exe $Key /grant:r "$($env:USERNAME):(R)"
ssh -i $Key "ubuntu@$PublicIp"
```

Never paste or upload the private key to the VM, a ticket, chat, or Git.

##### Reference: 7.2 Build the public network

For a new deployment, create a VCN (for example `10.0.0.0/16`), a public subnet (for
example `10.0.0.0/24`), and an enabled internet gateway. The subnet's effective route
table needs `0.0.0.0/0` with the internet gateway as target. Assign a public IPv4 to the
primary VNIC. A reserved public IP is preferable because an ephemeral address is lost
when its VNIC/instance is terminated. Oracle documents the required pieces under
[public IP addresses](https://docs.oracle.com/en-us/iaas/Content/Network/Tasks/managingpublicIPs.htm),
[internet gateways](https://docs.oracle.com/en-us/iaas/Content/Network/Tasks/managingIGs.htm),
and [route tables](https://docs.oracle.com/en-us/iaas/Content/Network/Tasks/managingroutetables.htm).

Use one application NSG attached to the VM's primary VNIC, with stateful ingress rules
matching the port contract. Security-list and NSG rules are a **union**: an NSG cannot
deny a port already allowed by the subnet security list. Remove broad duplicate rules
from the security list, and keep SSH limited to `<ADMIN_CIDR>`. See Oracle's
[security-rule behavior](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/securityrules.htm).

Before changing an existing security list, enumerate every subnet and VNIC that uses it.
Security-list changes affect every VNIC in each associated subnet and can lock out or
expose unrelated instances. Prefer the dedicated subnet/base list created below; if a
broad list is shared and cannot be safely tightened, move this VM to a dedicated subnet
instead of treating an NSG as a deny layer.

If the VNIC/private IP uses a per-resource route table, audit that effective route rather
than assuming the subnet table wins. If OCI Zero Trust Packet Routing security
attributes are configured, the applicable ZPR policy must also allow the traffic. A NAT
gateway supports outbound connections from private resources; it is not a substitute
for the public IP/internet-gateway path required by this direct inbound design.

###### Reference: Cloud Shell: create the OCI public network from zero

The console's **Automatically assign public IPv4** control is disabled when the selected
subnet prohibits public IPs, or when no suitable VCN/subnet has been selected. Do not
fight the disabled control. Create/select a public subnet whose
`prohibit-public-ip-on-vnic` value is false, then launch the instance into it.

In a fresh OCI Cloud Shell session, variables from an old session do not exist. The
creation block is **not** safe to rerun after a partial success: doing so can create
duplicate resources and consume quota. On the first run, persist the non-secret OCIDs in
the control file below. If interrupted, rediscover the already-created objects by display
name/console and reconstruct the file, or deliberately delete the verified partial
resources before restarting. Copy OCIDs from the console; never type them from a
screenshot:

```bash
set -euo pipefail
export OCI_CLI_REGION='<HOME_REGION>'
read -rp 'Paste target compartment OCID: ' COMPARTMENT_ID
case "$COMPARTMENT_ID" in
  ocid1.tenancy.*)
    oci iam tenancy get --tenancy-id "$COMPARTMENT_ID" \
      --query 'data.{Name:name,OCID:id}' --output table ;;
  ocid1.compartment.*)
    oci iam compartment get --compartment-id "$COMPARTMENT_ID" \
      --query 'data.{Name:name,OCID:id}' --output table ;;
  *) echo 'Not a tenancy or compartment OCID' >&2; exit 1 ;;
esac

VCN_ID=$(oci network vcn create --compartment-id "$COMPARTMENT_ID" \
  --display-name neko-cloud-vcn --dns-label nekocloud \
  --cidr-blocks '["10.0.0.0/16"]' --wait-for-state AVAILABLE \
  --query data.id --raw-output)

IGW_ID=$(oci network internet-gateway create \
  --compartment-id "$COMPARTMENT_ID" --vcn-id "$VCN_ID" \
  --display-name neko-cloud-internet-gateway --is-enabled true \
  --wait-for-state AVAILABLE --query data.id --raw-output)

jq -n --arg igw "$IGW_ID" '[{
  destination:"0.0.0.0/0",
  destinationType:"CIDR_BLOCK",
  networkEntityId:$igw
}]' > route-rules.json

ROUTE_TABLE_ID=$(oci network route-table create \
  --compartment-id "$COMPARTMENT_ID" --vcn-id "$VCN_ID" \
  --display-name neko-cloud-public-route-table \
  --route-rules file://route-rules.json --wait-for-state AVAILABLE \
  --query data.id --raw-output)

printf 'VCN_ID=%s\nIGW_ID=%s\nROUTE_TABLE_ID=%s\n' \
  "$VCN_ID" "$IGW_ID" "$ROUTE_TABLE_ID"
```

Create a subnet security list with outbound access and only the standard ICMP rules;
application ingress will live in the NSG from section 7.5:

```bash
jq -n '[{
  destination:"0.0.0.0/0",destinationType:"CIDR_BLOCK",
  protocol:"all",isStateless:false
}]' > base-egress.json

jq -n '[
  {source:"0.0.0.0/0",sourceType:"CIDR_BLOCK",protocol:"1",isStateless:false,
   icmpOptions:{type:3,code:4},description:"ICMP fragmentation required"},
  {source:"10.0.0.0/16",sourceType:"CIDR_BLOCK",protocol:"1",isStateless:false,
   icmpOptions:{type:3},description:"ICMP within VCN"}
]' > base-ingress.json

SECURITY_LIST_ID=$(oci network security-list create \
  --compartment-id "$COMPARTMENT_ID" --vcn-id "$VCN_ID" \
  --display-name neko-cloud-public-base \
  --egress-security-rules file://base-egress.json \
  --ingress-security-rules file://base-ingress.json \
  --wait-for-state AVAILABLE --query data.id --raw-output)

jq -n --arg id "$SECURITY_LIST_ID" '[$id]' > security-list-ids.json
SUBNET_ID=$(oci network subnet create \
  --compartment-id "$COMPARTMENT_ID" --vcn-id "$VCN_ID" \
  --display-name neko-cloud-public-subnet --dns-label public \
  --cidr-block 10.0.0.0/24 --route-table-id "$ROUTE_TABLE_ID" \
  --security-list-ids file://security-list-ids.json \
  --prohibit-public-ip-on-vnic false --wait-for-state AVAILABLE \
  --query data.id --raw-output)

oci network subnet get --subnet-id "$SUBNET_ID" \
  --query 'data.{id:id,cidr:"cidr-block",publicIpProhibited:"prohibit-public-ip-on-vnic",routeTable:"route-table-id",securityLists:"security-list-ids"}'

umask 077
printf 'COMPARTMENT_ID=%q\nVCN_ID=%q\nIGW_ID=%q\nROUTE_TABLE_ID=%q\nSECURITY_LIST_ID=%q\nSUBNET_ID=%q\n' \
  "$COMPARTMENT_ID" "$VCN_ID" "$IGW_ID" "$ROUTE_TABLE_ID" \
  "$SECURITY_LIST_ID" "$SUBNET_ID" > "$HOME/neko-cloud-oci-network.env"
chmod 0600 "$HOME/neko-cloud-oci-network.env"
```

Now create the NSG/rules in section 7.5. During Compute -> Create instance, select this
existing VCN/subnet, the application NSG, Canonical Ubuntu 24.04 aarch64,
`VM.Standard.A1.Flex` at 2 OCPU/12 GB, an appropriate boot volume, and the SSH
**public** key. For the reproducible static-address path, clear **Assign a public IPv4
address**, launch, and assign a new reserved public IP to the primary private IP with the
post-launch block in section 7.5. This avoids first creating an ephemeral address that
must be deleted and replaced. Do not launch until the NSG contains restricted SSH plus
the four public service rules.

In every later/new Cloud Shell session, restore those variables with
`. "$HOME/neko-cloud-oci-network.env"`; this directly prevents the empty `SUBNET_ID`/
`SECURITY_LIST_ID` failure seen in the original session. The file has resource IDs, not
passwords, but keep it private. Keep the generated JSON until the resource IDs and
effective rules are verified, then remove it from Cloud Shell. If an old deployment uses its
default security list instead, export and merge its complete rule array before updating;
never repeat the attached history's full replacement without preserving needed rules.

##### Reference: 7.3 OCI Ubuntu host firewall

Do **not** apply generic `ufw enable` instructions to an OCI Ubuntu platform image.
Oracle warns that UFW can replace essential link-local/iSCSI rules and can make an
instance fail to boot. UFW being absent or inactive does not mean the host has no
firewall: OCI Ubuntu images ship with an iptables policy. Read Oracle's
[known issue](https://docs.oracle.com/en-us/iaas/Content/Compute/known-issues.htm) and
[compute firewall best practices](https://docs.oracle.com/en-us/iaas/Content/Compute/References/bestpracticescompute.htm).

First back up and inspect, without flushing anything:

```bash
sudo install -d -m 0700 /root/firewall-backup
sudo iptables-save | sudo tee /root/firewall-backup/rules.v4.before >/dev/null
sudo ip6tables-save | sudo tee /root/firewall-backup/rules.v6.before >/dev/null
sudo iptables -S
sudo iptables -L -n -v --line-numbers
sudo nft list ruleset
```

Preserve all Oracle `InstanceServices` and `169.254.*` rules. Docker creates NAT and
forwarding rules for its published ports, so never flush the `nat`, `filter`,
`DOCKER`, or `DOCKER-USER` chains while the service is running. The provider NSG is the
primary public allow-list for this build.

If a host-native service must be allowed through the stock OCI `INPUT` chain, insert a
narrow stateful rule immediately before the terminal reject, verify the actual order,
and persist it. Oracle's own tutorial commonly inserts at position 6, but that position
must not be assumed on a modified machine:

```bash
sudo iptables -L INPUT -n --line-numbers
# Example only after identifying the reject's line number:
read -r -p 'Terminal reject line number: ' REJECT_LINE_NUMBER
read -r -p 'Host-native TCP port to allow: ' HOST_NATIVE_PORT
case "$REJECT_LINE_NUMBER:$HOST_NATIVE_PORT" in
  (*[!0-9:]*|:*|*:) echo 'Both values must be positive integers.' >&2; exit 1 ;;
esac
sudo iptables -I INPUT "$REJECT_LINE_NUMBER" -m conntrack --ctstate NEW \
  -p tcp --dport "$HOST_NATIVE_PORT" -j ACCEPT
command -v netfilter-persistent >/dev/null || {
  echo 'STOP: use the existing OCI image persistence mechanism; do not install/replace firewall tooling blindly' >&2
  exit 1
}
sudo netfilter-persistent save
sudo iptables -L INPUT -n --line-numbers
```

This deployment does not need host-native public 8080, 8444, or 3389. Do not add them.
KasmVNC remains on `127.0.0.1:8444`; its UDP listener remains blocked.

##### Reference: 7.4 Reproducible OCI Cloud Shell network audit

This replaces the deployment-specific OCIDs from the original Cloud Shell session
with placeholders. Run it in OCI Cloud Shell in the correct region. It is an **audit
first**; review every displayed resource before changing it. OCI OCIDs contain no
whitespace, so the unquoted shell variables below are safe for these exact values.

```bash
set -euo pipefail
export OCI_CLI_REGION='<HOME_REGION>'
COMPARTMENT_ID='<COMPARTMENT_OCID>'
INSTANCE_ID='<INSTANCE_OCID>'

oci compute instance list-vnics --instance-id $INSTANCE_ID --all
# Copy the primary VNIC and subnet OCIDs from the output:
VNIC_ID='<PRIMARY_VNIC_OCID>'
SUBNET_ID='<SUBNET_OCID>'

oci network vnic get --vnic-id $VNIC_ID
oci network subnet get --subnet-id $SUBNET_ID
# Copy vcn-id and route-table-id from the subnet output:
VCN_ID='<VCN_OCID>'
ROUTE_TABLE_ID='<ROUTE_TABLE_OCID>'

oci network route-table get --rt-id $ROUTE_TABLE_ID
oci network internet-gateway list --compartment-id $COMPARTMENT_ID \
  --vcn-id $VCN_ID --all
```

If the default route is missing, select the enabled internet gateway OCID and **merge**
the new rule into the existing array:

```bash
IGW_ID='<ENABLED_INTERNET_GATEWAY_OCID>'
oci network route-table get --rt-id "$ROUTE_TABLE_ID" > route-table.before.json
ETAG=$(jq -er '.etag' route-table.before.json)

# Normalize the CLI's kebab-case response into update input. OCI rejects a rule that
# contains both legacy cidrBlock and modern destination, so emit exactly one form.
jq '
  .data["route-rules"] | map(
    {networkEntityId: (."network-entity-id" // .networkEntityId)}
    + (if .destination != null then
         {destination: .destination,
          destinationType: (."destination-type" // .destinationType)}
       elif (."cidr-block" // .cidrBlock) != null then
         {cidrBlock: (."cidr-block" // .cidrBlock)}
       else
         error("route rule has neither destination nor cidrBlock")
       end)
    + (if .description then {description: .description} else {} end)
    + (if (."route-type" // .routeType) != null then
         {routeType: (."route-type" // .routeType)}
       else {} end)
  )
' route-table.before.json > route-rules.before.json

# Add the route only when absent; accept exactly one existing route only when it already
# targets the selected IGW. Abort on NAT/wrong-IGW/duplicate defaults for manual review.
jq --arg igw "$IGW_ID" '
  ([.[] | select((.destination // .cidrBlock) == "0.0.0.0/0")]) as $defaults
  | if ($defaults | length) == 0 then
      . + [{
        destination: "0.0.0.0/0",
        destinationType: "CIDR_BLOCK",
        networkEntityId: $igw
      }]
    elif (($defaults | length) == 1 and
          $defaults[0].networkEntityId == $igw) then .
    else error("conflicting or duplicate default route; review manually")
    end
' route-rules.before.json > route-rules.after.json

jq . route-rules.after.json
if ! cmp -s route-rules.before.json route-rules.after.json; then
  oci network route-table update --rt-id "$ROUTE_TABLE_ID" \
    --route-rules file://route-rules.after.json --if-match "$ETAG" --force
else
  echo 'The selected IGW default route is already present; no update required.'
fi
oci network route-table get --rt-id "$ROUTE_TABLE_ID"
```

`route-table update --route-rules` replaces the entire array. Never pass a one-rule
array unless deleting every other route is intentional. The ETag prevents overwriting a
concurrent edit. Keep the three JSON files as a short-lived audit/rollback record, then
delete them from Cloud Shell when the audit is done.

##### Reference: 7.5 Create the OCI application NSG

The console is least error-prone: VCN -> Network Security Groups -> Create, add the five
rules from section 3, then attach the NSG to the primary VNIC. The following CLI example
creates the NSG and rules. Replace the administrator CIDR before running it:

```bash
set -euo pipefail
NETWORK_ENV="$HOME/neko-cloud-oci-network.env"
[ -r "$NETWORK_ENV" ] || {
  echo "Missing $NETWORK_ENV; complete section 7.2 or reconstruct it from OCI." >&2
  exit 1
}
. "$NETWORK_ENV"
: "${COMPARTMENT_ID:?missing COMPARTMENT_ID}"
: "${VCN_ID:?missing VCN_ID}"
ADMIN_CIDR='<ADMIN_PUBLIC_IP>/32'
case "$ADMIN_CIDR" in *'<'*|'0.0.0.0/0')
  echo 'Replace ADMIN_CIDR with one trusted public IPv4 /32.' >&2; exit 1 ;;
esac

if [ -n "${NSG_ID:-}" ]; then
  ACTUAL_NSG_VCN=$(oci network nsg get --nsg-id "$NSG_ID" \
    --query 'data."vcn-id"' --raw-output)
  [ "$ACTUAL_NSG_VCN" = "$VCN_ID" ] || {
    echo 'Saved NSG_ID belongs to a different VCN.' >&2; exit 1;
  }
else
  NSG_ID=$(oci network nsg create \
    --compartment-id "$COMPARTMENT_ID" --vcn-id "$VCN_ID" \
    --display-name neko-cloud-web --wait-for-state AVAILABLE \
    --query data.id --raw-output)
fi

jq -n --arg admin "$ADMIN_CIDR" '[
  {direction:"INGRESS",protocol:"6",source:$admin,sourceType:"CIDR_BLOCK",
   tcpOptions:{destinationPortRange:{min:22,max:22}}},
  {direction:"INGRESS",protocol:"6",source:"0.0.0.0/0",sourceType:"CIDR_BLOCK",
   tcpOptions:{destinationPortRange:{min:80,max:80}}},
  {direction:"INGRESS",protocol:"6",source:"0.0.0.0/0",sourceType:"CIDR_BLOCK",
   tcpOptions:{destinationPortRange:{min:443,max:443}}},
  {direction:"INGRESS",protocol:"6",source:"0.0.0.0/0",sourceType:"CIDR_BLOCK",
   tcpOptions:{destinationPortRange:{min:59000,max:59000}}},
  {direction:"INGRESS",protocol:"17",source:"0.0.0.0/0",sourceType:"CIDR_BLOCK",
   udpOptions:{destinationPortRange:{min:59000,max:59000}}}
]' > nsg-rules.json

jq . nsg-rules.json
oci network nsg rules list --nsg-id "$NSG_ID" --all > nsg-rules.before.json

# Compare only security-relevant fields and add only absent rules. This makes a rerun
# safe after interruption instead of duplicating all five rules.
jq --slurpfile desired nsg-rules.json '
  def tuple: {
    direction: .direction,
    protocol: (.protocol | tostring),
    source: .source,
    sourceType: (."source-type" // .sourceType),
    isStateless: (."is-stateless" // .isStateless // false),
    sourceMin: (."tcp-options"."source-port-range".min //
                .tcpOptions.sourcePortRange.min //
                ."udp-options"."source-port-range".min //
                .udpOptions.sourcePortRange.min),
    sourceMax: (."tcp-options"."source-port-range".max //
                .tcpOptions.sourcePortRange.max //
                ."udp-options"."source-port-range".max //
                .udpOptions.sourcePortRange.max),
    min: (."tcp-options"."destination-port-range".min //
          .tcpOptions.destinationPortRange.min //
          ."udp-options"."destination-port-range".min //
          .udpOptions.destinationPortRange.min),
    max: (."tcp-options"."destination-port-range".max //
          .tcpOptions.destinationPortRange.max //
          ."udp-options"."destination-port-range".max //
          .udpOptions.destinationPortRange.max)
  };
  (.data | map(tuple)) as $have
  | [$desired[0][] | . as $rule | (tuple) as $key
     | select(($have | index($key)) == null) | $rule]
' nsg-rules.before.json > nsg-rules.missing.json

if [ "$(jq length nsg-rules.missing.json)" -gt 0 ]; then
  oci network nsg rules add --nsg-id "$NSG_ID" \
    --security-rules file://nsg-rules.missing.json
fi

# A dedicated application NSG must contain exactly these rules. Stop for manual review
# rather than silently preserving an unexpected broad rule or duplicate.
oci network nsg rules list --nsg-id "$NSG_ID" --all > nsg-rules.after.json
jq -e --slurpfile desired nsg-rules.json '
  def tuple: {
    direction: .direction,
    protocol: (.protocol | tostring),
    source: .source,
    sourceType: (."source-type" // .sourceType),
    isStateless: (."is-stateless" // .isStateless // false),
    sourceMin: (."tcp-options"."source-port-range".min //
                .tcpOptions.sourcePortRange.min //
                ."udp-options"."source-port-range".min //
                .udpOptions.sourcePortRange.min),
    sourceMax: (."tcp-options"."source-port-range".max //
                .tcpOptions.sourcePortRange.max //
                ."udp-options"."source-port-range".max //
                .udpOptions.sourcePortRange.max),
    min: (."tcp-options"."destination-port-range".min //
          .tcpOptions.destinationPortRange.min //
          ."udp-options"."destination-port-range".min //
          .udpOptions.destinationPortRange.min),
    max: (."tcp-options"."destination-port-range".max //
          .tcpOptions.destinationPortRange.max //
          ."udp-options"."destination-port-range".max //
          .udpOptions.destinationPortRange.max)
  };
  def ordered: sort_by(.direction,.protocol,.source,.sourceType,.isStateless,
                       .sourceMin,.sourceMax,.min,.max);
  (.data | map(tuple) | ordered) == ($desired[0] | map(tuple) | ordered)
' nsg-rules.after.json >/dev/null || {
  echo 'STOP: NSG contains unexpected or duplicate rules; review nsg-rules.after.json.' >&2
  exit 1
}

CONTROL_TMP=$(mktemp "$NETWORK_ENV.XXXXXX")
grep -Ev '^NSG_ID=' "$NETWORK_ENV" > "$CONTROL_TMP"
printf 'NSG_ID=%q\n' "$NSG_ID" >> "$CONTROL_TMP"
chmod 0600 "$CONTROL_TMP"
mv -f "$CONTROL_TMP" "$NETWORK_ENV"
```

For a fresh instance, create this NSG before launch and select it in the Create instance
networking panel. For an existing instance, attach it without discarding existing NSGs.
Use the console, or fetch the current VNIC `nsg-ids`, merge `$NSG_ID`, and pass the whole
resulting array to `vnic update` with the VNIC ETag:

```bash
VNIC_ID='<PRIMARY_VNIC_OCID>'
oci network vnic get --vnic-id "$VNIC_ID" > vnic.before.json
VNIC_ETAG=$(jq -er '.etag' vnic.before.json)
jq --arg nsg "$NSG_ID" \
  '(.data["nsg-ids"] // []) + [$nsg] | unique' \
  vnic.before.json > vnic-nsg-ids.after.json
jq . vnic-nsg-ids.after.json
oci network vnic update --vnic-id "$VNIC_ID" \
  --nsg-ids file://vnic-nsg-ids.after.json --if-match "$VNIC_ETAG"
oci network vnic get --vnic-id "$VNIC_ID" --query 'data."nsg-ids"'
```

After attachment, inspect both NSG and subnet security-list rules. Remove any security
list allow for 8080, 8444, 3389, or overly broad SSH.

If this is the fresh path, set `INSTANCE_ID` after launch, discover its primary VNIC and
private IP, then create and assign a **reserved** public IP. The VM remains unreachable
from the internet during the short interval before the final command completes:

```bash
set -euo pipefail
. "$HOME/neko-cloud-oci-network.env"
INSTANCE_ID='<NEW_INSTANCE_OCID>'
VNIC_ID=$(oci compute instance list-vnics --instance-id "$INSTANCE_ID" \
  --query 'data[0].id' --raw-output)

oci network private-ip list --vnic-id "$VNIC_ID" --all > private-ips.json
PRIMARY_COUNT=$(jq '[.data[] | select(.["is-primary"] == true)] | length' private-ips.json)
[ "$PRIMARY_COUNT" -eq 1 ] || {
  echo "Expected one primary private IP, found $PRIMARY_COUNT" >&2; exit 1;
}
PRIVATE_IP_ID=$(jq -er '.data[] | select(.["is-primary"] == true) | .id' private-ips.json)

PUBLIC_IP_ID=$(oci network public-ip create \
  --compartment-id "$COMPARTMENT_ID" --lifetime RESERVED \
  --display-name neko-cloud-public-ip --private-ip-id "$PRIVATE_IP_ID" \
  --wait-for-state ASSIGNED --query data.id --raw-output)
PUBLIC_IP=$(oci network public-ip get --public-ip-id "$PUBLIC_IP_ID" \
  --query 'data."ip-address"' --raw-output)

CONTROL_TMP=$(mktemp "$HOME/neko-cloud-oci-network.env.XXXXXX")
grep -Ev '^(INSTANCE_ID|VNIC_ID|PRIVATE_IP_ID|PUBLIC_IP_ID|PUBLIC_IP)=' \
  "$HOME/neko-cloud-oci-network.env" > "$CONTROL_TMP"
printf 'INSTANCE_ID=%q\nVNIC_ID=%q\nPRIVATE_IP_ID=%q\nPUBLIC_IP_ID=%q\nPUBLIC_IP=%q\n' \
  "$INSTANCE_ID" "$VNIC_ID" "$PRIVATE_IP_ID" "$PUBLIC_IP_ID" "$PUBLIC_IP" \
  >> "$CONTROL_TMP"
chmod 0600 "$CONTROL_TMP"
mv -f "$CONTROL_TMP" "$HOME/neko-cloud-oci-network.env"
printf 'Reserved public IPv4: %s\n' "$PUBLIC_IP"
```

Oracle does not convert an ephemeral public IP into a reserved public IP while keeping
the same address. On an existing instance, first plan the address/DNS/Caddy/Neko change,
unassign the ephemeral address, and then assign a reserved address; do not do this over
the only live SSH session without Cloud Shell/console recovery access.

Verify the OCI objects from **Cloud Shell**:

```bash
. "$HOME/neko-cloud-oci-network.env"
: "${NSG_ID:?missing NSG_ID}" "${VNIC_ID:?missing VNIC_ID}" \
  "${PUBLIC_IP_ID:?missing PUBLIC_IP_ID}"
oci network nsg rules list --nsg-id "$NSG_ID" --all
oci network vnic get --vnic-id "$VNIC_ID"
oci network public-ip get --public-ip-id "$PUBLIC_IP_ID" \
  --query 'data.{address:"ip-address",lifetime:lifetime,state:"lifecycle-state",privateIp:"private-ip-id"}'
```

After section 12.1 creates `/etc/neko-cloud.env`, verify egress from **inside the VM**:

```bash
. /etc/neko-cloud.env
ACTUAL_PUBLIC_IP=$(curl -fsS4 https://ifconfig.me)
[ "$ACTUAL_PUBLIC_IP" = "$PUBLIC_IP" ] || {
  printf 'Expected %s, observed %s\n' "$PUBLIC_IP" "$ACTUAL_PUBLIC_IP" >&2
  exit 1
}
```

The reserved address must equal `PUBLIC_IP` and later the Neko `nat1to1` value.
Delete `nsg-rules.json` after verification; it contains no secret, but temporary control
files should not accumulate.

#### Reference: 8. AWS provisioning

##### Reference: 8.1 Cost and instance choice

Use Ubuntu 24.04 ARM64 on `t4g.large` (2 vCPU/8 GiB) for a practical build, or
`t4g.medium` (2 vCPU/4 GiB) for a constrained test. `t4g.small` has 2 GiB and is
marginal for Neko plus GNOME. On AMD64 use a comparable `t3.large`/`t3a.large` and the
AMD64 Ubuntu image. AWS lists current T4g sizes on the
[T4g page](https://aws.amazon.com/ec2/instance-types/t4/).

AWS accounts created on or after 2025-07-15 use a credit-based free plan with a time limit;
the separate t4g.small trial is currently advertised through 2026-12-31. Neither
promise makes this whole stack free: EBS, snapshots, DNS, public IPv4, and data transfer
can be charged. Review the [EC2 free-tier rules](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-free-tier-usage.html),
[AWS free terms](https://aws.amazon.com/free/terms/), and
[VPC pricing](https://aws.amazon.com/vpc/pricing/). A public IPv4 is currently billed
hourly even while attached. Configure AWS Budgets, remembering alerts are not hard caps.

T4g is burstable and launches in `unlimited` mode by default. Sustained video encoding
above its CPU baseline can consume surplus credits and add charges; Standard mode avoids
surplus-credit charges but can throttle when credits run out. Monitor
`CPUCreditBalance`, `CPUSurplusCreditBalance`, and `CPUSurplusCreditsCharged`. For
predictable continuous encoding, use a paid fixed-performance Graviton family such as
M7g (or its current regional successor) after checking availability and price. See AWS's
[Unlimited-mode documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-performance-instances-unlimited-mode.html).

##### Reference: 8.2 Create a dedicated public VPC with AWS CLI

The console equivalent is VPC -> Create VPC -> VPC and more, with one public subnet.
For CLI, use an authenticated administrator shell and pick an Availability Zone that
supports the selected instance type:

```bash
set -euo pipefail
export AWS_REGION='<AWS_REGION>'
export AWS_DEFAULT_REGION="$AWS_REGION"
ADMIN_CIDR='<ADMIN_PUBLIC_IP>/32'
AZ='<AVAILABILITY_ZONE>'
KEY_NAME='<EC2_KEY_PAIR_NAME>'
PRIVATE_KEY_FILE='<LOCAL_MATCHING_SSH_PRIVATE_KEY_FILE>'
PUBLIC_KEY_FILE='<LOCAL_MATCHING_SSH_PUBLIC_KEY_FILE.pub>'

# Fail before creating billable resources unless the local private/public pair matches.
case "$PUBLIC_KEY_FILE" in
  *.pub) ;;
  *) printf 'PUBLIC_KEY_FILE must name a .pub file, not a private key\n' >&2; exit 1 ;;
esac
[ -f "$PRIVATE_KEY_FILE" ] || {
  printf 'Private key not found: %s\n' "$PRIVATE_KEY_FILE" >&2; exit 1;
}
[ -f "$PUBLIC_KEY_FILE" ] || {
  printf 'Public key not found: %s\n' "$PUBLIC_KEY_FILE" >&2; exit 1;
}
LOCAL_PUBLIC=$(awk 'NR == 1 { print $1 " " $2 } NR > 1 { exit 1 }' "$PUBLIC_KEY_FILE")
[[ "$LOCAL_PUBLIC" =~ ^ssh-(ed25519|rsa)[[:space:]] ]] || {
  printf 'Expected one ssh-ed25519 or ssh-rsa OpenSSH public key\n' >&2; exit 1;
}
[ "$(ssh-keygen -y -f "$PRIVATE_KEY_FILE")" = "$LOCAL_PUBLIC" ] || {
  printf 'The local private and public keys do not match\n' >&2; exit 1;
}

# EC2 key pairs are regional. Compare an existing pair's public material; otherwise
# import ONLY the public half. The private key never leaves this computer.
AWS_PUBLIC=$(aws ec2 describe-key-pairs --include-public-key \
  --filters "Name=key-name,Values=$KEY_NAME" \
  --query 'KeyPairs[0].PublicKey' --output text)
if [ "$AWS_PUBLIC" = 'None' ]; then
  aws ec2 import-key-pair --key-name "$KEY_NAME" \
    --public-key-material "fileb://$PUBLIC_KEY_FILE" >/dev/null
else
  AWS_PUBLIC=$(printf '%s\n' "$AWS_PUBLIC" | awk '{ print $1 " " $2 }')
  [ "$AWS_PUBLIC" = "$LOCAL_PUBLIC" ] || {
    printf 'EC2 key pair %s does not match the local private key\n' "$KEY_NAME" >&2
    exit 1
  }
fi

VPC_ID=$(aws ec2 create-vpc --cidr-block 10.20.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=neko-cloud-vpc}]' \
  --query Vpc.VpcId --output text)
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-support Value=true
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-hostnames Value=true

SUBNET_ID=$(aws ec2 create-subnet --vpc-id "$VPC_ID" \
  --cidr-block 10.20.0.0/24 --availability-zone "$AZ" \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=neko-cloud-public}]' \
  --query Subnet.SubnetId --output text)
aws ec2 modify-subnet-attribute --subnet-id "$SUBNET_ID" \
  --map-public-ip-on-launch

IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=neko-cloud-igw}]' \
  --query InternetGateway.InternetGatewayId --output text)
aws ec2 attach-internet-gateway --internet-gateway-id "$IGW_ID" --vpc-id "$VPC_ID"

RT_ID=$(aws ec2 create-route-table --vpc-id "$VPC_ID" \
  --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id "$RT_ID" \
  --destination-cidr-block 0.0.0.0/0 --gateway-id "$IGW_ID"
aws ec2 associate-route-table --route-table-id "$RT_ID" --subnet-id "$SUBNET_ID"
```

To reuse an existing private key, derive its public half locally with explicit paths:

```bash
PRIVATE_KEY_FILE='/secure/path/existing-private-key'
NEW_PUBLIC_KEY_FILE='/secure/path/derived-public-key'
ssh-keygen -y -f "$PRIVATE_KEY_FILE" > "${NEW_PUBLIC_KEY_FILE}.pub"
```

Protect the private file, and point `PUBLIC_KEY_FILE` at only the new `.pub` file. Never pass a
`.key`, `.pem`, or other private-key file to `import-key-pair`. The comparison uses
AWS CLI's documented [`--include-public-key`](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-key-pairs.html)
response; public-key import is documented in
[`import-key-pair`](https://docs.aws.amazon.com/cli/latest/reference/ec2/import-key-pair.html).

Do not add IPv6 unless it is also secured. If enabled, add an IPv6 subnet route and
equivalent restricted security-group rules intentionally.

##### Reference: 8.3 Security group, Ubuntu AMI, VM, and Elastic IP

Create an application security group. AWS security groups are stateful, so return
traffic is automatic:

```bash
SG_ID=$(aws ec2 create-security-group --group-name neko-cloud-web \
  --description 'Neko and KasmVNC web gateway' --vpc-id "$VPC_ID" \
  --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id "$SG_ID" \
  --protocol tcp --port 22 --cidr "$ADMIN_CIDR"
aws ec2 authorize-security-group-ingress --group-id "$SG_ID" \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id "$SG_ID" \
  --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id "$SG_ID" \
  --protocol tcp --port 59000 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id "$SG_ID" \
  --protocol udp --port 59000 --cidr 0.0.0.0/0
```

Resolve Canonical's current Ubuntu 24.04 ARM64 AMI through its official SSM parameter,
then record the returned AMI ID for the build manifest:

```bash
AMI_ID=$(aws ssm get-parameter \
  --name /aws/service/canonical/ubuntu/server/24.04/stable/current/arm64/hvm/ebs-gp3/ami-id \
  --query Parameter.Value --output text)
printf 'AMI_ID=%s\n' "$AMI_ID"
```

Canonical documents this path in its
[AWS Ubuntu launch guide](https://documentation.ubuntu.com/aws/aws-how-to/instances/launch-ubuntu-desktop/).
For AMD64, replace `arm64` with `amd64` and choose an x86 instance type.

Launch an encrypted 80 GiB instance with IMDSv2 required:

```bash
INSTANCE_ID=$(aws ec2 run-instances --image-id "$AMI_ID" \
  --instance-type t4g.large --key-name "$KEY_NAME" \
  --subnet-id "$SUBNET_ID" --security-group-ids "$SG_ID" \
  --no-associate-public-ip-address \
  --metadata-options HttpTokens=required,HttpEndpoint=enabled \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":80,"VolumeType":"gp3","Encrypted":true,"DeleteOnTermination":true}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=neko-cloud}]' \
  --query 'Instances[0].InstanceId' --output text)
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"

ALLOCATION_ID=$(aws ec2 allocate-address --domain vpc \
  --query AllocationId --output text)
aws ec2 associate-address --instance-id "$INSTANCE_ID" \
  --allocation-id "$ALLOCATION_ID"
PUBLIC_IP=$(aws ec2 describe-addresses --allocation-ids "$ALLOCATION_ID" \
  --query 'Addresses[0].PublicIp' --output text)
printf 'PUBLIC_IP=%s\n' "$PUBLIC_IP"
ssh -i "$PRIVATE_KEY_FILE" ubuntu@"$PUBLIC_IP"
```

If the selected AMI exposes a different root device name, inspect its block-device
mapping before launch and change `/dev/sda1`. Keep the allocation ID in the inventory;
an unattached Elastic IP is still billable. AWS Systems Manager Session Manager is a
safer optional alternative to public SSH when its agent, IAM role, and endpoints are
configured. Do not expose 8080, 8444, or 3389 in the security group or network ACL.

#### Reference: 9. GCP provisioning

##### Reference: 9.1 Cost and machine choice

Google Cloud's free Compute Engine allowance is one `e2-micro` in selected US regions,
with limited standard disk and egress. It is x86, has only 1 GB RAM, and is not suitable
for this combined stack. See the current
[Google Cloud Free Tier](https://cloud.google.com/free/docs/free-cloud-features) and
[E2 machine types](https://cloud.google.com/compute/docs/general-purpose-machines).
Streaming can consume the small free egress allowance quickly; budgets generate alerts,
not a hard spending cap.

Use a paid ARM VM such as `t2a-standard-2` (2 vCPU/8 GB) in a supported region, or
`c4a-standard-2` where C4A is available, including supported Mumbai zones. Prefer 4
vCPU/16 GB for simultaneous Neko and GNOME use. Confirm live availability under
[regions and zones](https://cloud.google.com/compute/docs/regions-zones) because ARM
families are not offered in every zone. The official Ubuntu ARM64 image family is
`ubuntu-2404-lts-arm64` in project `ubuntu-os-cloud`; see
[OS details](https://cloud.google.com/compute/docs/images/os-details).

External IPv4 and outbound traffic are billable. Reserve the address because a changed
IP breaks both DNS and Neko `nat1to1`. Review current
[VPC pricing](https://cloud.google.com/vpc/pricing/) and create a Billing budget/alert.

##### Reference: 9.2 VPC, static address, and firewall

Run with an authenticated `gcloud` CLI. Choose a zone that supports the selected ARM
machine type:

```bash
set -euo pipefail
PROJECT_ID='<GCP_PROJECT_ID>'
REGION='<GCP_REGION>'
ZONE='<GCP_ZONE>'
MACHINE_TYPE='<t2a-standard-2-or-c4a-standard-2>'
ADMIN_CIDR='<ADMIN_PUBLIC_IP>/32'

gcloud config set project "$PROJECT_ID"
gcloud services enable compute.googleapis.com
gcloud compute networks create neko-cloud-vpc --subnet-mode=custom
gcloud compute networks subnets create neko-cloud-public \
  --network=neko-cloud-vpc --region="$REGION" --range=10.30.0.0/24
gcloud compute addresses create neko-cloud-ip --region="$REGION" --network-tier=PREMIUM
PUBLIC_IP=$(gcloud compute addresses describe neko-cloud-ip --region="$REGION" \
  --format='value(address)')

gcloud compute firewall-rules create neko-cloud-ssh \
  --network=neko-cloud-vpc --direction=INGRESS --priority=1000 \
  --action=ALLOW --rules=tcp:22 --source-ranges="$ADMIN_CIDR" \
  --target-tags=neko-cloud-remote
gcloud compute firewall-rules create neko-cloud-public-web \
  --network=neko-cloud-vpc --direction=INGRESS --priority=1000 \
  --action=ALLOW --rules=tcp:80,tcp:443,tcp:59000,udp:59000 \
  --source-ranges=0.0.0.0/0 --target-tags=neko-cloud-remote
```

GCP VPC firewall rules are stateful. Do not create ingress rules for 8080, 8444, or
3389. Avoid broad default rules that undermine these target-tag rules; inspect them:

```bash
gcloud compute firewall-rules list \
  --filter='network:neko-cloud-vpc' \
  --format='table(name,direction,sourceRanges,allowed,targetTags)'
```

##### Reference: 9.3 Launch Ubuntu ARM64

Resolve the moving image family to a concrete image name and record it before launch:

```bash
IMAGE_NAME=$(gcloud compute images describe-from-family ubuntu-2404-lts-arm64 \
  --project=ubuntu-os-cloud --format='value(name)')
case "$MACHINE_TYPE" in
  t2a-*)
    BOOT_DISK_TYPE='pd-balanced'
    NETWORK_INTERFACE="network=neko-cloud-vpc,subnet=neko-cloud-public,address=$PUBLIC_IP,network-tier=PREMIUM,nic-type=GVNIC"
    ;;
  c4a-*)
    BOOT_DISK_TYPE='hyperdisk-balanced'
    NETWORK_INTERFACE="network=neko-cloud-vpc,subnet=neko-cloud-public,address=$PUBLIC_IP,network-tier=PREMIUM,nic-type=GVNIC"
    ;;
  *) echo "Unsupported documented ARM family: $MACHINE_TYPE" >&2; exit 1 ;;
esac
printf 'IMAGE_NAME=%s PUBLIC_IP=%s DISK=%s\n' \
  "$IMAGE_NAME" "$PUBLIC_IP" "$BOOT_DISK_TYPE"

gcloud compute instances create neko-cloud --zone="$ZONE" \
  --machine-type="$MACHINE_TYPE" --provisioning-model=STANDARD \
  --network-interface="$NETWORK_INTERFACE" \
  --image="$IMAGE_NAME" --image-project=ubuntu-os-cloud \
  --boot-disk-size=80GB --boot-disk-type="$BOOT_DISK_TYPE" \
  --boot-disk-auto-delete --shielded-secure-boot \
  --metadata=enable-oslogin=TRUE --tags=neko-cloud-remote \
  --no-service-account --no-scopes

gcloud compute instances describe neko-cloud --zone="$ZONE" \
  --format='yaml(name,machineType,disks,networkInterfaces,metadata,shieldedInstanceConfig)'
```

Because the VM enables OS Login, grant the human administrator OS Admin Login before
SSH. Grant it at the instance (least scope); Console/CLI users may additionally need a
project-level role containing `compute.projects.get`, as described in Google's
[OS Login setup guide](https://cloud.google.com/compute/docs/oslogin/set-up-oslogin):

```bash
GCP_LOGIN_MEMBER='<user:administrator@example.com>'
gcloud compute instances add-iam-policy-binding neko-cloud --zone="$ZONE" \
  --member="$GCP_LOGIN_MEMBER" --role=roles/compute.osAdminLogin
gcloud compute ssh neko-cloud --zone="$ZONE"
```

The exact availability of Shielded VM settings and service-account flags can differ by
image/family; do not silently drop a security flag after a CLI error. Review the live
`gcloud compute instances create --help`, decide why it is incompatible, and record the
change in the build manifest.

For safer SSH, configure
[IAP TCP forwarding](https://cloud.google.com/iap/docs/using-tcp-forwarding), allow TCP
22 only from `35.235.240.0/20`, grant `roles/iap.tunnelResourceAccessor` to the same
principal at the narrowest workable scope, remove the direct SSH rule, and connect with:

```bash
gcloud compute ssh neko-cloud --zone="$ZONE" --tunnel-through-iap
```

IAP's proxy, not the administrator's laptop address, becomes the source seen by the VM.
If section 12.2 enables UFW, set `SSH_SOURCE_CIDR='35.235.240.0/20'` there as well;
allowing that range only in the VPC firewall while UFW still permits only
`<ADMIN_PUBLIC_IP>/32` will block IAP. Do not retain the direct administrator-CIDR SSH
rule after confirming IAP recovery access.

The direct web and WebRTC ports still use the static public IP. A fully private VM would
require a different TURN/load-balancer/proxy architecture and is outside this direct
Neko design.

#### Reference: 10. Azure provisioning

##### Reference: 10.1 Cost and VM choice

Azure's eligible-new-account ARM offer includes B2pts v2 hours for 12 months, but that
SKU has only 1 GiB RAM and is below this project's normal minimum. A distinct AMD64
`Standard_B2ats_v2` 2-vCPU/1-GiB VM completed the sanitized Neko-only infrastructure
case study only with a reduced `1024x576@20` profile and 4 GiB of persistent swap. Do
not assume either 1-GiB SKU is free, comfortable, or suitable for desktop/combined use.
Use `Standard_B2pls_v2` (2 vCPU/4 GiB) for a constrained ARM test or
`Standard_B2ps_v2` (2 vCPU/8 GiB) for a practical ARM minimum.
Review the [Bpsv2 table](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/general-purpose/bpsv2-series),
[Azure free account](https://azure.microsoft.com/en-us/free/), and
[free service limits](https://azure.microsoft.com/en-us/pricing/free-services/).
Managed disk, snapshots, Standard public IP, bandwidth, DNS, and usage beyond an offer
can be billed. Create a Cost Management budget and alert; it does not stop resources.

The official ARM image URN is `Canonical:ubuntu-24_04-lts:server-arm64:<version>`.
ARM64 requires a compatible `p`-series size and currently uses `Standard` security type,
not Trusted Launch. Confirm current regional availability and the Canonical
[Azure Ubuntu image guidance](https://documentation.ubuntu.com/azure/azure-how-to/instances/find-ubuntu-images/).

##### Reference: 10.2 Resource group, VNet, NSG, and static IP

Use an authenticated Azure CLI. The example creates a dedicated resource group so the
deployment can be inventoried and, only when intentionally retired, removed as a unit:

```bash
set -euo pipefail
LOCATION='<AZURE_REGION>'
RESOURCE_GROUP='neko-cloud-rg'
ADMIN_CIDR='<ADMIN_PUBLIC_IP>/32'

az group create --name "$RESOURCE_GROUP" --location "$LOCATION"
az network vnet create --resource-group "$RESOURCE_GROUP" \
  --name neko-cloud-vnet --address-prefixes 10.40.0.0/16 \
  --subnet-name neko-cloud-public --subnet-prefixes 10.40.0.0/24

az network public-ip create --resource-group "$RESOURCE_GROUP" \
  --name neko-cloud-ip --sku Standard --allocation-method Static --version IPv4
az network nsg create --resource-group "$RESOURCE_GROUP" --name neko-cloud-nsg

az network nsg rule create --resource-group "$RESOURCE_GROUP" \
  --nsg-name neko-cloud-nsg --name AllowSshAdmin --priority 100 \
  --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefixes "$ADMIN_CIDR" --source-port-ranges '*' \
  --destination-address-prefixes '*' --destination-port-ranges 22
az network nsg rule create --resource-group "$RESOURCE_GROUP" \
  --nsg-name neko-cloud-nsg --name AllowPublicTcp --priority 110 \
  --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefixes Internet --source-port-ranges '*' \
  --destination-address-prefixes '*' --destination-port-ranges 80 443 59000
az network nsg rule create --resource-group "$RESOURCE_GROUP" \
  --nsg-name neko-cloud-nsg --name AllowWebRtcUdp --priority 120 \
  --direction Inbound --access Allow --protocol Udp \
  --source-address-prefixes Internet --source-port-ranges '*' \
  --destination-address-prefixes '*' --destination-port-ranges 59000

az network nic create --resource-group "$RESOURCE_GROUP" --name neko-cloud-nic \
  --vnet-name neko-cloud-vnet --subnet neko-cloud-public \
  --network-security-group neko-cloud-nsg --public-ip-address neko-cloud-ip
```

Azure NSGs are stateful. Their default deny handles all unlisted inbound ports. Do not
create 8080, 8444, or 3389 rules. Inspect the effective NSG after launch because subnet
and NIC NSGs both apply. See
[manage an NSG](https://learn.microsoft.com/en-us/azure/virtual-network/manage-network-security-group).

##### Reference: 10.2.1 Existing Azure VM and one-off CLI

For an existing VM, UFW and listening sockets can be correct while Azure still drops
ACME and WebRTC traffic. Discover whether the effective NSG is attached to the primary
NIC or its subnet before adding the exact rules above. Prefer Azure CLI on a trusted
workstation or Azure Cloud Shell.

If neither is available, Microsoft's official Azure CLI image can run temporarily on
the already trusted Docker host. Complete device authorization yourself, never record
the device code, account output, or token, and do not mount SSH keys, a persistent
`.azure` directory, or the Docker socket. Use `--rm`, run `az logout`, and remove
the exact unused CLI image after verifying the NSG.

```bash
# Host:
sudo docker run --rm -it mcr.microsoft.com/azure-cli:2.87.0-azurelinux3.0 sh

# Temporary container:
az login --use-device-code --output none
az account set --subscription '<SUBSCRIPTION_ID>'
az vm show -g '<VM_RESOURCE_GROUP>' -n '<VM_NAME>' \
  --query 'networkProfile.networkInterfaces[0].id' -o tsv
az network nic show --ids '<NIC_ID>' \
  --query '{nicNsg:networkSecurityGroup.id,subnet:ipConfigurations[0].subnet.id}' -o yaml
az network vnet subnet show --ids '<SUBNET_ID>' \
  --query 'networkSecurityGroup.id' -o tsv
```

Use the first nonempty NIC/subnet NSG, inspect existing priorities, and apply the two
rule definitions from section 10.2 to it. After verification, run `az logout`, exit the
container, and remove the exact unused image:

```bash
sudo docker image rm mcr.microsoft.com/azure-cli:2.87.0-azurelinux3.0
```

Re-test 80/443/59000 externally and confirm 8080/3389 remain closed. Caddy retries ACME
automatically; restarting only Caddy is acceptable when an immediate retry is needed.

##### Reference: 10.3 Resolve the image and launch

Resolve `latest` to a concrete version so the manifest is reproducible:

```bash
IMAGE_VERSION=$(az vm image list --location "$LOCATION" --all \
  --publisher Canonical --offer ubuntu-24_04-lts --sku server-arm64 \
  --query 'sort_by([], &version)[-1].version' --output tsv)
IMAGE_URN="Canonical:ubuntu-24_04-lts:server-arm64:$IMAGE_VERSION"
printf 'IMAGE_URN=%s\n' "$IMAGE_URN"

az vm create --resource-group "$RESOURCE_GROUP" --name neko-cloud \
  --nics neko-cloud-nic --image "$IMAGE_URN" --size Standard_B2ps_v2 \
  --admin-username azureuser \
  --ssh-key-values "$HOME/.ssh/id_ed25519.pub" \
  --os-disk-size-gb 80 --storage-sku Premium_LRS \
  --security-type Standard

PUBLIC_IP=$(az network public-ip show --resource-group "$RESOURCE_GROUP" \
  --name neko-cloud-ip --query ipAddress --output tsv)
printf 'PUBLIC_IP=%s\n' "$PUBLIC_IP"
az vm show --resource-group "$RESOURCE_GROUP" --name neko-cloud \
  --show-details --output yaml
ssh azureuser@"$PUBLIC_IP"
```

Use `azureuser` as `ADMIN_USER` in later commands, or deliberately create another user.
Do not assume every cloud's login name is `ubuntu`.

Azure changed new-VNet outbound behavior after 2026-03-31: workloads should use an
explicit outbound method. The NIC's Standard public IP in this design is explicit
inbound/outbound connectivity. If policy disallows it, use NAT Gateway for outbound and
redesign public ingress separately. See
[default outbound access](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/default-outbound-access)
and [public IP addresses](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/public-ip-addresses).

#### Reference: 11. Generic VPS or other cloud

For DigitalOcean, Hetzner, Linode/Akamai, Vultr, a private OpenStack cloud, or another
provider, reproduce the same invariants rather than copying a provider's button names:

1. Ubuntu Server 24.04 LTS on ARM64 or AMD64, 2 vCPU/8 GB minimum recommended.
2. A persistent 50+ GB disk and a stable public IPv4.
3. A default internet route and working outbound DNS/HTTPS/NTP.
4. A stateful cloud firewall permitting only the five rules in section 3.
5. SSH public-key login restricted to `<ADMIN_CIDR>`.
6. Provider snapshot/backup and cost alerts configured before application changes.
7. Reverse DNS is optional. Forward DNS or sslip.io is required by this runbook's Caddy
   workflow; the separate short-lived IP-certificate path described in section 5 is not
   implemented here.

Check the provider's current CPU architecture, egress, snapshot, public-IP, and stopped-
VM billing. Never infer â€œfreeâ€ from Ubuntu having no OS licence charge. If the image is
not an OCI platform image, follow the generic UFW section below; if it has provider-
specific firewall rules, preserve them and follow that provider's documentation.

#### Reference: 12. Prepare Ubuntu

##### Reference: 12.1 Record and update the host

SSH in as the provider's initial user, then record the immutable facts before changing
packages. Replace the variables with this deployment's values:

```bash
set -euo pipefail
ADMIN_USER=$(id -un)
ADMIN_HOME=$(getent passwd "$ADMIN_USER" | cut -d: -f6)
DESKTOP_USER=desktop
DESKTOP_HOME=/home/$DESKTOP_USER
KASM_WEB_USER='<KASM_WEB_USER>'
ARCH=$(dpkg --print-architecture)
PUBLIC_IP='<STATIC_PUBLIC_IPV4>'
NEKO_HOST='<NEKO_FQDN>'
DESKTOP_HOST='<DESKTOP_FQDN>'

case "$KASM_WEB_USER" in
  ''|*'<'*|*'>'*) echo 'Replace <KASM_WEB_USER> with a real web login name.' >&2; exit 1 ;;
esac

printf 'admin=%s home=%s desktop=%s desktop_home=%s arch=%s\n' \
  "$ADMIN_USER" "$ADMIN_HOME" "$DESKTOP_USER" "$DESKTOP_HOME" "$ARCH"
cat /etc/os-release
uname -a
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
free -h
ip -brief address
ip route
timedatectl
```

Ensure `PUBLIC_IP` matches the cloud console and DNS. Persist these **non-secret** values
so later sections and post-reboot shells do not silently use empty variables:

```bash
python3 - "$PUBLIC_IP" "$NEKO_HOST" "$DESKTOP_HOST" <<'PY'
import ipaddress
import re
import sys

ip_text, *hosts = sys.argv[1:]
try:
    ip = ipaddress.ip_address(ip_text)
except ValueError as exc:
    raise SystemExit(f"invalid PUBLIC_IP: {exc}")
if ip.version != 4 or not ip.is_global:
    raise SystemExit("PUBLIC_IP must be one globally routable IPv4 address")
label = re.compile(r"^(?!-)[A-Za-z0-9-]{1,63}(?<!-)$")
for host in hosts:
    if len(host) > 253 or "." not in host or host.endswith("."):
        raise SystemExit(f"invalid FQDN: {host!r}")
    if not all(label.fullmatch(part) for part in host.split(".")):
        raise SystemExit(f"invalid FQDN: {host!r}")
if hosts[0].lower() == hosts[1].lower():
    raise SystemExit("NEKO_HOST and DESKTOP_HOST must be different")
PY
printf 'ADMIN_USER=%q\nADMIN_HOME=%q\nDESKTOP_USER=%q\nDESKTOP_HOME=%q\nKASM_WEB_USER=%q\nARCH=%q\nPUBLIC_IP=%q\nNEKO_HOST=%q\nDESKTOP_HOST=%q\n' \
  "$ADMIN_USER" "$ADMIN_HOME" "$DESKTOP_USER" "$DESKTOP_HOME" \
  "$KASM_WEB_USER" "$ARCH" "$PUBLIC_IP" "$NEKO_HOST" "$DESKTOP_HOST" \
  | sudo tee /etc/neko-cloud.env >/dev/null
sudo chmod 0644 /etc/neko-cloud.env
sudo cat /etc/neko-cloud.env
```

The file contains identifiers and public endpoints only. Never add passwords, keys, or
cloud credentials. Then patch and reboot:

```bash
set -euo pipefail
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get dist-upgrade -y
sudo apt-get install -y ca-certificates curl gnupg jq openssl socat \
  dbus-x11 net-tools unzip unattended-upgrades
if [ "$(timedatectl show -p NTPSynchronized --value)" = yes ] || \
   systemctl is-active --quiet chrony.service || \
   systemctl is-active --quiet chronyd.service || \
   systemctl is-active --quiet systemd-timesyncd.service; then
  echo 'Keeping the active provider/default time service'
else
  sudo apt-get install -y systemd-timesyncd
  sudo systemctl enable --now systemd-timesyncd.service
fi
timedatectl show -p NTPSynchronized -p NTP
sudo reboot
```

Reconnect, source the deployment values, and confirm host health:

```bash
set -a
. /etc/neko-cloud.env
set +a
uptime
timedatectl
systemctl --failed --no-pager
printf '%s %s %s\n' "$PUBLIC_IP" "$NEKO_HOST" "$DESKTOP_HOST"
```

Run `. /etc/neko-cloud.env` at the start of each later SSH shell. Confirm the public IP.
Do not edit cloud-init, netplan, the active network renderer, or the provider agent merely
to install a desktop. A network change can strand the VM; make such a change only with
serial-console/boot-volume recovery prepared.

Create a locked-password, non-sudo OS account for the graphical session. This prevents
a stolen KasmVNC credential from inheriting the cloud administrator's usual passwordless
sudo access:

```bash
set -euo pipefail
. /etc/neko-cloud.env
[ "$DESKTOP_USER" != "$ADMIN_USER" ] || {
  echo 'The desktop and SSH administrator must be different accounts.' >&2; exit 1;
}
if ! id "$DESKTOP_USER" >/dev/null 2>&1; then
  sudo adduser --disabled-password --gecos '' "$DESKTOP_USER"
fi

# A pre-existing same-named account is reused only if it exactly matches the isolated
# account contract. Abort instead of silently adopting another human/service account.
IFS=: read -r _ _ ACTUAL_UID ACTUAL_GID _ ACTUAL_HOME ACTUAL_SHELL \
  <<<"$(getent passwd "$DESKTOP_USER")"
UID_MIN=$(awk '$1 == "UID_MIN" {print $2; exit}' /etc/login.defs)
PRIMARY_GROUP=$(getent group "$ACTUAL_GID" | cut -d: -f1)
[ "$ACTUAL_UID" -ge "$UID_MIN" ] && [ "$ACTUAL_HOME" = "$DESKTOP_HOME" ] && \
  [ "$PRIMARY_GROUP" = "$DESKTOP_USER" ] && \
  ! [[ "$ACTUAL_SHELL" =~ /(false|nologin)$ ]] && \
  grep -Fxq "$ACTUAL_SHELL" /etc/shells || {
    echo 'STOP: existing desktop account has unexpected UID/home/group/shell.' >&2
    getent passwd "$DESKTOP_USER" >&2
    exit 1
  }
[ -d "$DESKTOP_HOME" ] && \
  [ "$(stat -c %u:%g "$DESKTOP_HOME")" = "$ACTUAL_UID:$ACTUAL_GID" ] || {
    echo 'STOP: desktop home is missing or has unexpected user/group ownership.' >&2
    exit 1
  }
SUPPLEMENTARY_GROUPS=$(id -nG "$DESKTOP_USER" | tr ' ' '\n' | grep -vx "$PRIMARY_GROUP" || true)
[ -z "$SUPPLEMENTARY_GROUPS" ] || {
  printf 'STOP: desktop account has supplementary groups:\n%s\n' \
    "$SUPPLEMENTARY_GROUPS" >&2
  exit 1
}
if sudo test -s "$DESKTOP_HOME/.ssh/authorized_keys"; then
  echo 'STOP: desktop has SSH authorized_keys; remove and investigate before continuing.' >&2
  exit 1
fi

sudo passwd -l "$DESKTOP_USER"
sudo chmod 0750 "$DESKTOP_HOME"
id "$DESKTOP_USER"
SUDO_LIST=$(LC_ALL=C sudo -l -U "$DESKTOP_USER" 2>&1 || true)
printf '%s\n' "$SUDO_LIST"
if ! grep -Fq "User $DESKTOP_USER is not allowed to run sudo" <<<"$SUDO_LIST"; then
  echo 'STOP: the desktop account still has sudo policy; remove it before continuing' >&2
  exit 1
else
  echo 'Verified: the desktop account has no sudo policy'
fi
```

The final sudo query should show that `desktop` is not allowed to run sudo. Do not copy
the administrator's `authorized_keys` into this account. KasmVNC supplies its own web
authentication later; the Linux password can remain locked.

Create an operational directory and a root-readable build manifest containing no
passwords:

```bash
sudo install -d -m 0755 -o root -g root /opt/neko-cloud
sudo install -d -m 0750 -o root -g root /opt/neko-cloud-manifest
{
  date -u
  cat /etc/os-release
  uname -a
  printf 'ADMIN_USER=%s\nDESKTOP_USER=%s\nARCH=%s\nPUBLIC_IP=%s\nNEKO_HOST=%s\nDESKTOP_HOST=%s\n' \
    "$ADMIN_USER" "$DESKTOP_USER" "$ARCH" "$PUBLIC_IP" "$NEKO_HOST" "$DESKTOP_HOST"
} | sudo tee /opt/neko-cloud-manifest/base.txt >/dev/null
```

##### Reference: 12.2 Generic host firewall (not OCI platform Ubuntu)

On AWS, GCP, Azure, and ordinary VPS images, UFW may protect host-native listeners.
Skip this subsection on OCI Ubuntu and follow section 7.3 instead.

```bash
set -euo pipefail
sudo apt-get install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
SSH_SOURCE_CIDR='<ADMIN_PUBLIC_IP>/32'
# When GCP IAP from section 9 is selected, use: SSH_SOURCE_CIDR='35.235.240.0/20'
case "$SSH_SOURCE_CIDR" in *'<'*|'0.0.0.0/0')
  echo 'Set SSH_SOURCE_CIDR to the trusted admin /32 or the documented GCP IAP range.' >&2
  exit 1 ;;
esac
sudo ufw allow from "$SSH_SOURCE_CIDR" to any port 22 proto tcp comment 'SSH admin'
sudo ufw allow 80/tcp comment 'Caddy HTTP ACME'
sudo ufw allow 443/tcp comment 'Caddy HTTPS'
sudo ufw allow 59000/tcp comment 'Neko WebRTC TCP'
sudo ufw allow 59000/udp comment 'Neko WebRTC UDP'
sudo ufw --force enable
sudo ufw status numbered
```

Docker-published traffic can traverse rules before UFW's normal chains, so UFW is not a
substitute for the provider firewall. Publish only the ports shown later. Do not add
8080, 8444, or 3389. Docker explains this interaction in its
[firewall documentation](https://docs.docker.com/engine/network/packet-filtering-firewalls/).

##### Reference: 12.3 SSH and account safety

Keep the first SSH session open, open a second session successfully, and only then
disable password/root SSH. The KasmVNC password is independent of the Linux/SSH
password, so SSH can remain key-only.

```bash
set -euo pipefail
. /etc/neko-cloud.env
sudo install -d -m 0755 /etc/ssh/sshd_config.d
sudo tee /etc/ssh/sshd_config.d/00-neko-cloud-hardening.conf >/dev/null <<EOF
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
X11Forwarding no
AllowTcpForwarding yes
DenyUsers $DESKTOP_USER
EOF
sudo sshd -t
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|denyusers'
sudo systemctl reload ssh.service
```

`AllowTcpForwarding yes` retains SSH recovery/tunneling capability for administrators
with keys. Change it to `local` or `no` if that capability is not needed. Never add the
login user to the Docker group unless accepting that Docker group membership is
root-equivalent. Use `sudo docker ...` in this guide.

##### Reference: 12.4 Optional swap for small VMs

If RAM is 8 GB or less, a 4 GiB swap file can provide an out-of-memory safety margin.
Active swap is not necessarily persistent: inventory its source and verify that the
same source returns after reboot. Swap does not make a CPU-starved VM fast and consumes
disk:

```bash
set -euo pipefail

if swapon --noheadings --show=NAME | grep -Fxq /swapfile; then
  echo '/swapfile is already active; verify persistence below.'
elif [ -e /swapfile ]; then
  # swapon validates the swap signature; it refuses an ordinary data file.
  sudo chmod 0600 /swapfile
  sudo swapon /swapfile
elif swapon --noheadings --show=NAME | grep -q .; then
  echo 'Another swap source is active; do not add /swapfile blindly.'
  swapon --show
else
  sudo fallocate -l 4G /swapfile
  sudo chmod 0600 /swapfile
  sudo mkswap /swapfile
  sudo swapon /swapfile
fi

if swapon --noheadings --show=NAME | grep -Fxq /swapfile; then
  grep -Eq '^[[:space:]]*/swapfile[[:space:]]' /etc/fstab || \
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
  printf 'vm.swappiness=10\n' | sudo tee /etc/sysctl.d/90-neko-cloud-swap.conf
  sudo sysctl --load /etc/sysctl.d/90-neko-cloud-swap.conf
  sudo systemctl daemon-reload
  sudo findmnt --verify --tab-file /etc/fstab
fi

swapon --show
free -h
```

Never create a second swap source blindly or place the file on provider-local storage
that can be discarded. Reboot once, then require the configured source to be active:

```bash
sudo reboot
# Reconnect:
swapon --show
grep -E '^[[:space:]]*/swapfile[[:space:]]+none[[:space:]]+swap[[:space:]]' /etc/fstab
free -h
```

If `/swapfile` existed before this procedure but `swapon /swapfile` rejects it, stop
and identify the file; do not run `mkswap` over an unknown existing path.

##### Reference: 12.5 DNS preflight

Create real `A` records for both hostnames, or derive sslip.io names from the static IP:

```bash
set -euo pipefail
. /etc/neko-cloud.env
printf 'PUBLIC_IP=%s\nNEKO_HOST=%s\nDESKTOP_HOST=%s\n' \
  "$PUBLIC_IP" "$NEKO_HOST" "$DESKTOP_HOST"
for host in "$NEKO_HOST" "$DESKTOP_HOST"; do
  RESOLVED=$(getent ahostsv4 "$host" 2>/dev/null | awk '{print $1}' | LC_ALL=C sort -u || true)
  [ -n "$RESOLVED" ] || {
    echo "No public IPv4 result for $host" >&2; exit 1;
  }
  BAD=$(printf '%s\n' "$RESOLVED" | awk -v expected="$PUBLIC_IP" '$1 != expected')
  [ -z "$BAD" ] || {
    printf '%s resolves outside PUBLIC_IP %s:\n%s\n' "$host" "$PUBLIC_IP" "$BAD" >&2
    exit 1
  }
  printf '%s -> %s\n' "$host" "$RESOLVED"
done
```

Both must resolve publicly to exactly `PUBLIC_IP`. Remove incorrect `AAAA` records
unless IPv6 is fully routed and firewalled. ACME issuance will fail if DNS points
elsewhere or TCP 80/443 is blocked.

#### Reference: 13. Install Docker Engine and Compose

Use Docker's official APT repository, not Ubuntu's old `docker.io` package or Docker's
convenience script. These are Docker's current documented Ubuntu steps:

```bash
set -euo pipefail
EXISTING_RUNTIME_PACKAGES=$(
  for pkg in docker.io docker-ce docker-ce-cli containerd containerd.io runc \
             podman podman-docker docker-compose docker-compose-v2; do
    dpkg-query -W -f='${binary:Package} ${db:Status-Status}\n' "$pkg" 2>/dev/null || true
  done | awk '$2 == "installed" { print $1 }'
)
if command -v docker >/dev/null 2>&1 || [ -n "$EXISTING_RUNTIME_PACKAGES" ]; then
  echo 'STOP: existing container runtime/workloads require an inventory and migration plan' >&2
  printf 'Installed runtime packages:\n%s\n' "$EXISTING_RUNTIME_PACKAGES" >&2
  command -v docker >/dev/null 2>&1 && sudo docker ps -a
  exit 1
fi
# Fresh host only after the preflight above:
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt-get remove -y "$pkg" 2>/dev/null || true
done
sudo apt-get update
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

. /etc/os-release
CODENAME=${UBUNTU_CODENAME:-$VERSION_CODENAME}
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $CODENAME stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker.service containerd.service
sudo docker run --rm hello-world
sudo docker version
sudo docker compose version
```

Source: [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
and [install the Compose plugin](https://docs.docker.com/compose/install/linux/).

Do not modify an existing `/etc/docker/daemon.json` without merging its settings. On a
fresh host, bound container logs to protect the disk:

```bash
set -euo pipefail
if sudo test -e /etc/docker/daemon.json; then
  echo 'STOP: merge log settings into the existing /etc/docker/daemon.json' >&2
  exit 1
fi
sudo tee /etc/docker/daemon.json >/dev/null <<'JSON'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true
}
JSON
sudo dockerd --validate --config-file=/etc/docker/daemon.json
sudo systemctl restart docker.service
sudo docker info
```

If the `test` command fails, stop and merge the keys using `jq` or a careful editor;
do not overwrite registry mirrors, storage drivers, runtimes, or provider settings.
Keep using `sudo docker`. Docker's
[post-install documentation](https://docs.docker.com/engine/install/linux-postinstall/)
explains why the `docker` group grants root-level privileges.

Record installed versions:

```bash
sudo docker version --format '{{json .Server}}' | sudo tee /opt/neko-cloud-manifest/docker-server.json >/dev/null
sudo docker compose version | sudo tee /opt/neko-cloud-manifest/docker-compose.txt >/dev/null
apt-cache policy docker-ce containerd.io | sudo tee /opt/neko-cloud-manifest/docker-apt.txt >/dev/null
```

#### Reference: 14. Deploy Neko and Caddy

##### Reference: 14.1 Create secrets and environment

Generate three **different** passwords: Neko administrator, Neko regular user, and
KasmVNC write user. Never reuse a credential previously posted in chat; rotate it now.
Generate locally or on the VM in a private shell, save it in a password manager, and do
not paste it into shell history. The following generates the two Neko values without
putting them on the command line; KasmVNC prompts for its separate value in section 15:

```bash
sudo -i
umask 077
cd /opt/neko-cloud
. /etc/neko-cloud.env
NEKO_USER_PASSWORD=$(openssl rand -hex 24)
NEKO_ADMIN_PASSWORD=$(openssl rand -hex 24)
printf 'PUBLIC_IP=%s\nNEKO_HOST=%s\nDESKTOP_HOST=%s\nNEKO_USER_PASSWORD=%s\nNEKO_ADMIN_PASSWORD=%s\n' \
  "$PUBLIC_IP" "$NEKO_HOST" "$DESKTOP_HOST" \
  "$NEKO_USER_PASSWORD" "$NEKO_ADMIN_PASSWORD" > .env
chmod 0600 .env
printf 'Store these now in a password manager.\nUser: %s\nAdmin: %s\n' \
  "$NEKO_USER_PASSWORD" "$NEKO_ADMIN_PASSWORD"
unset NEKO_USER_PASSWORD NEKO_ADMIN_PASSWORD
exit
```

Clear the visible screen **and the terminal application's scrollback** after saving them;
the shell `clear` command alone does not erase scrollback, terminal recordings, audit
logs, or chat history. `.env` is readable by root and can be inspected
by anyone with Docker/root privileges; it is not a secret vault. Do not place it in
Git, a VM image shared with others, or an unencrypted backup. Compose secrets are
preferable when the application natively supports file-based secrets; these Neko
variables do not become secret merely because Compose performs interpolation.

Create a safe example for backups/source control:

```bash
sudo tee /opt/neko-cloud/.env.example >/dev/null <<'EOF'
PUBLIC_IP=203.0.113.10
NEKO_HOST=neko.example.com
DESKTOP_HOST=desktop.example.com
NEKO_USER_PASSWORD=replace-with-a-long-random-password
NEKO_ADMIN_PASSWORD=replace-with-a-different-long-random-password
EOF
sudo chmod 0644 /opt/neko-cloud/.env.example
sudo test "$(sudo stat -c %a /opt/neko-cloud/.env)" = 600
```

The two Neko roles are password-based. The login display name is chosen at sign-in and
is not the Kasm web user or an OS account. Use the administrator
password only when control/admin capability is needed. Do not reuse it for KasmVNC.

##### Reference: 14.2 Choose and pin the Neko image

This guide uses `ghcr.io/m1k1o/neko/firefox:3.1.4`, supported on AMD64 and ARM64.
Google Chrome is AMD64-only. Chromium works on both but Neko documents no DRM on ARM,
a 2 GiB shared-memory recommendation, and `--no-sandbox` inside the container. Refer to
the [official image matrix](https://neko.m1k1o.net/docs/v3/installation/docker-images).

A version tag is repeatable only until a registry tag is changed. Pull it once, record
the immutable digest, and use `image@sha256:...` in a high-assurance rebuild:

```bash
sudo docker pull ghcr.io/m1k1o/neko/firefox:3.1.4
sudo docker image inspect ghcr.io/m1k1o/neko/firefox:3.1.4 \
  --format '{{index .RepoDigests 0}}' \
  | sudo tee /opt/neko-cloud-manifest/neko-image.txt >/dev/null
sudo cat /opt/neko-cloud-manifest/neko-image.txt
```

The supplied Compose templates intentionally have no persistent browser-profile volume. Open
tabs, extensions, cookies, and downloads inside that container can disappear when it is
recreated. Keep it stateless for safer shared use, or design a separately backed-up,
encrypted profile mount only after verifying the exact path for the chosen image.

##### Reference: 14.3 Create `compose.yaml`

The mode-0600 `.env` supplies the public address, two hostnames, and passwords. The
Compose and Caddy templates therefore stay portable and contain no rendered endpoint.

```bash
sudo install -d -m 0755 /run/kasmvnc
sudo tee /opt/neko-cloud/compose.yaml >/dev/null <<'YAML'
name: neko-and-ubuntu-desktop

services:
  neko:
    image: ghcr.io/m1k1o/neko/firefox:3.1.4
    restart: unless-stopped
    init: true
    shm_size: '2gb'
    environment:
      NEKO_DESKTOP_SCREEN: '1280x720@24'
      NEKO_MEMBER_PROVIDER: 'multiuser'
      NEKO_MEMBER_MULTIUSER_USER_PASSWORD: '${NEKO_USER_PASSWORD:?Set NEKO_USER_PASSWORD in .env}'
      NEKO_MEMBER_MULTIUSER_ADMIN_PASSWORD: '${NEKO_ADMIN_PASSWORD:?Set NEKO_ADMIN_PASSWORD in .env}'
      NEKO_WEBRTC_UDPMUX: '59000'
      NEKO_WEBRTC_TCPMUX: '59000'
      NEKO_WEBRTC_ICELITE: 'true'
      NEKO_WEBRTC_NAT1TO1: '${PUBLIC_IP:?Set PUBLIC_IP in .env}'
      NEKO_SERVER_PROXY: 'true'
      NEKO_LEGACY: 'true'
      NEKO_SESSION_COOKIE_ENABLED: 'false'
    expose:
      - '8080'
    ports:
      - '59000:59000/udp'
      - '59000:59000/tcp'
    networks:
      - frontend
    logging:
      driver: json-file
      options:
        max-size: '10m'
        max-file: '3'

  caddy:
    image: caddy:2.11.4-alpine
    restart: unless-stopped
    depends_on:
      - neko
    environment:
      NEKO_HOST: '${NEKO_HOST:?Set NEKO_HOST in .env}'
      DESKTOP_HOST: '${DESKTOP_HOST:?Set DESKTOP_HOST in .env}'
      PUBLIC_IP: '${PUBLIC_IP:?Set PUBLIC_IP in .env}'
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - /run/kasmvnc:/run/kasmvnc:ro
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - frontend
    logging:
      driver: json-file
      options:
        max-size: '10m'
        max-file: '3'

networks:
  frontend:

volumes:
  caddy_data:
  caddy_config:
YAML
sudo chmod 0644 /opt/neko-cloud/compose.yaml
```

Why these details matter:

- `NEKO_SERVER_PROXY=true` is required behind Caddy.
- UDP and TCP mux share 59000, matching the cloud firewall and `nat1to1` address.
- `expose: 8080` is Docker-network metadata, not a host publication.
- The legacy/token settings match the Neko 3.1.4 bundled client and prevent the prior
  `token not found`/cookie-auth mismatch. Re-evaluate them when upgrading Neko.
- Caddy data/config volumes persist ACME account and certificate state.
- Caddy only orders after Neko; `depends_on` is not a readiness check. Validation later
  tests both services.

Validate interpolation without printing resolved secrets. `config --quiet` parses and
validates without rendering the environment values:

```bash
cd /opt/neko-cloud
sudo docker compose config --quiet
sudo docker compose config --services
```

Do not run plain `docker compose config` in a recorded/shared terminal because it can
render resolved passwords.

##### Reference: 14.4 Create the Caddyfile

The Caddyfile uses environment placeholders supplied by Compose. `admin off` removes
Caddy's local admin API; configuration changes therefore require a container
restart/recreation. Restricting protocols to `h1 h2` disables HTTP/3, so UDP 443 is not
required.

```bash
cd /opt/neko-cloud
. /etc/neko-cloud.env
sudo tee Caddyfile >/dev/null <<'EOF'
{
	admin off
	servers {
		protocols h1 h2
	}
}

{$NEKO_HOST} {
	encode zstd gzip
	reverse_proxy neko:8080
	header {
		Strict-Transport-Security "max-age=31536000"
		X-Content-Type-Options "nosniff"
		X-Frame-Options "SAMEORIGIN"
		Referrer-Policy "no-referrer"
	}
}

{$DESKTOP_HOST} {
	reverse_proxy unix//run/kasmvnc/kasm.sock {
		header_up Host 127.0.0.1:8444
		flush_interval -1
	}
	header {
		Strict-Transport-Security "max-age=31536000"
		X-Content-Type-Options "nosniff"
		X-Frame-Options "SAMEORIGIN"
		Referrer-Policy "no-referrer"
	}
}

http://{$PUBLIC_IP} {
	redir https://{$NEKO_HOST}{uri} permanent
}
EOF
sudo chmod 0644 Caddyfile

sudo docker run --rm \
  -e NEKO_HOST="$NEKO_HOST" -e DESKTOP_HOST="$DESKTOP_HOST" -e PUBLIC_IP="$PUBLIC_IP" \
  -v /opt/neko-cloud/Caddyfile:/etc/caddy/Caddyfile:ro \
  caddy:2.11.4-alpine caddy validate --config /etc/caddy/Caddyfile
```

The quoted heredoc keeps Caddy's environment placeholders literal; Compose provides
their public, non-secret values to the Caddy container. Inspect the file before
starting. Do not put passwords in a Caddyfile.

Pull, record, and pin the exact image digests before first production start:

```bash
set -euo pipefail
cd /opt/neko-cloud
sudo docker compose pull
NEKO_REF=$(sudo docker image inspect ghcr.io/m1k1o/neko/firefox:3.1.4 \
  --format '{{index .RepoDigests 0}}')
CADDY_REF=$(sudo docker image inspect caddy:2.11.4-alpine \
  --format '{{index .RepoDigests 0}}')
for ref in "$NEKO_REF" "$CADDY_REF"; do
  [[ "$ref" =~ ^[^[:space:]]+@sha256:[0-9a-f]{64}$ ]] || {
    printf 'Invalid immutable image reference: %q\n' "$ref" >&2; exit 1;
  }
done
[ "$(grep -Fxc '    image: ghcr.io/m1k1o/neko/firefox:3.1.4' compose.yaml)" -eq 1 ]
[ "$(grep -Fxc '    image: caddy:2.11.4-alpine' compose.yaml)" -eq 1 ]
printf '%s\n' "$NEKO_REF" | sudo tee /opt/neko-cloud-manifest/neko-image.txt >/dev/null
printf '%s\n' "$CADDY_REF" | sudo tee /opt/neko-cloud-manifest/caddy-image.txt >/dev/null
sudo sed -i "s|image: ghcr.io/m1k1o/neko/firefox:3.1.4|image: $NEKO_REF|" compose.yaml
sudo sed -i "s|image: caddy:2.11.4-alpine|image: $CADDY_REF|" compose.yaml
grep -Fqx "    image: $NEKO_REF" compose.yaml
grep -Fqx "    image: $CADDY_REF" compose.yaml
sudo docker compose config --quiet
```

The digest is platform-specific and makes subsequent rebuilds deterministic. Upgrades
must deliberately replace it after testing, not silently follow a floating tag.

Start the container half of the stack:

```bash
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs --tail=100 neko caddy
sudo ss -lntup | grep -E ':(80|443|59000)\b'
if curl -fsS --connect-timeout 2 http://127.0.0.1:8080/ >/dev/null; then
  echo 'ERROR: host port 8080 is unexpectedly reachable' >&2
  exit 1
else
  echo 'Host port 8080 is correctly not published'
fi
```

Test the backend from Caddy's Docker network instead:

```bash
sudo docker compose exec caddy wget -qO- http://neko:8080/ >/dev/null
```

Do not â€œfixâ€ the first curl by publishing 8080. Neko's official
[installation](https://neko.m1k1o.net/docs/v3/installation),
[configuration](https://neko.m1k1o.net/docs/v3/configuration), and
[reverse-proxy guide](https://neko.m1k1o.net/docs/v3/reverse-proxy-setup) are the
authoritative upstream references.

#### Reference: 15. Install Ubuntu GNOME and KasmVNC

##### Reference: 15.1 Install the Ubuntu desktop without a physical display manager

Take a provider snapshot first if this VM already contains important data. Keep SSH and
serial-console recovery available. Install the minimal official Ubuntu desktop, but keep
the headless VM's default target non-graphical and do not run GDM:

```bash
sudo systemctl set-default multi-user.target
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y \
  ubuntu-desktop-minimal ubuntu-session gnome-shell gnome-terminal nautilus \
  yaru-theme-gtk yaru-theme-icon dbus-x11 x11-xserver-utils xauth xdg-user-dirs
sudo systemctl disable --now gdm3.service 2>/dev/null || true
sudo systemctl mask gdm3.service
systemctl get-default
systemctl is-enabled gdm3.service || true
```

This installs the familiar GNOME/Yaru/Ubuntu Dock experience. KasmVNC supplies the X11
display, so GDM/Wayland is neither needed nor desired on this headless design. Do not
alter netplan just because `ubuntu-desktop-minimal` installed NetworkManager packages.

If XFCE was installed during an earlier attempt, retain it until GNOME works and a
snapshot exists. It is a useful rollback session and normally costs disk, not runtime
CPU, while inactive. A fresh build does not need XFCE or XRDP.

##### Reference: 15.2 Install the verified KasmVNC package

KasmVNC's current installation matrix supports Ubuntu Noble on AMD64 and ARM64. This
runbook pins release 1.4.0 and verifies the SHA-256 digest published by GitHub's official
release API before installation:

```bash
set -euo pipefail
ARCH=$(dpkg --print-architecture)
case "$ARCH" in
  arm64)
    KASM_URL='https://github.com/kasmtech/KasmVNC/releases/download/v1.4.0/kasmvncserver_noble_1.4.0_arm64.deb'
    KASM_SHA256='120d9462cb5e917cad91a23f6cb0b780c06f701def40e900b29f996979200638'
    ;;
  amd64)
    KASM_URL='https://github.com/kasmtech/KasmVNC/releases/download/v1.4.0/kasmvncserver_noble_1.4.0_amd64.deb'
    KASM_SHA256='12bac6014149c5fdee75f0d403785aaa3e5dd4ea222de73253a5d4181bc9567e'
    ;;
  *) echo "Unsupported architecture: $ARCH" >&2; exit 1 ;;
esac

KASM_DEB=$(mktemp --suffix=.deb)
trap 'rm -f -- "$KASM_DEB"' EXIT HUP INT TERM
curl -fL "$KASM_URL" -o "$KASM_DEB"
printf '%s  %s\n' "$KASM_SHA256" "$KASM_DEB" | sha256sum -c -
sudo apt-get install -y "$KASM_DEB"
rm -f -- "$KASM_DEB"
trap - EXIT HUP INT TERM
sudo loginctl enable-linger "$DESKTOP_USER"
DESKTOP_UID=$(id -u "$DESKTOP_USER")
sudo systemctl start "user@$DESKTOP_UID.service"
loginctl show-user "$DESKTOP_USER" -p Linger -p State -p RuntimePath
dpkg-query -W kasmvncserver
```

Do not add this internet-controlled desktop account to `sudo`, `docker`, `lxd`, or
`ssl-cert`. This design terminates public TLS in Caddy and gives KasmVNC a private,
home-owned placeholder certificate, so access to the system certificate group is not
needed. Sources:
[KasmVNC installation](https://www.kasmweb.com/kasmvnc/docs/latest/install.html) and
[KasmVNC 1.4.0 release](https://github.com/kasmtech/KasmVNC/releases/tag/v1.4.0).

##### Reference: 15.3 Create the KasmVNC write-user credential

From the SSH administrator shell, enter the non-sudo desktop user's shell and connect it
to that lingering user manager:

```bash
sudo -iu "$DESKTOP_USER"
. /etc/neko-cloud.env
export XDG_RUNTIME_DIR=/run/user/$(id -u)
export DBUS_SESSION_BUS_ADDRESS=unix:path=$XDG_RUNTIME_DIR/bus
test -S "$XDG_RUNTIME_DIR/bus"
```

The remaining commands through the explicit `exit` near the end of section 15.4 run in
this `desktop` shell. Create a write-capable Kasm user; owner/user-management permission
is not needed for ordinary remote control:

```bash
vncpasswd -u "$KASM_WEB_USER" -w
chmod 0600 "$HOME/.kasmpasswd"
stat -c '%a %U:%G %n' "$HOME/.kasmpasswd"
```

`-w` grants keyboard/mouse control. Add `-o` only if web-based Kasm user management is a
real requirement; it is not required by this build.
KasmVNC explicitly warns that `.kasmpasswd` is obfuscated, not securely encrypted, and
can be trivially reversed by anyone who obtains it. Treat it as plaintext-equivalent,
keep mode 0600, use a unique password, and exclude it from ordinary Git/backups. See
[`vncpasswd`](https://kasmweb.com/kasmvnc/docs/master/man/vncpasswd.html).

##### Reference: 15.4 Configure a private, persistent KasmVNC session

As the desktop user, render the actual home path into the YAML:

```bash
install -d -m 0700 "$HOME/.vnc"
openssl req -x509 -nodes -newkey rsa:2048 -days 3650 \
  -subj '/CN=localhost' \
  -keyout "$HOME/.vnc/kasmvnc.key" \
  -out "$HOME/.vnc/kasmvnc.crt"
chmod 0600 "$HOME/.vnc/kasmvnc.key" "$HOME/.vnc/kasmvnc.crt"

cat > "$HOME/.vnc/kasmvnc.yaml" <<EOF
desktop:
  resolution:
    width: 1280
    height: 720
  allow_resize: true
  pixel_depth: 24

network:
  protocol: http
  interface: 127.0.0.1
  websocket_port: 8444
  use_ipv4: true
  use_ipv6: false
  udp:
    public_ip: 127.0.0.1
    port: 8444
  ssl:
    pem_certificate: $HOME/.vnc/kasmvnc.crt
    pem_key: $HOME/.vnc/kasmvnc.key
    require_ssl: false

user_session:
  session_type: exclusive
  new_session_disconnects_existing_exclusive_session: true
  concurrent_connections_prompt: false
  idle_timeout: 1800

runtime_configuration:
  allow_client_to_override_kasm_server_settings: false

encoding:
  max_frame_rate: 30

server:
  auto_shutdown:
    no_user_session_timeout: never
    active_user_session_timeout: never
    inactive_user_session_timeout: never

command_line:
  prompt: false
EOF
chmod 0600 "$HOME/.vnc/kasmvnc.yaml"
```

The endpoint is deliberately HTTP on loopback because Caddy supplies public TLS. One
exclusive browser connection is active; a new login disconnects the old viewer while
the desktop itself persists. Viewers idle for 30 minutes disconnect. The desktop never
auto-shuts down. Kasm's WebSocket TCP port is loopback-only; its UDP behavior must still
be blocked by both provider and host firewalls.

Create the GNOME startup script:

```bash
cat > "$HOME/.vnc/xstartup" <<'SH'
#!/bin/sh
set -eu

unset SESSION_MANAGER
unset WAYLAND_DISPLAY

export XDG_CONFIG_HOME=${XDG_CONFIG_HOME:-${HOME}/.config}
export XDG_DATA_HOME=${XDG_DATA_HOME:-${HOME}/.local/share}
export XDG_CACHE_HOME=${XDG_CACHE_HOME:-${HOME}/.cache}
export XDG_CONFIG_DIRS=/etc/xdg/xdg-ubuntu:${XDG_CONFIG_DIRS:-/etc/xdg}
export XDG_DATA_DIRS=/usr/share/ubuntu:/usr/share/gnome:${XDG_DATA_DIRS:-/usr/local/share:/usr/share}:/var/lib/snapd/desktop
export XDG_SESSION_TYPE=x11
export XDG_SESSION_CLASS=user
export DESKTOP_SESSION=ubuntu
export XDG_SESSION_DESKTOP=ubuntu
export XDG_CURRENT_DESKTOP=ubuntu:GNOME
export GNOME_SHELL_SESSION_MODE=ubuntu
export GDMSESSION=ubuntu
export XAUTHORITY=${XAUTHORITY:-${HOME}/.Xauthority}

mkdir -p "$XDG_CONFIG_HOME" "$XDG_DATA_HOME" "$XDG_CACHE_HOME"

if [ -z "${XDG_RUNTIME_DIR:-}" ] || [ ! -S "${XDG_RUNTIME_DIR}/bus" ]; then
  echo 'GNOME requires the systemd user runtime directory and user D-Bus' >&2
  exit 1
fi

export DBUS_SESSION_BUS_ADDRESS=unix:path=${XDG_RUNTIME_DIR}/bus
dbus-update-activation-environment --systemd \
  DISPLAY XAUTHORITY DBUS_SESSION_BUS_ADDRESS \
  XDG_SESSION_TYPE XDG_SESSION_CLASS DESKTOP_SESSION XDG_SESSION_DESKTOP \
  XDG_CURRENT_DESKTOP GNOME_SHELL_SESSION_MODE GDMSESSION \
  XDG_CONFIG_DIRS XDG_DATA_DIRS

exec /usr/bin/gnome-session --session=ubuntu
SH
chmod 0755 "$HOME/.vnc/xstartup"
```

Create a portable **user** systemd template. `%t` resolves to this user's runtime
directory, avoiding the non-portable `/run/user/1001` hardcoding from the first build:

```bash
install -d -m 0700 "$HOME/.config/systemd/user"
cat > "$HOME/.config/systemd/user/kasmvncserver@.service" <<'UNIT'
[Unit]
Description=KasmVNC Ubuntu desktop on display %i
After=network.target

[Service]
Type=forking
Environment=XDG_RUNTIME_DIR=%t
Environment=DBUS_SESSION_BUS_ADDRESS=unix:path=%t/bus
Environment=XDG_CONFIG_DIRS=/etc/xdg/xdg-ubuntu:/etc/xdg
Environment=XDG_DATA_DIRS=/usr/share/ubuntu:/usr/share/gnome:/usr/local/share:/usr/share:/var/lib/snapd/desktop
Environment=XDG_SESSION_TYPE=x11
Environment=XDG_SESSION_CLASS=user
Environment=DESKTOP_SESSION=ubuntu
Environment=XDG_SESSION_DESKTOP=ubuntu
Environment=XDG_CURRENT_DESKTOP=ubuntu:GNOME
Environment=GNOME_SHELL_SESSION_MODE=ubuntu
Environment=GDMSESSION=ubuntu
ExecStartPre=-/usr/bin/kasmvncserver -kill %i
ExecStart=/usr/bin/kasmvncserver %i
ExecStop=/usr/bin/kasmvncserver -kill %i
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=default.target
UNIT

# This is a permanently headless personal session. Prevent an idle GNOME lock screen
# that could not be unlocked with the separate Kasm credential, and prevent suspend.
gsettings set org.gnome.desktop.session idle-delay 'uint32 0'
gsettings set org.gnome.desktop.screensaver lock-enabled false
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'

systemctl --user daemon-reload
systemctl --user enable 'kasmvncserver@:1.service'
loginctl show-user "$USER" -p Linger -p State -p RuntimePath
exit
```

Linger starts the user's systemd manager at boot even without SSH login; it does not
make the Kasm password a Linux login password. Disabling GNOME's idle lock is deliberate
for this locked, non-sudo, virtual-only account. Do not copy that policy to a physical
workstation or a shared Linux account.

##### Reference: 15.5 Preserve the GNOME handover service used by this build

On the tested Ubuntu 24.04 GNOME X11 stack, the system
`gnome-remote-desktop.service` handover component is needed for the virtual GNOME
session to remain alive even though RDP itself is not configured. Stopping it terminated
the desktop during testing. Keep the **system** service enabled, while keeping TCP 3389
closed at every firewall:

```bash
sudo apt-get install -y gnome-remote-desktop
sudo systemctl unmask gnome-remote-desktop.service
sudo systemctl enable --now gnome-remote-desktop.service
sudo systemctl status gnome-remote-desktop.service --no-pager
if sudo ss -lntup | grep -qE ':(3389)\b'; then
  echo 'ERROR: investigate unexpected RDP listener before continuing' >&2
  exit 1
fi
```

Install a narrow operator wrapper so the SSH administrator can manage the dedicated
desktop user's lingering service without pretending the administrator owns that user
manager:

```bash
sudo tee /usr/local/sbin/neko-cloud-desktop >/dev/null <<'SH'
#!/bin/sh
set -eu
[ "$(id -u)" -eq 0 ] || {
  echo 'run with sudo' >&2
  exit 1
}
. /etc/neko-cloud.env
desktop_uid=$(id -u "$DESKTOP_USER")
tool=${1:-}
[ "$#" -gt 0 ] && shift
case "$tool" in
  systemctl) command=/usr/bin/systemctl ;;
  journalctl) command=/usr/bin/journalctl ;;
  *) echo 'usage: neko-cloud-desktop systemctl|journalctl [arguments...]' >&2; exit 2 ;;
esac
exec /usr/sbin/runuser -u "$DESKTOP_USER" -- /usr/bin/env \
  HOME="$DESKTOP_HOME" USER="$DESKTOP_USER" LOGNAME="$DESKTOP_USER" \
  XDG_RUNTIME_DIR="/run/user/$desktop_uid" \
  DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$desktop_uid/bus" \
  "$command" --user "$@"
SH
sudo chown root:root /usr/local/sbin/neko-cloud-desktop
sudo chmod 0755 /usr/local/sbin/neko-cloud-desktop
sudo /usr/local/sbin/neko-cloud-desktop systemctl start 'kasmvncserver@:1.service'
```

Do not generalize this into enabling GNOME RDP credentials or opening 3389. On a later
Ubuntu/GNOME version, test session persistence before deciding whether the handover unit
is still needed, and document any change.

Verify the private listener and session:

```bash
sudo ss -lntup | grep -E '127\.0\.0\.1:8444|:8444'
pgrep -af -u "$DESKTOP_USER" 'Xkasmvnc|Xvnc|gnome-shell|gnome-session'
sudo neko-cloud-desktop systemctl status 'kasmvncserver@:1.service' --no-pager
sudo neko-cloud-desktop journalctl -u 'kasmvncserver@:1.service' -n 100 --no-pager
sudo find "$DESKTOP_HOME/.vnc" -maxdepth 1 -name '*.log' \
  -exec tail -n 100 {} +
```

TCP 8444 must bind only to 127.0.0.1. If a UDP 8444 socket is broader, do not expose it
in the provider firewall and verify the host INPUT policy blocks it externally.

#### Reference: 16. Connect KasmVNC privately to Caddy

The Caddy container cannot directly reach the host loopback address. A small host
service bridges a Unix socket mounted into Caddy to KasmVNC's loopback TCP port. Create
this system unit exactly:

```bash
sudo tee /etc/systemd/system/kasmvnc-socat.service >/dev/null <<'UNIT'
[Unit]
Description=Private Unix-socket bridge from Caddy to KasmVNC
After=network.target

[Service]
Type=simple
ExecStartPre=/usr/bin/install -d -m 0755 /run/kasmvnc
ExecStartPre=-/usr/bin/rm -f /run/kasmvnc/kasm.sock
ExecStart=/usr/bin/socat UNIX-LISTEN:/run/kasmvnc/kasm.sock,fork,mode=0660,user=root,group=root TCP:127.0.0.1:8444
ExecStopPost=-/usr/bin/rm -f /run/kasmvnc/kasm.sock
Restart=always
RestartSec=2s

[Install]
WantedBy=multi-user.target
UNIT

sudo systemd-analyze verify /etc/systemd/system/kasmvnc-socat.service
sudo systemctl daemon-reload
sudo systemctl enable --now kasmvnc-socat.service
sudo systemctl status kasmvnc-socat.service --no-pager
sudo stat -c '%a %U:%G %F %n' /run/kasmvnc/kasm.sock
sudo curl --unix-socket /run/kasmvnc/kasm.sock -I http://localhost/
```

An HTTP `401 Unauthorized` from the last command is success: KasmVNC is reachable and
requesting its own Basic-auth credential. `Connection refused` means KasmVNC is down;
`Permission denied` means socket/container ownership no longer matches. The current
Caddy image runs sufficiently privileged to connect to a root-owned mode-0660 socket.
If Caddy is later made non-root or user-namespace remapping is enabled, create a shared
group and redesign ownership rather than making the socket world-writable.

Restart Caddy so its bind mount sees the live runtime directory and validate both routes:

```bash
cd /opt/neko-cloud
sudo docker compose up -d --force-recreate caddy
sudo docker compose ps
sudo docker compose logs --tail=100 caddy
curl -I "https://$NEKO_HOST"
curl -I "https://$DESKTOP_HOST"
```

Expect a successful Neko response and `401` from the desktop until Kasm credentials are
supplied. Caddy preserves WebSocket upgrades automatically.

#### Reference: 17. Start everything and validate end to end

##### Reference: 17.1 Reboot test

An installation is not complete until it survives a reboot:

```bash
sudo reboot
```

Reconnect after the provider reports the VM running, then run:

```bash
systemctl --failed --no-pager
systemctl is-active docker.service kasmvnc-socat.service gnome-remote-desktop.service
sudo neko-cloud-desktop systemctl is-active 'kasmvncserver@:1.service'
cd /opt/neko-cloud && sudo docker compose ps
sudo docker inspect --format '{{.Name}} restarts={{.RestartCount}} oom={{.State.OOMKilled}}' \
  $(sudo docker compose ps -q)
sudo docker stats --no-stream
sudo ss -lntup
df -hT
swapon --show
free -h
```

Every required service should be active; Caddy and Neko should be `Up`; disk and memory
must have safe headroom; no container should report an OOM kill or unexplained restart.
Any configured swap source must be active after reboot. Confirm 8080 and TCP 8444 are
not bound to a public interface.

##### Reference: 17.2 External network and TLS tests

Run from a machine outside the cloud VPC, ideally a second network. On Windows:

```powershell
$Ip = '<PUBLIC_IP>'
$NekoHost = '<NEKO_FQDN>'
$DesktopHost = '<DESKTOP_FQDN>'
22,80,443,59000,5901,8080,8444,3389,631,111,1194 | ForEach-Object {
  Test-NetConnection -ComputerName $Ip -Port $_ -InformationLevel Detailed |
    Select-Object ComputerName,RemotePort,TcpTestSucceeded
}
curl.exe -I "http://$NekoHost"
curl.exe -I "https://$NekoHost"
curl.exe -I "https://$DesktopHost"
```

Expected TCP result from the administrator CIDR:

- Open: 22, 80, 443, 59000.
- Closed/filtered: 5901, 8080, 8444, 3389, 631, 111, 1194.
- From any other CIDR, 22 should also be closed/filtered.

TCP testing does not prove UDP 59000. From Linux/macOS with Nmap available:

```bash
PUBLIC_IP='<STATIC_PUBLIC_IPV4>'
nmap -Pn -sT -p 22,80,443,59000,5901,8080,8444,3389,631,111,1194 "$PUBLIC_IP"
sudo nmap -Pn -sU -p 59000,8444 "$PUBLIC_IP"
```

UDP scans can report `open|filtered`; an actual Neko media session is the definitive
test. UDP 59000 must work and UDP 8444 must not be provider-allowed.

Inspect both certificates and require a successful chain verification:

```bash
set -euo pipefail
for host in '<NEKO_FQDN>' '<DESKTOP_FQDN>'; do
  printf '\n' | openssl s_client -connect "$host:443" -servername "$host" \
    -verify_return_error -verify_hostname "$host" 2>/dev/null \
    | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
done
```

Test this configured deployment with its hostnames; its certificates are for those
hostnames. The optional short-lived IP-certificate design in section 5 is not configured.

##### Reference: 17.3 Functional acceptance test

Neko:

1. Open the Neko HTTPS URL in local Chrome/Edge/Firefox.
2. Enter a display name and the regular-user password; confirm video and audio arrive.
3. Enter with the admin password; claim/unlock control and test keyboard, mouse, URL
   navigation, a second tab, sound, and clipboard policy.
4. Test from a second internet connection. Confirm WebRTC establishes; if UDP is
   blocked by that network, confirm TCP 59000 fallback still connects.
5. Recreate only the Neko container and confirm the session is intentionally stateless.

KasmVNC/GNOME:

1. Open the desktop HTTPS URL and authenticate as the configured `KASM_WEB_USER` with
   its unique Kasm password.
2. Confirm Ubuntu Dock, Activities, Files, Settings, terminal, keyboard, mouse,
   clipboard, fullscreen, and display resize. This exact standalone KasmVNC path does
   not configure a host-audio transport, so GNOME desktop sound is not promised; Neko
   audio is the separately tested streaming-audio path.
3. Open an application, close the local browser tab without logging out, then reconnect;
   the application should still be open.
4. Connect from a second browser. It should replace the old exclusive viewer, not spawn
   a second GNOME desktop.
5. Leave a viewer idle for 30 minutes and confirm only the viewer disconnects.

Afterward, inspect:

```bash
cd /opt/neko-cloud
sudo docker compose logs --since=30m neko caddy
sudo neko-cloud-desktop journalctl -u 'kasmvncserver@:1.service' --since=-30min --no-pager
sudo journalctl -u kasmvnc-socat.service --since=-30min --no-pager
sudo journalctl -p warning --since=-30min --no-pager
```

No crash loop, authentication error, certificate error, out-of-memory event, or disk
error should remain unexplained.

#### Reference: 18. How to use it from any laptop

The laptop is only a viewer. You are accessing **your cloud machine** from the other
person's laptop; they do not need an account and you do not give them the credentials.

```text
borrowed laptop's Chrome Guest window
        -> HTTPS hostname
        -> Caddy on your VM
        -> either Neko Firefox OR your Ubuntu GNOME session
```

Use this flow:

1. Prefer a trusted laptop and a current Chromium browser. In Chrome choose profile
   menu -> **Guest** (best), or use Chrome Incognito/Edge InPrivate. Firefox has KasmVNC
   feature limitations; direct Safari use can fail because Basic Authentication is not
   reliably propagated to WebSockets.
2. For the isolated remote browser, open `https://<NEKO_FQDN>`. The outer browser may
   be Chrome while the visible remote browser is Firefox; they are separate programs.
   Enter a display name such as `viewer` and the appropriate Neko password.
3. For the persistent Ubuntu GNOME desktop, open `https://<DESKTOP_FQDN>` and authenticate as
   configured `KASM_WEB_USER` with the separate Kasm password.
4. Use the Kasm side toolbar -> Displays to resize, or its fullscreen control. Browser
   zoom is not the same as changing the virtual display resolution.
5. When done, sign out of sensitive websites inside the remote browser/desktop, clear
   the local clipboard, close **all** Guest/Private windows, and verify the Guest profile
   disappeared. Do not select â€œsave passwordâ€ or download VM files to the borrowed PC.

The reproducible GNOME session runs as the separate locked, non-sudo Linux account
`desktop`; it is not the SSH administrator's desktop session. Its home directory is
separate, so files under the administrator's home do not automatically appear there.
GUI authorization prompts and `sudo` are intentionally unavailable. Perform package,
service, and system changes over SSH as `ADMIN_USER`, using `sudo` there.

Closing the local viewer does not stop the VM. Neko continues according to its session
state; the GNOME desktop and open apps persist. To intentionally end GNOME applications,
log out inside Ubuntu or stop the Kasm user service over SSH.

No VPN client, RDP client, browser extension, phone authenticator, or passkey is required
by this deployment. That convenience does not make an unknown laptop trustworthy.
Guest/Private mode reduces ordinary history, cookies, and saved-password residue but
cannot defeat malware, browser policy, keyloggers, screenshots, network monitoring,
or a malicious owner. Avoid banking, primary email, cloud consoles, password managers,
and private SSH keys on a machine you do not control.

â€œAny laptopâ€ means no special client installation; it does not mean every guest network
will pass the media traffic. The Kasm desktop uses HTTPS/WebSocket on TCP 443, but this
direct ICE-lite Neko design needs UDP 59000 or its TCP 59000 fallback. A corporate,
hotel, or school network that permits only 80/443 can load Neko's page yet still block
its media session. Supporting 443-only networks requires a separately designed and
tested TURN/TLS or compatible relay path; it is not part of this deployment.

To change Neko's remote resolution persistently, edit `NEKO_DESKTOP_SCREEN` in
`/opt/neko-cloud/compose.yaml`, for example `1366x768@24`, then recreate only Neko:

```bash
cd /opt/neko-cloud
sudo docker compose up -d --no-deps --force-recreate neko
sudo docker compose logs --tail=100 neko
```

For GNOME, prefer KasmVNC's Displays control; `allow_resize: true` permits browser-driven
changes. Very high resolution/frame rate increases VM CPU and network use.

#### Reference: 19. Routine operations, updates, backups, and rollback

##### Reference: 19.1 Status, start, stop, and logs

```bash
# Containers
cd /opt/neko-cloud
sudo docker compose ps
sudo docker compose logs --tail=200 neko caddy
sudo docker stats --no-stream

# Host desktop and bridge
sudo neko-cloud-desktop systemctl status 'kasmvncserver@:1.service' --no-pager
sudo systemctl status kasmvnc-socat.service gnome-remote-desktop.service --no-pager
sudo neko-cloud-desktop journalctl -u 'kasmvncserver@:1.service' -n 200 --no-pager
sudo journalctl -u kasmvnc-socat.service -n 200 --no-pager

# Capacity and listeners
free -h
df -hT
sudo du -xhd1 /var/lib/docker /home /opt 2>/dev/null
sudo ss -lntup
```

Start/restart:

```bash
sudo neko-cloud-desktop systemctl restart 'kasmvncserver@:1.service'
sudo systemctl restart kasmvnc-socat.service
cd /opt/neko-cloud && sudo docker compose up -d
```

Graceful maintenance stop:

```bash
cd /opt/neko-cloud && sudo docker compose stop
sudo neko-cloud-desktop systemctl stop 'kasmvncserver@:1.service'
sudo systemctl stop kasmvnc-socat.service
```

Resume in the reverse order. Do not disable the linger setting unless the desktop should
no longer start at boot. Caddy renews certificates automatically while ports 80/443,
DNS, its `/data` volume, and outbound ACME access remain healthy.

##### Reference: 19.2 Safe updates and rollback

Before a meaningful update, create a provider boot-volume snapshot/backup and copy the
current configuration plus image digests. Treat the following as one controlled Ubuntu
OS maintenance window; APT can update the kernel, GNOME, Docker dependencies, and other
packages together, so inspect the candidate list before approving it:

```bash
sudo apt-get update
apt list --upgradable
sudo apt-get upgrade -y
test -e /var/run/reboot-required && cat /var/run/reboot-required || true
```

Reboot if required, then repeat section 17. For containers, never use an unattended
floating-tag updater. Pull the desired tag, inspect its release notes and digest, replace
one pinned digest in `compose.yaml`, recreate that service, test, and retain the old
digest for rollback:

```bash
cd /opt/neko-cloud
sudo cp -a compose.yaml "compose.yaml.before-$(date -u +%Y%m%dT%H%M%SZ)"
sudo docker pull '<IMAGE:TESTED_TAG>'
sudo docker image inspect '<IMAGE:TESTED_TAG>' --format '{{index .RepoDigests 0}}'
# Put that exact digest in compose.yaml, then:
sudo docker compose config --quiet
sudo docker compose up -d --no-deps --force-recreate '<SERVICE>'
sudo docker compose logs --tail=200 '<SERVICE>'
```

If acceptance fails, restore the saved Compose file and run the same `up` command. For
KasmVNC/GNOME or kernel upgrades, a provider snapshot is the reliable rollback; APT does
not provide a universal atomic rollback.

##### Reference: 19.3 Rotate credentials and SSH keys

Rotate any credential disclosed in chat, screenshots, terminal recording, or an
unencrypted backup. For Neko, edit the mode-0600 `.env` with `sudoedit`, verify mode,
and recreate Neko:

```bash
sudoedit /opt/neko-cloud/.env
sudo chmod 0600 /opt/neko-cloud/.env
cd /opt/neko-cloud
sudo docker compose up -d --no-deps --force-recreate neko
```

For KasmVNC, run as the desktop user:

```bash
. /etc/neko-cloud.env
sudo -iu "$DESKTOP_USER"
. /etc/neko-cloud.env
export XDG_RUNTIME_DIR=/run/user/$(id -u)
export DBUS_SESSION_BUS_ADDRESS=unix:path=$XDG_RUNTIME_DIR/bus
vncpasswd -u "$KASM_WEB_USER" -w
chmod 0600 "$HOME/.kasmpasswd"
systemctl --user restart 'kasmvncserver@:1.service'
exit
```

For SSH, append the new public key to `~/.ssh/authorized_keys`, fix mode 0600, prove the
new key in a second session, then remove only the old public-key line. Never delete the
only working key before testing replacement access.

##### Reference: 19.4 Backups and restore order

Use two independent layers:

1. A provider boot-volume snapshot/backup after gracefully stopping the applications.
   This captures packages, GNOME, Docker volumes, and system services. Snapshots cost
   storage and can share the same account failure domain, so they are not the only copy.
2. An encrypted file-level backup stored off the VM and periodically restore-tested.

Create a non-secret configuration archive:

```bash
set -euo pipefail
. /etc/neko-cloud.env
[ "$(id -un)" = "$ADMIN_USER" ] || {
  echo "Run this as the recorded SSH administrator: $ADMIN_USER" >&2
  exit 1
}
STAMP=$(date -u +%Y%m%dT%H%M%SZ)
ADMIN_GROUP=$(id -gn "$ADMIN_USER")
DESKTOP_REL=${DESKTOP_HOME#/}
CONFIG_ARCHIVE="/opt/backups/neko-cloud-config-$STAMP.tgz"
sudo install -d -m 0700 /opt/backups
sudo tar --xattrs --acls -C / -czf "$CONFIG_ARCHIVE" \
  opt/neko-cloud/compose.yaml opt/neko-cloud/Caddyfile opt/neko-cloud/.env.example \
  opt/neko-cloud-manifest/base.txt \
  opt/neko-cloud-manifest/docker-server.json \
  opt/neko-cloud-manifest/docker-compose.txt \
  opt/neko-cloud-manifest/docker-apt.txt \
  opt/neko-cloud-manifest/neko-image.txt \
  opt/neko-cloud-manifest/caddy-image.txt \
  etc/neko-cloud.env etc/docker/daemon.json \
  etc/ssh/sshd_config.d/00-neko-cloud-hardening.conf \
  etc/systemd/system/kasmvnc-socat.service \
  usr/local/sbin/neko-cloud-desktop \
  "$DESKTOP_REL/.config/systemd/user/kasmvncserver@.service" \
  "$DESKTOP_REL/.vnc/kasmvnc.yaml" "$DESKTOP_REL/.vnc/xstartup"
sudo chmod 0600 "$CONFIG_ARCHIVE"
sudo tar -tzf "$CONFIG_ARCHIVE"
sudo sha256sum "$CONFIG_ARCHIVE"
sudo install -m 0600 -o "$ADMIN_USER" -g "$ADMIN_GROUP" \
  "$CONFIG_ARCHIVE" "$ADMIN_HOME/neko-cloud-config-$STAMP.tgz"
```

That archive intentionally excludes `.env`, `.kasmpasswd`, private SSH keys, and normal
home data. Separately export the provider's instance/NIC/IP/firewall/route descriptions
or infrastructure-as-code state, and encrypt that inventory because it contains account
and resource identifiers even though it has no login secret. For a separate encrypted
credential archive, install GnuPG and use a unique
backup passphrase stored outside the VM:

```bash
set -euo pipefail
sudo apt-get install -y gnupg
. /etc/neko-cloud.env
[ "$(id -un)" = "$ADMIN_USER" ] || {
  echo "Run this as the recorded SSH administrator: $ADMIN_USER" >&2
  exit 1
}
STAMP=$(date -u +%Y%m%dT%H%M%SZ)
ADMIN_GROUP=$(id -gn "$ADMIN_USER")
DESKTOP_PASS_MEMBER="${DESKTOP_HOME#/}/.kasmpasswd"
SECRET_TMPDIR=$(mktemp -d /dev/shm/neko-cloud-secrets.XXXXXX)
SECRET_TAR="$SECRET_TMPDIR/secrets.tar"
trap 'rm -rf -- "$SECRET_TMPDIR"' EXIT HUP INT TERM
# Do not pre-create SECRET_TAR as the unprivileged user: protected_regular can prevent
# root from reopening that file. Root creates a new tar inside this private tmpfs dir.
sudo tar -C / -cf "$SECRET_TAR" opt/neko-cloud/.env "$DESKTOP_PASS_MEMBER"
sudo chown "$ADMIN_USER:$ADMIN_GROUP" "$SECRET_TAR"
chmod 0600 "$SECRET_TAR"

EXPECTED=$(printf '%s\n' opt/neko-cloud/.env "$DESKTOP_PASS_MEMBER" | LC_ALL=C sort)
ACTUAL=$(tar -tf "$SECRET_TAR" | LC_ALL=C sort)
[ "$ACTUAL" = "$EXPECTED" ] || {
  echo 'Secret archive contains an unexpected path; refusing to encrypt it.' >&2
  printf 'Expected:\n%s\nActual:\n%s\n' "$EXPECTED" "$ACTUAL" >&2
  exit 1
}
tar -tvf "$SECRET_TAR" | awk '
  substr($1,1,1) != "-" { bad=1 }
  END { exit bad }
'

SECRET_ARCHIVE="$ADMIN_HOME/neko-cloud-secrets-$STAMP.tar.gpg"
gpg --symmetric --cipher-algo AES256 --output "$SECRET_ARCHIVE" "$SECRET_TAR"
chmod 0600 "$SECRET_ARCHIVE"
sha256sum "$SECRET_ARCHIVE"
rm -rf -- "$SECRET_TMPDIR"
trap - EXIT HUP INT TERM
```

The unencrypted secret tar exists only inside a mode-0700 `/dev/shm` directory, has mode
0600 after creation, and is removed explicitly after success or by the trap on failure;
encryption must finish successfully before the result is accepted. Both
transferable archives are now in `$ADMIN_HOME`. Copy them over authenticated SSH to
encrypted storage, verify their remote checksums, then remove the home-directory copies.
Keep or expire the root-owned `/opt/backups` copy according to retention policy. Back up
user documents separately;
browser caches and the stateless Neko profile generally should not be archived.

The file-level archives deliberately exclude Caddy's named `/data` and `/config` Docker
volumes. An exact copy of those volumes comes only from the provider boot-volume
snapshot in this procedure. Without that snapshot, Caddy recreates its configuration
volume and obtains new certificates after DNS, ports 80/443, outbound ACME access, and
rate limits are healthy.

Restore in this order:

1. Provision the stable IP, DNS, five cloud firewall rules, and Ubuntu of the same CPU
   architecture.
2. Install Docker, GNOME, and the exact verified KasmVNC package.
3. Recreate the dedicated `desktop` account and target `/etc/neko-cloud.env` using section
   12. Do not overwrite that target inventory with the old cloud's IP/user values.
4. Verify the recorded SHA-256 values, stop the new services, and stage the non-secret
   archive as the SSH administrator rather than extracting it directly over `/`:

```bash
set -euo pipefail
. /etc/neko-cloud.env
CONFIG_ARCHIVE='<PATH_TO_VERIFIED_CONFIG_ARCHIVE.tgz>'
CONFIG_STAGE=$(mktemp -d)
trap 'rm -rf -- "$CONFIG_STAGE"' EXIT HUP INT TERM
DESKTOP_REL=${DESKTOP_HOME#/}
EXPECTED_CONFIG=$(printf '%s\n' \
  opt/neko-cloud/compose.yaml \
  opt/neko-cloud/Caddyfile \
  opt/neko-cloud/.env.example \
  opt/neko-cloud-manifest/base.txt \
  opt/neko-cloud-manifest/docker-server.json \
  opt/neko-cloud-manifest/docker-compose.txt \
  opt/neko-cloud-manifest/docker-apt.txt \
  opt/neko-cloud-manifest/neko-image.txt \
  opt/neko-cloud-manifest/caddy-image.txt \
  etc/neko-cloud.env \
  etc/docker/daemon.json \
  etc/ssh/sshd_config.d/00-neko-cloud-hardening.conf \
  etc/systemd/system/kasmvnc-socat.service \
  usr/local/sbin/neko-cloud-desktop \
  "$DESKTOP_REL/.config/systemd/user/kasmvncserver@.service" \
  "$DESKTOP_REL/.vnc/kasmvnc.yaml" \
  "$DESKTOP_REL/.vnc/xstartup" | LC_ALL=C sort)
ACTUAL_CONFIG=$(tar -tzf "$CONFIG_ARCHIVE" | LC_ALL=C sort)
[ "$ACTUAL_CONFIG" = "$EXPECTED_CONFIG" ] || {
  echo 'STOP: config archive has missing, duplicate, or unexpected members.' >&2
  printf 'Expected:\n%s\nActual:\n%s\n' "$EXPECTED_CONFIG" "$ACTUAL_CONFIG" >&2
  exit 1
}
tar -tvzf "$CONFIG_ARCHIVE" | awk '
  substr($1,1,1) != "-" { bad=1 }
  END { exit bad }
' || {
  echo 'STOP: config archive contains a non-regular member.' >&2; exit 1;
}
tar --no-same-owner --no-same-permissions -xzf "$CONFIG_ARCHIVE" -C "$CONFIG_STAGE"

# Inspect/edit the staged Compose and Caddy files for this target IP and hostnames first.
sudo install -m 0644 "$CONFIG_STAGE/opt/neko-cloud/compose.yaml" /opt/neko-cloud/compose.yaml
sudo install -m 0644 "$CONFIG_STAGE/opt/neko-cloud/Caddyfile" /opt/neko-cloud/Caddyfile
sudo install -m 0644 "$CONFIG_STAGE/opt/neko-cloud/.env.example" /opt/neko-cloud/.env.example
if sudo test -e /etc/docker/daemon.json; then
  if ! sudo cmp -s "$CONFIG_STAGE/etc/docker/daemon.json" /etc/docker/daemon.json; then
    echo 'STOP: staged and target Docker settings differ. Compare privately and merge;' >&2
    echo 'do not print possible private registry/mirror values into a recorded terminal.' >&2
    exit 1
  fi
else
  sudo install -m 0644 "$CONFIG_STAGE/etc/docker/daemon.json" \
    /etc/docker/daemon.json
fi
sudo dockerd --validate --config-file=/etc/docker/daemon.json
sudo install -m 0644 \
  "$CONFIG_STAGE/etc/ssh/sshd_config.d/00-neko-cloud-hardening.conf" \
  /etc/ssh/sshd_config.d/00-neko-cloud-hardening.conf
sudo install -m 0644 \
  "$CONFIG_STAGE/etc/systemd/system/kasmvnc-socat.service" \
  /etc/systemd/system/kasmvnc-socat.service
sudo install -m 0755 "$CONFIG_STAGE/usr/local/sbin/neko-cloud-desktop" \
  /usr/local/sbin/neko-cloud-desktop

sudo install -d -m 0700 -o "$DESKTOP_USER" -g "$DESKTOP_USER" \
  "$DESKTOP_HOME/.vnc" "$DESKTOP_HOME/.config/systemd/user"
sudo install -m 0600 -o "$DESKTOP_USER" -g "$DESKTOP_USER" \
  "$CONFIG_STAGE/$DESKTOP_REL/.vnc/kasmvnc.yaml" "$DESKTOP_HOME/.vnc/kasmvnc.yaml"
sudo install -m 0755 -o "$DESKTOP_USER" -g "$DESKTOP_USER" \
  "$CONFIG_STAGE/$DESKTOP_REL/.vnc/xstartup" "$DESKTOP_HOME/.vnc/xstartup"
sudo install -m 0644 -o "$DESKTOP_USER" -g "$DESKTOP_USER" \
  "$CONFIG_STAGE/$DESKTOP_REL/.config/systemd/user/kasmvncserver@.service" \
  "$DESKTOP_HOME/.config/systemd/user/kasmvncserver@.service"
rm -rf -- "$CONFIG_STAGE"
trap - EXIT HUP INT TERM
```

5. Decrypt secrets to protected memory-backed staging, require successful GPG
   authentication, verify exactly two regular-file members, and install them with target
   ownership. Never pipe unverified GPG output directly into root `tar`:

```bash
set -euo pipefail
umask 077
. /etc/neko-cloud.env
SECRET_ARCHIVE='<PATH_TO_VERIFIED_SECRET_ARCHIVE.tar.gpg>'
RESTORE_ROOT=$(mktemp -d /dev/shm/neko-cloud-restore.XXXXXX)
trap 'rm -rf -- "$RESTORE_ROOT"' EXIT HUP INT TERM
RESTORE_TAR="$RESTORE_ROOT/secrets.tar"
RESTORE_FILES="$RESTORE_ROOT/files"
mkdir -m 0700 "$RESTORE_FILES"

gpg --output "$RESTORE_TAR" --decrypt "$SECRET_ARCHIVE"
DESKTOP_PASS_MEMBER="${DESKTOP_HOME#/}/.kasmpasswd"
EXPECTED=$(printf '%s\n' opt/neko-cloud/.env "$DESKTOP_PASS_MEMBER" | LC_ALL=C sort)
ACTUAL=$(tar -tf "$RESTORE_TAR" | LC_ALL=C sort)
[ "$ACTUAL" = "$EXPECTED" ] || {
  echo 'Unexpected secret-archive member; refusing restore.' >&2
  exit 1
}
tar -tvf "$RESTORE_TAR" | awk '
  substr($1,1,1) != "-" { bad=1 }
  END { exit bad }
'
tar --no-same-owner --no-same-permissions -xf "$RESTORE_TAR" -C "$RESTORE_FILES"
sudo install -m 0600 -o root -g root \
  "$RESTORE_FILES/opt/neko-cloud/.env" /opt/neko-cloud/.env
sudo install -m 0600 -o "$DESKTOP_USER" -g "$DESKTOP_USER" \
  "$RESTORE_FILES/$DESKTOP_PASS_MEMBER" "$DESKTOP_HOME/.kasmpasswd"
rm -rf -- "$RESTORE_ROOT"
trap - EXIT HUP INT TERM
```

6. **Before starting anything**, update DNS, Caddy hostnames, and Neko `nat1to1` if the
   target has a different public IP. Regenerate the local Kasm placeholder certificate
   if section 15 did not create it. Reapply the three `gsettings` idle/lock/suspend values
   from section 15 as `DESKTOP_USER` because the general dconf database is deliberately
   not archived. Validate SSH with `sshd -t`, reload systemd, enable linger, then start
   the GNOME handover/Kasm/socat/Compose dependency chain.
7. Run every section 17 test. Rotate all three restored application passwords (Neko
   administrator, Neko regular user, and the configured Kasm web user) after the
   recovery test, then create a new target-specific backup.

After editing the target-specific DNS/Caddy/Neko values, the concrete step 6 activation
sequence is:

```bash
set -euo pipefail
. /etc/neko-cloud.env
sudo -iu "$DESKTOP_USER" dbus-run-session -- \
  gsettings set org.gnome.desktop.session idle-delay 'uint32 0'
sudo -iu "$DESKTOP_USER" dbus-run-session -- \
  gsettings set org.gnome.desktop.screensaver lock-enabled false
sudo -iu "$DESKTOP_USER" dbus-run-session -- \
  gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'

sudo sshd -t
sudo systemctl reload ssh.service
sudo systemctl daemon-reload
sudo systemctl enable --now gnome-remote-desktop.service
if sudo ss -lntup | grep -qE ':(3389)\b'; then
  echo 'STOP: unexpected RDP listener; do not open or continue.' >&2; exit 1
fi
sudo loginctl enable-linger "$DESKTOP_USER"
DESKTOP_UID=$(id -u "$DESKTOP_USER")
sudo systemctl start "user@$DESKTOP_UID.service"
sudo neko-cloud-desktop systemctl daemon-reload
sudo neko-cloud-desktop systemctl enable --now 'kasmvncserver@:1.service'
sudo systemctl enable --now kasmvnc-socat.service
cd /opt/neko-cloud
sudo docker compose config --quiet
sudo docker compose up -d
```

At least quarterly, restore into an isolated temporary VM and prove the URLs, WebRTC,
GNOME persistence, reboot, and closed-port checks. A backup that has never been restored
is unverified.

##### Reference: 19.5 Monitoring and cost controls

Set provider budget alerts at conservative thresholds, but remember they do not stop
spend. Review monthly compute, disk, snapshot, public IPv4, DNS, and egress line items.
Also monitor:

- provider VM/agent/infrastructure health and CPU/network graphs;
- disk usage and Docker log growth;
- memory/swap/OOM events (`journalctl -k | grep -i oom`);
- failed systemd units and container restart counts;
- public TLS expiry and endpoint availability;
- upstream Neko, KasmVNC, Caddy, Docker, Ubuntu, and kernel security releases;
- OCI A1 idle-reclamation metrics if using Always Free.

An external HTTPS uptime check should authenticate nowhere and should avoid aggressive
polling. A TCP/UDP media health check cannot replace a periodic real browser session.

#### Reference: 20. Safe cleanup

Cleanup begins with evidence, not a purge:

```bash
df -hT
sudo du -xhd1 /var /opt /home 2>/dev/null | sort -h
sudo docker system df -v
sudo apt-get autoremove --dry-run
sudo journalctl --disk-usage
systemctl --failed --no-pager
systemctl list-unit-files --state=enabled
snap list 2>/dev/null || true
```

Low-risk periodic cleanup:

```bash
sudo apt-get clean
sudo journalctl --vacuum-time=14d --vacuum-size=200M
sudo apt-get autoremove --dry-run
```

Only run the real `apt-get autoremove` after reviewing every proposed package and taking
a snapshot. Never run `docker system prune -a --volumes`: it can remove rollback images,
build cache, unused networks, and persistent Caddy data/certificates. `docker image
prune` should also wait until the old pinned digest is no longer needed for rollback.

On this headless workload, the following services may be disabled **only if their named
features are not used**:

```bash
sudo systemctl disable --now \
  cups.service cups.socket cups.path cups-browsed.service \
  avahi-daemon.service avahi-daemon.socket \
  ModemManager.service rpcbind.service rpcbind.socket bluetooth.service \
  sssd.service openvpn.service 2>/dev/null || true
```

Those correspond to printing, local mDNS discovery, modem hardware, RPC/NFS discovery,
Bluetooth hardware, SSSD domain login, and the legacy generic OpenVPN unit. Do not
disable a unit if the VM actually uses that feature or a provider depends on it.

Preserve the components required by the selected mode:

- Docker/containerd and Caddy/Neko for Neko-containing modes.
- `kasmvnc-socat`, the KasmVNC user service, user lingering, and the tested
  `gnome-remote-desktop.service` handover unit only for desktop-containing modes,
  while keeping RDP off.
- `netfilter-persistent` and OCI `InstanceServices`/link-local/iSCSI rules on OCI.
- The active network renderer, cloud-init/provider guest agent, SSH, time sync, and DNS.
- Caddy's `/data` and `/config` volumes.
- Provider snapshots and `/opt/backups` until a verified independent replacement exists.

Do not retain XFCE, XRDP, GNOME, or KasmVNC on a deliberate Neko-only host as an
unrequested fallback. Inventory packages and services first, then remove only components
that are both unused and outside the selected mode.

Do not disable `systemd-networkd-wait-online.service` merely because a different host
once had a slow boot. Change networking units only after proving which renderer owns the
NIC, observing the actual delay, and arranging serial-console recovery.

##### Reference: 20.1 Retiring the cloud deployment without orphaned charges

Stopping or terminating only the VM is not a complete teardown. Reserved/static public
IPv4 addresses, boot/data disks, snapshots, backup vaults, DNS zones/records, and some
network services can survive and remain billable or consume free-tier quota. Retirement
is intentionally a reviewed checklist rather than a one-line recursive delete:

1. Freeze changes; create, checksum, copy off-cloud, and test-restore the final encrypted
   backups. Export the provider inventory, tags, effective firewall rules, DNS, and
   current billing view.
2. Enumerate exact dependencies and mark which are shared. Do not delete a shared VPC,
   VNet, VCN, subnet, DNS zone, key, bucket, or backup repository.
3. Revoke application credentials and SSH keys, lower DNS TTL in advance, remove/change
   public records, and wait for the old TTL before releasing the address.
4. Delete the intended VM only after checking its boot/data-disk delete-on-termination
   settings. Separately decide retention for every disk, image, snapshot, and backup.
5. Release the now-unneeded stable address and delete only unshared firewall/security
   groups, NIC/VNICs, routes, gateways, subnets, and networks after confirming no
   dependents remain.
6. In OCI, check reserved public IPs, boot/block volumes and backups, VCN/NSG/security
   lists, DNS, and object storage. In AWS, check EIPs, EBS volumes/snapshots/AMIs,
   security groups/VPC objects, Route 53, and S3. In GCP, check static external IPs,
   persistent disks/snapshots/images, firewall/VPC objects, Cloud DNS, and Cloud Storage.
   In Azure, enumerate the resource group before deleting any member: public IP, NIC,
   managed disks/snapshots, NSG/VNet, DNS, and Recovery Services/Backup vault contents.
7. Recheck provider billing/cost analysis and resource search after eventual consistency
   settles and again on the next billing day. Keep the final inventory and deletion
   evidence, but never keep plaintext credentials.

#### Reference: 21. Troubleshooting

##### Reference: Cloud console says â€œunresponsiveâ€ or SSH fails

1. Check provider lifecycle state and infrastructure/instance health metrics; â€œRunningâ€
   does not prove the guest OS is healthy.
2. Check whether HTTPS still works. If only SSH fails, the administrator public IP may
   have changed or the `/32` rule may be wrong.
3. Re-audit public IP, VNIC/NIC, subnet, internet route, cloud firewall union/effective
   rules, and host firewall. On OCI, rerun section 7.4 without changing anything.
4. Use the provider serial console/boot diagnostics. Check `df -h`, `free -h`,
   `journalctl -b -p warning`, `journalctl -k`, failed units, and provider guest agent.
5. If CPU is saturated, wait for package/video work to finish or stop Neko from console;
   if disk is full, remove only identified cache/log files. Repeated forced reboots can
   worsen filesystem/package state.
6. If the OS cannot boot, restore the last verified snapshot or attach the boot volume
   to a rescue VM. Preserve the original volume until data recovery is complete.

##### Reference: HTTP works but HTTPS/certificate issuance fails

```bash
getent ahostsv4 '<NEKO_FQDN>'
getent ahostsv4 '<DESKTOP_FQDN>'
timedatectl
cd /opt/neko-cloud
sudo docker compose ps
sudo docker compose logs --tail=300 caddy
sudo ss -lntp | grep -E ':(80|443)\b'
```

The names must resolve to the current static IP; TCP 80/443 must be allowed at cloud and
host firewalls; no Apache/Nginx process may already own them; time must be correct; Caddy
must retain a writable `/data` volume. Remove an incorrect `AAAA` record. Do not request
repeated certificates while debugging DNS because ACME rate limits apply. Raw-IP HTTPS
is not the configured site.

##### Reference: Neko shows `token not found` or a cookie-auth warning

For the pinned 3.1.4 client, confirm the working compatibility settings are present:

```bash
cd /opt/neko-cloud
NEKO_CONTAINER=$(sudo docker compose ps -q neko)
sudo docker inspect "$NEKO_CONTAINER" \
  --format '{{range .Config.Env}}{{println .}}{{end}}' \
  | grep -E '^(NEKO_LEGACY|NEKO_SESSION_COOKIE_ENABLED|NEKO_SERVER_PROXY)='
sudo docker compose logs --tail=300 neko caddy
```

Expected: legacy `true`, session-cookie auth `false`, proxy `true`. The filter prints only
those three names; do not replace it with plain `docker compose config`, which resolves
and prints the passwords. After correcting configuration, recreate Neko, clear site data
for the Neko hostname or open a fresh Guest window, and log in again. When upgrading
Neko, follow that release's authentication migration rather than carrying compatibility
flags forever.

##### Reference: Neko is visible but clicks/typing do not work

- A regular user can normally obtain control when the room controls are unlocked. Inspect
  Neko's mouse/control/lock state and claim control there.
- Use the administrator login only to unlock or override room policy; do not use it for
  ordinary browsing merely because input is not working.
- Close stale duplicate tabs and log in again if the token changed.
- Check that another administrator has not locked control.
- Inspect Neko logs; verify the browser window itself is not displaying an extension or
  modal overlay that intercepts input.

This is distinct from KasmVNC permissions. For Kasm, ensure the user has write permission:

```bash
. /etc/neko-cloud.env
sudo -iu "$DESKTOP_USER"
. /etc/neko-cloud.env
vncpasswd -u "$KASM_WEB_USER" -w -n
exit
sudo neko-cloud-desktop systemctl restart 'kasmvncserver@:1.service'
```

##### Reference: Neko page loads but video/audio/WebRTC does not

```bash
grep -E 'NAT1TO1|UDPMUX|TCPMUX' /opt/neko-cloud/compose.yaml
curl -4 https://ifconfig.me
sudo ss -lntup | grep 59000
cd /opt/neko-cloud && sudo docker compose logs --tail=300 neko
```

`nat1to1` must equal the current external IPv4. Both TCP and UDP 59000 must be published,
allowed by the provider, and allowed by any host/edge firewall. Test another network;
corporate Wi-Fi may block UDP, in which case TCP fallback should work. Reduce resolution
and frame rate if the session connects but encoding stalls.

##### Reference: Desktop returns 401, 502, or a blank page

- `401 Unauthorized` is normal before Kasm Basic authentication. Use the Kasm username
  and Kasm password, not the Neko password or Linux SSH key.
- `502 Bad Gateway` means Caddy cannot traverse the socket bridge to TCP 8444.

```bash
sudo neko-cloud-desktop systemctl status 'kasmvncserver@:1.service' --no-pager
sudo systemctl status kasmvnc-socat.service --no-pager
sudo ss -lntup | grep 8444
sudo stat /run/kasmvnc/kasm.sock
sudo curl --unix-socket /run/kasmvnc/kasm.sock -I http://localhost/
cd /opt/neko-cloud && sudo docker compose logs --tail=300 caddy
```

Restart in dependency order: Kasm user service, socat system service, then Caddy. If the
socket was created after the container, force-recreate Caddy. Never solve a 502 by
publishing 8444 publicly.

##### Reference: The desktop looks like XFCE instead of normal Ubuntu

```bash
. /etc/neko-cloud.env
sudo grep -E 'gnome-session|GNOME_SHELL_SESSION_MODE|DESKTOP_SESSION' \
  "$DESKTOP_HOME/.vnc/xstartup"
pgrep -af -u "$DESKTOP_USER" 'xfce|Xkasmvnc|gnome-shell|gnome-session'
sudo neko-cloud-desktop systemctl restart 'kasmvncserver@:1.service'
```

The xstartup file must end with `gnome-session --session=ubuntu`, and the user unit must
carry the Ubuntu/GNOME environment. Do not start both XFCE and GNOME on display `:1`.

##### Reference: GNOME starts and immediately exits

```bash
. /etc/neko-cloud.env
DESKTOP_UID=$(id -u "$DESKTOP_USER")
loginctl show-user "$DESKTOP_USER" -p Linger -p State -p RuntimePath
sudo test -S "/run/user/$DESKTOP_UID/bus" && echo 'user bus exists'
sudo neko-cloud-desktop systemctl status 'kasmvncserver@:1.service' --no-pager
sudo systemctl status gnome-remote-desktop.service --no-pager
sudo neko-cloud-desktop journalctl -b -u 'kasmvncserver@:1.service' --no-pager
sudo find "$DESKTOP_HOME/.vnc" -maxdepth 1 -name '*.log' \
  -exec tail -n 200 {} +
```

Enable linger, confirm `%t` resolved to the user's runtime directory, preserve the tested
GNOME handover service, and confirm GDM is masked. A hardcoded UID from another VM causes
the D-Bus check to fail.

##### Reference: Screen is too large, small, slow, or cropped

Use Kasm's Displays control for GNOME and fullscreen for the viewer. `Ctrl+0` resets the
outer browser's zoom; it does not change the VM resolution. For Neko edit
`NEKO_DESKTOP_SCREEN` and recreate only Neko. Reduce from 1080p/60 to 1280x720@24 or
1024x576@20 when CPU or bandwidth is limited. Confirm local browser zoom is 100% before
diagnosing remote scaling.

##### Reference: High CPU/RAM, disk full, or crash loops

```bash
top -o %CPU
sudo docker stats --no-stream
ps -eo pid,user,%cpu,%mem,rss,cmd --sort=-%cpu | head -30
df -hT
sudo docker system df -v
sudo journalctl -k | grep -iE 'oom|out of memory|killed process'
systemctl show -p NRestarts kasmvnc-socat.service
```

Close video-heavy tabs, lower resolution/frame rate, stop whichever remote UI is not in
use, or resize the VM. Clear identified caches/logs only; do not prune volumes. A 2 vCPU
VM can saturate during real-time encoding even with free RAM.

##### Reference: Wrong architecture or package

`exec format error` means an AMD64 artifact was used on ARM64 or vice versa. Compare
`dpkg --print-architecture`, `docker image inspect '<IMAGE_REFERENCE>' --format
'{{.Architecture}}'`, the cloud shape, and the Kasm package suffix. Google Chrome Neko
is not native on ARM64; use Firefox/Chromium or move to an AMD64 VM.

##### Reference: An internal port is public

```bash
sudo docker ps --format 'table {{.Names}}\t{{.Ports}}'
sudo ss -lntup
cd /opt/neko-cloud
NEKO_CONTAINER=$(sudo docker compose ps -q neko)
sudo docker inspect "$NEKO_CONTAINER" \
  --format '{{json .HostConfig.PortBindings}}' | jq .
```

Remove accidental Compose `ports` for 5901/8080/8444/3389, recreate the affected service,
remove cloud firewall/security-list rules, then retest externally. On Docker hosts, do
not rely on UFW alone. On OCI, preserve Oracle firewall rules and do not enable UFW.

#### Reference: 22. Deployment inventory template

Record each real installation in a private operations system. Do not turn the public
tutorial into an asset inventory. Public IP addresses are not credentials, but publishing
them together with usernames, versions, topology, and maintenance details gives an
attacker unnecessary targeting information.

##### Reference: Deployment identity

| Item | Value to record privately |
|---|---|
| Deployment mode | `neko-only`, `desktop-only`, or `combined` |
| Provider and region | `<PROVIDER>` / `<REGION>` |
| Instance/resource ID | `<PRIVATE_INVENTORY_VALUE>` |
| Ubuntu release and architecture | `24.04` / `amd64` or `arm64` |
| Shape | `<VCPU>` vCPU, `<RAM_GIB>` GiB RAM |
| Reserved public IP status | reserved/static or ephemeral/dynamic |
| Neko URL | `https://<NEKO_FQDN>` or not deployed |
| Desktop URL | `https://<DESKTOP_FQDN>` or not deployed |
| SSH administrator | `<ADMIN_USER>`; never the browser-desktop user |
| Desktop OS account | locked, non-sudo account such as `desktop` |
| Kasm web user | `<KASM_WEB_USER>`; password stored separately |
| Backup location and date | off-VM location plus last restore-test date |

Never record a private SSH key, password, recovery code, cloud API credential, rendered
secret environment file, or decrypted backup passphrase in this table.

##### Reference: Version and immutable-image record

Capture the values from the deployed system rather than copying an example:

```bash
cat /etc/os-release
dpkg --print-architecture
docker version
docker compose version
docker image inspect ghcr.io/m1k1o/neko/firefox:3.1.4 \
  --format '{{json .RepoDigests}}'
docker image inspect caddy:2.11.4-alpine --format '{{json .RepoDigests}}'
dpkg-query -W kasmvncserver ubuntu-desktop-minimal gnome-shell 2>/dev/null
```

Record both the human-friendly release/tag and the architecture-specific image digest.
A digest copied from an ARM64 deployment may not be valid for AMD64. Treat updates as a
reviewed change with a rollback point.

##### Reference: Configuration-path record

A portable combined build normally uses paths similar to these:

```text
/opt/neko-cloud/.env                               secret Neko environment, mode 0600
/opt/neko-cloud/compose.yaml                       Neko/Caddy stack
/opt/neko-cloud/Caddyfile                          selected HTTPS routes
/home/<DESKTOP_USER>/.kasmpasswd                   Kasm credential, mode 0600
/home/<DESKTOP_USER>/.vnc/kasmvnc.yaml             private Kasm configuration
/home/<DESKTOP_USER>/.vnc/xstartup                 GNOME startup
/home/<DESKTOP_USER>/.config/systemd/user/kasmvncserver@.service
/etc/systemd/system/kasmvnc-socat.service
/run/kasmvnc/kasm.sock                             transient private bridge socket
/opt/backups/                                      local staging only; keep an off-VM copy
```

Derive the desktop home and runtime directory dynamically. Do not copy a literal
`/home/ubuntu` or `/run/user/1001` from another machine.

##### Reference: Acceptance record

For the selected mode, record the date and result of all applicable checks:

- Local service/container status and absence of failed systemd units.
- HTTPS response and certificate hostname verification for every configured FQDN.
- Keyboard/mouse control, clipboard policy, reconnect, resize, audio where applicable,
  and desktop persistence.
- Neko TCP and UDP media paths from a second network.
- Expected public ports open and internal ports 5901, 8080, 8444, and 3389 closed.
- Reboot recovery.
- Backup decryption and restore rehearsal on a disposable host.
- Billing alert, static-address status, and deletion protection/backup policy.

The repository's deployment templates intentionally contain only documentation-range IPs
and placeholders. Keep rendered host inventories outside Git or in an access-controlled,
encrypted system.

#### Reference: 23. Authoritative references

All provider limits, prices, regions, and software versions can change. The statements
above were checked through 2026-07-20 against these primary sources.

##### Reference: OCI

- [Always Free resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)
- [Compute shapes](https://docs.oracle.com/en-us/iaas/Content/Compute/References/computeshapes.htm)
- [Canonical Ubuntu images on OCI](https://ubuntu.com/docs/oracle/oracle-how-to/find-ubuntu-images/)
- [Public IP addresses](https://docs.oracle.com/en-us/iaas/Content/Network/Tasks/managingpublicIPs.htm)
- [Internet gateways](https://docs.oracle.com/en-us/iaas/Content/Network/Tasks/managingIGs.htm)
- [Route tables](https://docs.oracle.com/en-us/iaas/Content/Network/Tasks/managingroutetables.htm)
- [Security rules and NSG/security-list behavior](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/securityrules.htm)
- [OCI Ubuntu/UFW known issue](https://docs.oracle.com/en-us/iaas/Content/Compute/known-issues.htm)
- [Cost Analysis](https://docs.oracle.com/en-us/iaas/Content/Billing/Concepts/costanalysisoverview.htm)

##### Reference: AWS

- [T4g instances](https://aws.amazon.com/ec2/instance-types/t4/)
- [Unlimited mode and surplus CPU credits](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-performance-instances-unlimited-mode.html)
- [EC2 free-tier usage](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-free-tier-usage.html)
- [AWS free terms](https://aws.amazon.com/free/terms/)
- [Canonical Ubuntu AMI discovery](https://documentation.ubuntu.com/aws/aws-how-to/instances/launch-ubuntu-desktop/)
- [AMI architecture](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html)
- [Security-group rules](https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html)
- [Elastic IP addresses](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)
- [VPC/public IPv4 pricing](https://aws.amazon.com/vpc/pricing/)

##### Reference: GCP

- [Google Cloud Free Tier](https://cloud.google.com/free/docs/free-cloud-features)
- [General-purpose and ARM machine families](https://cloud.google.com/compute/docs/general-purpose-machines)
- [ARM VMs](https://cloud.google.com/compute/docs/instances/arm-on-compute)
- [Regions and zones](https://cloud.google.com/compute/docs/regions-zones)
- [Ubuntu image details](https://cloud.google.com/compute/docs/images/os-details)
- [VPC firewall rules](https://cloud.google.com/firewall/docs/using-firewalls)
- [Reserve a static external IP](https://cloud.google.com/vpc/docs/reserve-static-external-ip-address)
- [IAP TCP forwarding](https://cloud.google.com/iap/docs/using-tcp-forwarding)
- [OS Login roles and setup](https://cloud.google.com/compute/docs/oslogin/set-up-oslogin)
- [VPC pricing](https://cloud.google.com/vpc/pricing)

##### Reference: Azure

- [Bpsv2 ARM sizes](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/general-purpose/bpsv2-series)
- [Azure free account](https://azure.microsoft.com/en-us/free/)
- [Canonical Ubuntu image discovery](https://documentation.ubuntu.com/azure/azure-how-to/instances/find-ubuntu-images/)
- [Network security groups](https://learn.microsoft.com/en-us/azure/virtual-network/manage-network-security-group)
- [Run Azure CLI in an official Docker container](https://learn.microsoft.com/en-us/cli/azure/run-azure-cli-docker)
- [Public IP addresses](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/public-ip-addresses)
- [Explicit/default outbound access](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/default-outbound-access)

##### Reference: Neko, KasmVNC, Caddy, Docker, and Ubuntu

- [m1k1o/neko upstream repository](https://github.com/m1k1o/neko)
- [Neko introduction](https://neko.m1k1o.net/docs/v3/introduction)
- [Neko quick start and sizing](https://neko.m1k1o.net/docs/v3/quick-start)
- [Neko installation](https://neko.m1k1o.net/docs/v3/installation)
- [Neko configuration](https://neko.m1k1o.net/docs/v3/configuration)
- [Neko reverse proxy](https://neko.m1k1o.net/docs/v3/reverse-proxy-setup)
- [Neko images and architectures](https://neko.m1k1o.net/docs/v3/installation/docker-images)
- [KasmVNC installation/support matrix](https://www.kasmweb.com/kasmvnc/docs/latest/install.html)
- [KasmVNC server configuration](https://www.kasmweb.com/kasmvnc/docs/master/serverside.html)
- [KasmVNC client/browser requirements](https://www.kasmweb.com/kasmvnc/docs/master/clientside.html)
- [KasmVNC `vncpasswd`](https://kasmweb.com/kasmvnc/docs/master/man/vncpasswd.html)
- [KasmVNC 1.4.0 release](https://github.com/kasmtech/KasmVNC/releases/tag/v1.4.0)
- [Caddy automatic HTTPS](https://caddyserver.com/docs/automatic-https)
- [Caddy reverse proxy](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy)
- [Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Docker Compose plugin](https://docs.docker.com/compose/install/linux/)
- [Docker port publishing](https://docs.docker.com/engine/network/port-publishing/)
- [Docker packet filtering/firewalls](https://docs.docker.com/engine/network/packet-filtering-firewalls/)
- [Ubuntu firewall guidance](https://documentation.ubuntu.com/server/how-to/security/firewalls/)
- [sslip.io mapping behavior](https://sslip.io/)
- [Let's Encrypt short-lived and IP certificates](https://letsencrypt.org/2026/01/15/6day-and-ip-general-availability.html)

The KasmVNC `latest`/`master` pages are moving documentation and may change after this
runbook's review date. The package URL, SHA-256 values, and GitHub release are pinned to
1.4.0; use the installed 1.4.0 man pages plus its release notes when a moving page differs,
and re-audit every command before upgrading KasmVNC.

#### Reference: 24. Replication completion checklist

Do not declare a replica complete until every applicable item is checked:

- [ ] Provider account, region, free-offer eligibility, price estimate, budget alerts,
  quotas, and public-IP/egress/snapshot charges were reviewed in the live console.
- [ ] Exact Ubuntu image ID/version, CPU architecture, VM shape, disk type/size, region,
  zone, public-IP resource ID, and creation date are recorded without account secrets.
- [ ] Public IP is static/reserved; both DNS names resolve only to it; time and outbound
  DNS/HTTPS/NTP work.
- [ ] Cloud firewall/effective rules expose only TCP 22 from admin `/32`, TCP 80/443,
  and TCP+UDP 59000; IPv6 is either equally secured or absent.
- [ ] Host firewall follows the provider-specific rule: OCI stock iptables preserved
  with no UFW; generic hosts reviewed for UFW/Docker interaction.
- [ ] SSH is key-only, root/password login is off, a second key session was tested, and
  console/recovery access is known.
- [ ] Docker/Compose are installed from the official repository; versions, daemon log
  limits, and root-equivalent access policy are recorded.
- [ ] Neko user/admin and Kasm write-user passwords are unique, mode-0600 at rest, stored in
  a password manager, absent from Git/docs, and rotated if ever disclosed.
- [ ] Neko `nat1to1` exactly matches the current public IPv4; proxy/auth compatibility,
  TCP+UDP mux, resolution, and stateless profile policy are intentional.
- [ ] Neko and Caddy image digests are recorded/pinned; Caddy `/data` and `/config`
  persist; 8080 is not host-published.
- [ ] Correct checksum-verified KasmVNC package is installed for the VM architecture;
  GNOME/Yaru starts through the Ubuntu X11 session, not XFCE or GDM.
- [ ] The Kasm/GNOME OS account is the locked, non-sudo `desktop` user and belongs to no
  privileged group; the cloud SSH administrator does not own the public desktop session.
- [ ] Kasm user service uses `%t`, linger is enabled, socat socket bridge is mode 0660,
  Caddy reaches it, TCP 8444 is loopback-only, and 3389 remains closed.
- [ ] Required services survive a full reboot with no unexplained failed unit or crash
  loop; every configured swap source is active; CPU, RAM, disk, and Docker logs have
  safe headroom.
- [ ] External TCP/TLS/closed-port tests, real UDP WebRTC media, Neko control/audio,
  Kasm authentication/input/resize, exclusive reconnect, and desktop persistence pass.
- [ ] Provider snapshot and encrypted off-VM file backups exist, have checksums, exclude
  unneeded browser caches, and have passed an isolated restore test.
- [ ] Update/rollback ownership, monthly cost review, certificate/endpoint monitoring,
  security release review, and credential rotation dates are assigned.

This checklist closes the deployment; screenshots or a single successful page load do
not. Preserve the manifest and test evidence with the operational documentation, never
with live credentials.

### Reference: Sanitized OCI ARM64 combined deployment case study

This case study explains what was learned from the reference deployment without
publishing its public/private IPs, hostnames, user identity, cloud OCIDs, SSH key name,
passwords, or unredacted screenshots.

#### Reference: Environment

| Item | Sanitized value |
|---|---|
| Provider | Oracle Cloud Infrastructure, public subnet |
| OS | Canonical Ubuntu Server 24.04 ARM64 |
| Compute | 2 OCPU / 12 GiB RAM |
| Storage | Approximately 200 GB boot allocation |
| Modes | Neko Firefox plus Ubuntu GNOME/KasmVNC on one VM |
| Gateway | Caddy with two HTTPS hostnames |
| Media | Neko TCP+UDP mux on 59000 |

This combined build was verified on 2026-07-19. Azure Neko-only was separately
infrastructure-tested in the next case study; AWS, GCP, and generic VPS paths remain
source-grounded equivalents rather than claims of executed deployments.

#### Reference: What happened

1. OCI routing and security rules were built first. TCP 22/80/443/59000 and UDP 59000
   were allowed; application ports 8080/8444/5901/3389 stayed closed.
2. Neko Firefox was deployed behind Caddy. The pinned 3.1.4 client required the legacy
   compatibility path and bearer-token login rather than cookie-session auth.
3. The page initially appeared unclickable because the viewer did not own Neko control.
   Taking control in the UI fixed input; it was not a browser-engine failure.
4. Screen size was changed through Neko's display selector. Firefox was retained because
   the VM is ARM64 and the official Google Chrome Neko image is AMD64-only.
5. A lightweight XFCE/KasmVNC desktop proved browser delivery but did not resemble a
   normal Ubuntu laptop. The host was migrated to `ubuntu-desktop-minimal`, GNOME Shell,
   Yaru, and Ubuntu Dock while KasmVNC continued to supply a virtual X11 display.
6. GDM was masked to prevent an unused second graphical console. User lingering and the
   user D-Bus/runtime directory kept GNOME persistent across disconnects and reboot.
7. Caddy reached loopback-only KasmVNC through a private Unix-socket/socat bridge.
8. Resource pressure during desktop installation temporarily made the VM appear
   unresponsive. Provider metrics/console, reboot recovery, swap, and measured cleanup
   were added to the runbook rather than recreating the VM.

#### Reference: Corrections carried into the generic project

- The publishable Kasm unit uses `%t`, never a hardcoded `/run/user/<UID>`.
- The browser desktop uses a dedicated locked, non-sudo OS account, not the cloud admin.
- Caddy/Compose use required environment values, not rendered live IPs or hostnames.
- Neko-only, desktop-only, and combined modes now have distinct port and sizing rules.
- OCI Ubuntu preserves Oracle's firewall/iSCSI rules and does not enable UFW.
- Static/reserved address state is a required inventory item before relying on DNS.
- Password-only access is documented as a baseline tradeoff, not a universal best
  practice; stronger proxy/VPN identity is an optional security layer.

#### Reference: Acceptance evidence

The case validated HTTPS/certificate verification, authenticated video, remote mouse and
keyboard, GNOME Activities, screen resizing, session persistence, reboot recovery, Neko
TCP/UDP media, and external closure of internal ports. Screenshots, rendered host
configuration, credentials, and provider identifiers are excluded from this repository;
store any future evidence in access-controlled private storage.

### Reference: Sanitized Azure AMD64 Neko-only deployment case study

This case records the 2026-07-20 Azure deployment without publishing its address,
hostname, subscription/resource names, administrator CIDR, account identity, SSH key,
passwords, device codes, tokens, or screenshots.

#### Reference: Azure case environment

| Item | Sanitized value |
|---|---|
| Provider | Microsoft Azure, public IPv4 VM |
| OS | Canonical Ubuntu Server 24.04 AMD64 |
| Compute | `Standard_B2ats_v2`, 2 vCPU / approximately 1 GiB RAM |
| Mode | Neko-only; no GNOME, XFCE, KasmVNC, or XRDP |
| Runtime | Neko Firefox 3.1.4 and Caddy 2.11.4 |
| Constrained profile | `1024x576@20`, 1-GiB shared memory |
| Memory safety margin | 4-GiB persistent disk-backed swapfile |
| Public boundary | TCP 80/443/59000 and UDP 59000 |

This size is not presented as free or recommended. Static/reserved address status and
immutable image digests were not captured in the publishable evidence and therefore are
not claimed.

#### Reference: Azure case findings

1. The 1-GiB VM was deliberately limited to Neko-only. A full Ubuntu web desktop was
   rejected because it would not have safe memory headroom.
2. Existing Docker state was inventoried first. One stopped unrelated workload, its
   volume, and its image were removed by exact name; active Docker, networking, provider
   agent, SSH, and swap components were retained.
3. UFW and local listeners were correct, but the Azure NSG initially allowed only SSH.
   ACME timed out until the effective NSG allowed TCP 80/443/59000 and UDP 59000.
   Neko 8080 and RDP 3389 remained externally closed.
4. An official Azure CLI container was used only for device-authorized NSG remediation,
   then its container, writable auth state, exact image, and temporary scripts were
   removed.
5. After ingress was corrected, Caddy obtained a trusted certificate. External checks
   returned HTTP 200 with successful TLS verification; TCP exposure matched the contract,
   and a captured external datagram proved UDP 59000 reached the VM.
6. The first reboot restored Neko and Caddy but revealed that an existing active
   `/swapfile` lacked a persistent fstab entry. The entry was validated and added; a
   second reboot restored the 4-GiB swap, healthy containers, and zero failed services.

#### Reference: Azure acceptance boundary

The case proves installation, container health, trusted TLS, effective Azure/UFW
ingress, private 8080, transport-level UDP delivery, exact-target cleanup, persistent
swap, and reboot recovery. A captured UDP packet is not a WebRTC session. Authenticated
browser video/audio, ICE negotiation, mouse/keyboard control, and a second viewer remain
mandatory real-browser acceptance tests and are not claimed by this infrastructure-only
case.

### Reference: Project history

All notable project changes are recorded here. Dates use ISO 8601.

#### Reference: Unreleased

##### Reference: Added

- Three explicit deployment modes: Neko-only, Ubuntu GNOME/KasmVNC-only, and combined.
- Portable deployment templates, preflight/validation scripts, provider navigation,
  security policy, contribution guide, and continuous validation workflow.
- A publication-safe inventory template and ignore rules for rendered local operations
  material.

##### Reference: Changed

- Reorganized the monolithic runbook as a deep reference behind shorter tutorials.
- Removed live hostnames, public/private IPs, `/home/ubuntu`, and UID `1001` from
  publishable deployment assets.
- Replaced the host-specific landing page with a generic repository guide.

#### Reference: 2026-07-20

- Added the sanitized Azure AMD64 Neko-only infrastructure case study.
- Added existing-VM effective-NSG remediation and one-off official Azure CLI guidance.
- Changed swap validation to prove persistence after reboot rather than assuming an
  active swap source will return.
- Clarified the constrained 1-GiB Neko-only floor without lowering recommended sizing or
  claiming browser-level acceptance that was not performed.

#### Reference: 2026-07-19

- Initial end-to-end OCI reference deployment and multi-cloud operations runbook.

### Reference: Contributing and validation

Contributions that improve portability, safety, accessibility, or reproducibility are
welcome.

#### Reference: Before changing the project

1. Open an issue describing the target mode, cloud, CPU architecture, Ubuntu release,
   and observed behavior.
2. Never attach live credentials, private keys, cloud resource IDs, personal hostnames,
   public IPs, unredacted screenshots, or decrypted backups.
3. Keep Neko-only, desktop-only, and combined instructions independently usable.
4. Prefer official upstream documentation and link it next to time-sensitive claims.
5. Label commands by execution context: local workstation, cloud shell, or Ubuntu VM.

#### Reference: Validation

Keep tracked text in UTF-8 with LF line endings and a final newline. Trim trailing
whitespace except where Markdown requires it, indent YAML with two spaces, and
treat PNG, JPEG, GIF, WebP, ICO, PDF, and DEB artifacts as binary content.

Run from the repository root:

```bash
git diff --check
```

For the complete container-backed check, including Caddy parsing and mandatory
ShellCheck, run `make check-full`. The repository validation must parse every shell
script and Compose variant, reject committed secrets/rendered endpoints, and check
documentation links. [the post-deployment helper](#reference-helper-script-post-deployment-validation) saved as `/usr/local/sbin/neko-cloud-validate` is a separate post-deployment check that
runs on an installed VM and requires a mode argument. If a change cannot be tested end
to end, state exactly which provider/mode was not tested.

Do not silently update image tags, KasmVNC packages, checksums, firewall behavior, or
cloud-free-tier claims. Include the upstream source and a rollback note.

### Reference: Copyable deployment templates

These are the exact flat deployment assets preserved in this guide. Choose one mode and
save only that mode's three blocks to the labeled runtime filenames. Desktop-only and
combined deployments also require the four shared desktop blocks. Create parent
directories first, paste without Markdown fences, then apply the ownership and modes
shown in the tutorial. The `.env` blocks are examples: save as `/opt/neko-cloud/.env`,
replace every placeholder, and never commit the populated file.

#### Reference template: Neko-only runtime files

##### Reference template: Neko Compose — `/opt/neko-cloud/compose.yaml`

Source asset: `neko.compose.yaml`. Intended runtime filename: `/opt/neko-cloud/compose.yaml`.

```yaml
name: neko-cloud-browser

services:
  neko:
    image: ghcr.io/m1k1o/neko/firefox:3.1.4
    restart: unless-stopped
    init: true
    shm_size: '2gb'
    environment:
      NEKO_DESKTOP_SCREEN: '1280x720@24'
      NEKO_MEMBER_PROVIDER: 'multiuser'
      NEKO_MEMBER_MULTIUSER_USER_PASSWORD: '${NEKO_USER_PASSWORD:?Set NEKO_USER_PASSWORD in .env}'
      NEKO_MEMBER_MULTIUSER_ADMIN_PASSWORD: '${NEKO_ADMIN_PASSWORD:?Set NEKO_ADMIN_PASSWORD in .env}'
      NEKO_WEBRTC_UDPMUX: '59000'
      NEKO_WEBRTC_TCPMUX: '59000'
      NEKO_WEBRTC_ICELITE: 'true'
      NEKO_WEBRTC_NAT1TO1: '${PUBLIC_IP:?Set PUBLIC_IP in .env}'
      NEKO_SERVER_PROXY: 'true'
      # Compatibility settings for the pinned Neko 3.1.4 web client.
      NEKO_LEGACY: 'true'
      NEKO_SESSION_COOKIE_ENABLED: 'false'
    expose:
      - '8080'
    ports:
      - '59000:59000/udp'
      - '59000:59000/tcp'
    networks:
      - frontend
    logging:
      driver: json-file
      options:
        max-size: '10m'
        max-file: '3'

  caddy:
    image: caddy:2.11.4-alpine
    restart: unless-stopped
    depends_on:
      - neko
    environment:
      NEKO_HOST: '${NEKO_HOST:?Set NEKO_HOST in .env}'
      PUBLIC_IP: '${PUBLIC_IP:?Set PUBLIC_IP in .env}'
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - frontend
    logging:
      driver: json-file
      options:
        max-size: '10m'
        max-file: '3'

networks:
  frontend:

volumes:
  caddy_data:
  caddy_config:
```

##### Reference template: Neko Caddyfile — `/opt/neko-cloud/Caddyfile`

Source asset: `neko.Caddyfile`. Intended runtime filename: `/opt/neko-cloud/Caddyfile`.

```caddyfile
{
	admin off
	servers {
		protocols h1 h2
	}
}

{$NEKO_HOST} {
	encode zstd gzip
	reverse_proxy neko:8080

	header {
		Strict-Transport-Security "max-age=31536000"
		X-Content-Type-Options "nosniff"
		X-Frame-Options "SAMEORIGIN"
		Referrer-Policy "no-referrer"
	}
}

http://{$PUBLIC_IP} {
	redir https://{$NEKO_HOST}{uri} permanent
}
```

##### Reference template: Neko environment — `/opt/neko-cloud/.env`

Source asset: `neko.env.example`. Intended runtime filename after placeholder replacement: `/opt/neko-cloud/.env`.

```dotenv
# Copy to .env in the deployment directory and replace every example value.
# Keep the real .env out of Git and readable only by root (mode 0600).
PUBLIC_IP=203.0.113.10
NEKO_HOST=neko.example.com
NEKO_USER_PASSWORD=replace-with-a-long-random-password
NEKO_ADMIN_PASSWORD=replace-with-a-different-long-random-password
```

#### Reference template: Desktop-only runtime files

##### Reference template: Desktop Compose — `/opt/neko-cloud/compose.yaml`

Source asset: `desktop.compose.yaml`. Intended runtime filename: `/opt/neko-cloud/compose.yaml`.

```yaml
name: ubuntu-cloud-desktop

services:
  caddy:
    image: caddy:2.11.4-alpine
    restart: unless-stopped
    environment:
      DESKTOP_HOST: '${DESKTOP_HOST:?Set DESKTOP_HOST in .env}'
      PUBLIC_IP: '${PUBLIC_IP:?Set PUBLIC_IP in .env}'
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - /run/kasmvnc:/run/kasmvnc:ro
      - caddy_data:/data
      - caddy_config:/config
    logging:
      driver: json-file
      options:
        max-size: '10m'
        max-file: '3'

volumes:
  caddy_data:
  caddy_config:
```

##### Reference template: Desktop Caddyfile — `/opt/neko-cloud/Caddyfile`

Source asset: `desktop.Caddyfile`. Intended runtime filename: `/opt/neko-cloud/Caddyfile`.

```caddyfile
{
	admin off
	servers {
		protocols h1 h2
	}
}

{$DESKTOP_HOST} {
	reverse_proxy unix//run/kasmvnc/kasm.sock {
		header_up Host 127.0.0.1:8444
		flush_interval -1
	}

	header {
		Strict-Transport-Security "max-age=31536000"
		X-Content-Type-Options "nosniff"
		X-Frame-Options "SAMEORIGIN"
		Referrer-Policy "no-referrer"
	}
}

http://{$PUBLIC_IP} {
	redir https://{$DESKTOP_HOST}{uri} permanent
}
```

##### Reference template: Desktop environment — `/opt/neko-cloud/.env`

Source asset: `desktop.env.example`. Intended runtime filename after placeholder replacement: `/opt/neko-cloud/.env`.

```dotenv
# Copy to .env in the deployment directory and replace every example value.
# Keep the real .env out of Git. This mode stores no KasmVNC password here;
# create that credential interactively with vncpasswd as the desktop user.
PUBLIC_IP=203.0.113.10
DESKTOP_HOST=desktop.example.com
```

#### Reference template: Combined runtime files

##### Reference template: Combined Compose — `/opt/neko-cloud/compose.yaml`

Source asset: `combined.compose.yaml`. Intended runtime filename: `/opt/neko-cloud/compose.yaml`.

```yaml
name: neko-and-ubuntu-desktop

services:
  neko:
    image: ghcr.io/m1k1o/neko/firefox:3.1.4
    restart: unless-stopped
    init: true
    shm_size: '2gb'
    environment:
      NEKO_DESKTOP_SCREEN: '1280x720@24'
      NEKO_MEMBER_PROVIDER: 'multiuser'
      NEKO_MEMBER_MULTIUSER_USER_PASSWORD: '${NEKO_USER_PASSWORD:?Set NEKO_USER_PASSWORD in .env}'
      NEKO_MEMBER_MULTIUSER_ADMIN_PASSWORD: '${NEKO_ADMIN_PASSWORD:?Set NEKO_ADMIN_PASSWORD in .env}'
      NEKO_WEBRTC_UDPMUX: '59000'
      NEKO_WEBRTC_TCPMUX: '59000'
      NEKO_WEBRTC_ICELITE: 'true'
      NEKO_WEBRTC_NAT1TO1: '${PUBLIC_IP:?Set PUBLIC_IP in .env}'
      NEKO_SERVER_PROXY: 'true'
      # Compatibility settings for the pinned Neko 3.1.4 web client.
      NEKO_LEGACY: 'true'
      NEKO_SESSION_COOKIE_ENABLED: 'false'
    expose:
      - '8080'
    ports:
      - '59000:59000/udp'
      - '59000:59000/tcp'
    networks:
      - frontend
    logging:
      driver: json-file
      options:
        max-size: '10m'
        max-file: '3'

  caddy:
    image: caddy:2.11.4-alpine
    restart: unless-stopped
    depends_on:
      - neko
    environment:
      NEKO_HOST: '${NEKO_HOST:?Set NEKO_HOST in .env}'
      DESKTOP_HOST: '${DESKTOP_HOST:?Set DESKTOP_HOST in .env}'
      PUBLIC_IP: '${PUBLIC_IP:?Set PUBLIC_IP in .env}'
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - /run/kasmvnc:/run/kasmvnc:ro
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - frontend
    logging:
      driver: json-file
      options:
        max-size: '10m'
        max-file: '3'

networks:
  frontend:

volumes:
  caddy_data:
  caddy_config:
```

##### Reference template: Combined Caddyfile — `/opt/neko-cloud/Caddyfile`

Source asset: `combined.Caddyfile`. Intended runtime filename: `/opt/neko-cloud/Caddyfile`.

```caddyfile
{
	admin off
	servers {
		protocols h1 h2
	}
}

{$NEKO_HOST} {
	encode zstd gzip
	reverse_proxy neko:8080

	header {
		Strict-Transport-Security "max-age=31536000"
		X-Content-Type-Options "nosniff"
		X-Frame-Options "SAMEORIGIN"
		Referrer-Policy "no-referrer"
	}
}

{$DESKTOP_HOST} {
	reverse_proxy unix//run/kasmvnc/kasm.sock {
		header_up Host 127.0.0.1:8444
		flush_interval -1
	}

	header {
		Strict-Transport-Security "max-age=31536000"
		X-Content-Type-Options "nosniff"
		X-Frame-Options "SAMEORIGIN"
		Referrer-Policy "no-referrer"
	}
}

http://{$PUBLIC_IP} {
	redir https://{$NEKO_HOST}{uri} permanent
}
```

##### Reference template: Combined environment — `/opt/neko-cloud/.env`

Source asset: `combined.env.example`. Intended runtime filename after placeholder replacement: `/opt/neko-cloud/.env`.

```dotenv
# Copy to .env in the deployment directory and replace every example value.
# Keep the real .env out of Git and readable only by root (mode 0600).
# The KasmVNC password is created separately and is never stored here.
PUBLIC_IP=203.0.113.10
NEKO_HOST=neko.example.com
DESKTOP_HOST=desktop.example.com
NEKO_USER_PASSWORD=replace-with-a-long-random-password
NEKO_ADMIN_PASSWORD=replace-with-a-different-long-random-password
```

#### Reference template: Shared desktop runtime files

Use these exact four blocks for both desktop-only and combined modes.

##### Reference template: KasmVNC configuration — `/home/desktop/.vnc/kasmvnc.yaml`

Source asset: `kasmvnc.yaml`. Intended runtime filename: `/home/desktop/.vnc/kasmvnc.yaml`.

```yaml
desktop:
  resolution:
    width: 1280
    height: 720
  allow_resize: true
  pixel_depth: 24

network:
  protocol: http
  interface: 127.0.0.1
  websocket_port: 8444
  use_ipv4: true
  use_ipv6: false
  udp:
    public_ip: 127.0.0.1
    port: 8444
  ssl:
    pem_certificate: /home/desktop/.vnc/kasmvnc.crt
    pem_key: /home/desktop/.vnc/kasmvnc.key
    require_ssl: false

user_session:
  session_type: exclusive
  new_session_disconnects_existing_exclusive_session: true
  concurrent_connections_prompt: false
  idle_timeout: 1800

runtime_configuration:
  allow_client_to_override_kasm_server_settings: false

encoding:
  max_frame_rate: 30

server:
  auto_shutdown:
    no_user_session_timeout: never
    active_user_session_timeout: never
    inactive_user_session_timeout: never

command_line:
  prompt: false
```

##### Reference template: GNOME startup — `/home/desktop/.vnc/xstartup`

Source asset: `xstartup`. Intended runtime filename: `/home/desktop/.vnc/xstartup` (executable).

```bash
#!/bin/sh
set -eu

unset SESSION_MANAGER
unset WAYLAND_DISPLAY

export XDG_CONFIG_HOME=${XDG_CONFIG_HOME:-${HOME}/.config}
export XDG_DATA_HOME=${XDG_DATA_HOME:-${HOME}/.local/share}
export XDG_CACHE_HOME=${XDG_CACHE_HOME:-${HOME}/.cache}
export XDG_CONFIG_DIRS=/etc/xdg/xdg-ubuntu:${XDG_CONFIG_DIRS:-/etc/xdg}
export XDG_DATA_DIRS=/usr/share/ubuntu:/usr/share/gnome:${XDG_DATA_DIRS:-/usr/local/share:/usr/share}:/var/lib/snapd/desktop
export XDG_SESSION_TYPE=x11
export XDG_SESSION_CLASS=user
export DESKTOP_SESSION=ubuntu
export XDG_SESSION_DESKTOP=ubuntu
export XDG_CURRENT_DESKTOP=ubuntu:GNOME
export GNOME_SHELL_SESSION_MODE=ubuntu
export GDMSESSION=ubuntu
export XAUTHORITY=${XAUTHORITY:-${HOME}/.Xauthority}

mkdir -p "$XDG_CONFIG_HOME" "$XDG_DATA_HOME" "$XDG_CACHE_HOME"

if [ -z "${XDG_RUNTIME_DIR:-}" ] || [ ! -S "${XDG_RUNTIME_DIR}/bus" ]; then
  echo 'GNOME requires the systemd user runtime directory and user D-Bus' >&2
  exit 1
fi

export DBUS_SESSION_BUS_ADDRESS=unix:path=${XDG_RUNTIME_DIR}/bus
dbus-update-activation-environment --systemd \
  DISPLAY XAUTHORITY DBUS_SESSION_BUS_ADDRESS \
  XDG_SESSION_TYPE XDG_SESSION_CLASS DESKTOP_SESSION XDG_SESSION_DESKTOP \
  XDG_CURRENT_DESKTOP GNOME_SHELL_SESSION_MODE GDMSESSION \
  XDG_CONFIG_DIRS XDG_DATA_DIRS

exec /usr/bin/gnome-session --session=ubuntu
```

##### Reference template: KasmVNC user service — `/home/desktop/.config/systemd/user/kasmvncserver@.service`

Source asset: `kasmvncserver@.service`. Intended runtime filename: `/home/desktop/.config/systemd/user/kasmvncserver@.service`.

```text
[Unit]
Description=KasmVNC Ubuntu desktop on display %i
After=network.target

[Service]
Type=forking
Environment=XDG_RUNTIME_DIR=%t
Environment=DBUS_SESSION_BUS_ADDRESS=unix:path=%t/bus
Environment=XDG_CONFIG_DIRS=/etc/xdg/xdg-ubuntu:/etc/xdg
Environment=XDG_DATA_DIRS=/usr/share/ubuntu:/usr/share/gnome:/usr/local/share:/usr/share:/var/lib/snapd/desktop
Environment=XDG_SESSION_TYPE=x11
Environment=XDG_SESSION_CLASS=user
Environment=DESKTOP_SESSION=ubuntu
Environment=XDG_SESSION_DESKTOP=ubuntu
Environment=XDG_CURRENT_DESKTOP=ubuntu:GNOME
Environment=GNOME_SHELL_SESSION_MODE=ubuntu
Environment=GDMSESSION=ubuntu
ExecStartPre=-/usr/bin/kasmvncserver -kill %i
ExecStart=/usr/bin/kasmvncserver %i
ExecStop=/usr/bin/kasmvncserver -kill %i
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=default.target
```

##### Reference template: KasmVNC socket bridge — `/etc/systemd/system/kasmvnc-socat.service`

Source asset: `kasmvnc-socat.service`. Intended runtime filename: `/etc/systemd/system/kasmvnc-socat.service`.

```text
[Unit]
Description=Private Unix-socket bridge from Caddy to KasmVNC
After=network.target

[Service]
Type=simple
ExecStartPre=/usr/bin/install -d -m 0755 /run/kasmvnc
ExecStartPre=-/usr/bin/rm -f /run/kasmvnc/kasm.sock
ExecStart=/usr/bin/socat UNIX-LISTEN:/run/kasmvnc/kasm.sock,fork,mode=0660,user=root,group=root TCP:127.0.0.1:8444
ExecStopPost=-/usr/bin/rm -f /run/kasmvnc/kasm.sock
Restart=always
RestartSec=2s

[Install]
WantedBy=multi-user.target
```

### Reference: Optional original helper scripts

The deployment tutorials are complete in Markdown. The following three original helper
sources are retained verbatim for lossless reproduction. To use the read-only VM checks,
save the preflight and post-deployment blocks to the labeled `/usr/local/sbin` filenames,
set owner `root:root` and mode `0755`, and invoke the names shown by the tutorials. Their
historical usage banners still mention the former folder-based paths. The repository
checker expects that former multi-folder layout and is reference-only in this four-file
documentation repository.

#### Reference helper script: Preflight

Intended optional runtime filename: `/usr/local/sbin/neko-cloud-preflight`.

```bash
#!/usr/bin/env bash
# Read-only prerequisite and configuration checks for one deployment mode.
set -euo pipefail

usage() {
  cat >&2 <<'EOF'
Usage: sudo ./scripts/preflight.sh <neko|desktop|combined> [deployment-directory]

The deployment directory defaults to /opt/neko-cloud. It must contain compose.yaml,
Caddyfile, and a populated mode-specific .env.

Set SKIP_DNS_CHECK=1 only while DNS records are deliberately still propagating.
EOF
  exit 2
}

fail() { printf 'FAIL: %s\n' "$*" >&2; exit 1; }
pass() { printf 'PASS: %s\n' "$*"; }
warn() { printf 'WARN: %s\n' "$*" >&2; }

require_command() {
  command -v "$1" >/dev/null 2>&1 || fail "required command is missing: $1"
}

require_file() {
  [ -f "$1" ] || fail "required file is missing: $1"
}

dotenv_count() {
  awk -F= -v wanted="$1" '$1 == wanted { count++ } END { print count + 0 }' "$ENV_FILE"
}

dotenv_value() {
  awk -F= -v wanted="$1" '
    $1 == wanted { value = substr($0, index($0, "=") + 1) }
    END { sub(/\r$/, "", value); print value }
  ' "$ENV_FILE"
}

require_env_value() {
  local key=$1 count value
  count=$(dotenv_count "$key")
  [ "$count" -eq 1 ] || fail "$ENV_FILE must contain exactly one $key= line"
  value=$(dotenv_value "$key")
  [ -n "$value" ] || fail "$key is empty in $ENV_FILE"
  case "$value" in
    *replace-with*|*CHANGE-ME*|*change-me*) fail "$key still contains an example placeholder" ;;
  esac
  printf '%s' "$value"
}

validate_public_ip() {
  python3 - "$1" <<'PY'
import ipaddress
import sys

try:
    address = ipaddress.ip_address(sys.argv[1])
except ValueError as exc:
    raise SystemExit(f"invalid PUBLIC_IP: {exc}")
if address.version != 4 or not address.is_global:
    raise SystemExit("PUBLIC_IP must be a globally routable IPv4 address")
PY
}

validate_fqdn() {
  python3 - "$1" <<'PY'
import re
import sys

name = sys.argv[1]
if name.endswith('.'):
    name = name[:-1]
if len(name) > 253 or '.' not in name:
    raise SystemExit(f"not a valid fully-qualified hostname: {sys.argv[1]}")
label = re.compile(r"^(?!-)[A-Za-z0-9-]{1,63}(?<!-)$")
if not all(label.fullmatch(part) for part in name.split('.')):
    raise SystemExit(f"not a valid fully-qualified hostname: {sys.argv[1]}")
PY
}

check_dns() {
  local host=$1 expected=$2 resolved
  if [ "${SKIP_DNS_CHECK:-0}" = 1 ]; then
    warn "DNS check skipped for $host"
    return
  fi
  resolved=$(getent ahostsv4 "$host" | awk '{ print $1 }' | sort -u || true)
  [ -n "$resolved" ] || fail "$host has no IPv4 DNS result"
  [ "$resolved" = "$expected" ] || {
    printf 'Resolved addresses for %s:\n%s\n' "$host" "$resolved" >&2
    fail "$host must resolve only to PUBLIC_IP $expected"
  }
  pass "$host resolves to $expected"
}

[ "$#" -ge 1 ] && [ "$#" -le 2 ] || usage
MODE=$1
case "$MODE" in
  neko|desktop|combined) ;;
  *) usage ;;
esac
[ "$(id -u)" -eq 0 ] || fail 'run this read-only check with sudo'

if [ "$#" -eq 2 ]; then
  DEPLOY_DIR=$2
else
  DEPLOY_DIR=/opt/neko-cloud
fi
[ -d "$DEPLOY_DIR" ] || fail "deployment directory does not exist: $DEPLOY_DIR"
DEPLOY_DIR=$(CDPATH= cd -- "$DEPLOY_DIR" && pwd -P)
ENV_FILE=$DEPLOY_DIR/.env

for command_name in awk curl docker getent grep openssl python3 sed sort ss stat; do
  require_command "$command_name"
done
docker compose version >/dev/null 2>&1 || fail 'Docker Compose v2 plugin is unavailable'
require_file "$DEPLOY_DIR/compose.yaml"
require_file "$DEPLOY_DIR/Caddyfile"
require_file "$ENV_FILE"

env_mode=$(stat -c '%a' "$ENV_FILE")
[ "$env_mode" = 600 ] || fail "$ENV_FILE must have mode 0600, not $env_mode"
[ "$(stat -c '%u' "$ENV_FILE")" -eq 0 ] || fail "$ENV_FILE must be owned by root"
pass 'deployment files and protected environment file exist'

[ -r /etc/os-release ] || fail '/etc/os-release is unavailable'
os_id=$(awk -F= '$1 == "ID" { gsub(/\"/, "", $2); print $2 }' /etc/os-release)
os_version=$(awk -F= '$1 == "VERSION_ID" { gsub(/\"/, "", $2); print $2 }' /etc/os-release)
[ "$os_id" = ubuntu ] || fail "supported baseline is Ubuntu, found $os_id"
[ "$os_version" = 24.04 ] || fail "supported baseline is Ubuntu 24.04, found $os_version"
case "$(uname -m)" in
  x86_64|aarch64) ;;
  *) fail "unsupported CPU architecture: $(uname -m)" ;;
esac
pass "Ubuntu $os_version on $(uname -m) is supported"

PUBLIC_IP=$(require_env_value PUBLIC_IP)
validate_public_ip "$PUBLIC_IP" || fail 'PUBLIC_IP validation failed'
pass 'PUBLIC_IP is a globally routable IPv4 address'

if [ "$MODE" = neko ] || [ "$MODE" = combined ]; then
  NEKO_HOST=$(require_env_value NEKO_HOST)
  NEKO_USER_PASSWORD=$(require_env_value NEKO_USER_PASSWORD)
  NEKO_ADMIN_PASSWORD=$(require_env_value NEKO_ADMIN_PASSWORD)
  validate_fqdn "$NEKO_HOST" || fail 'NEKO_HOST validation failed'
  [ "$NEKO_HOST" != neko.example.com ] || fail 'NEKO_HOST still uses the example domain'
  [ "${#NEKO_USER_PASSWORD}" -ge 24 ] || fail 'NEKO_USER_PASSWORD must be at least 24 characters'
  [ "${#NEKO_ADMIN_PASSWORD}" -ge 24 ] || fail 'NEKO_ADMIN_PASSWORD must be at least 24 characters'
  [ "$NEKO_USER_PASSWORD" != "$NEKO_ADMIN_PASSWORD" ] || fail 'Neko user and administrator passwords must differ'
  check_dns "$NEKO_HOST" "$PUBLIC_IP"
  unset NEKO_USER_PASSWORD NEKO_ADMIN_PASSWORD
fi

if [ "$MODE" = desktop ] || [ "$MODE" = combined ]; then
  DESKTOP_HOST=$(require_env_value DESKTOP_HOST)
  validate_fqdn "$DESKTOP_HOST" || fail 'DESKTOP_HOST validation failed'
  [ "$DESKTOP_HOST" != desktop.example.com ] || fail 'DESKTOP_HOST still uses the example domain'
  check_dns "$DESKTOP_HOST" "$PUBLIC_IP"
  if [ "$MODE" = combined ]; then
    [ "$NEKO_HOST" != "$DESKTOP_HOST" ] || fail 'NEKO_HOST and DESKTOP_HOST must be different'
  fi

  require_file /home/desktop/.vnc/kasmvnc.yaml
  require_file /home/desktop/.vnc/xstartup
  require_file '/home/desktop/.config/systemd/user/kasmvncserver@.service'
  require_file /etc/systemd/system/kasmvnc-socat.service
  for command_name in dbus-update-activation-environment gnome-session kasmvncserver loginctl runuser socat systemctl; do
    require_command "$command_name"
  done

  grep -Fq 'Environment=XDG_RUNTIME_DIR=%t' '/home/desktop/.config/systemd/user/kasmvncserver@.service' || fail 'Kasm user unit must use systemd %t'
  if grep -Eq '/run/user/[0-9]+' '/home/desktop/.config/systemd/user/kasmvncserver@.service'; then
    fail 'Kasm user unit contains a hardcoded numeric UID'
  fi
  grep -Fq 'interface: 127.0.0.1' /home/desktop/.vnc/kasmvnc.yaml || fail 'KasmVNC must bind to loopback'
  grep -Fq '/home/desktop/.vnc/kasmvnc.crt' /home/desktop/.vnc/kasmvnc.yaml || fail 'Kasm config does not target /home/desktop'

  installed_kasm=$(dpkg-query -W -f='${Version}' kasmvncserver 2>/dev/null || true)
  case "$installed_kasm" in
    1.4.0-1) ;;
    '') fail 'KasmVNC package is not installed' ;;
    *) fail "expected KasmVNC 1.4.0-1, found $installed_kasm" ;;
  esac
  pass 'GNOME/KasmVNC commands, version, and installed templates are compatible'
fi

docker compose --env-file "$ENV_FILE" -f "$DEPLOY_DIR/compose.yaml" config --quiet
pass 'Docker Compose interpolation and schema validation succeeded'
if [ "$MODE" = neko ] || [ "$MODE" = combined ]; then
  grep -Fq 'ghcr.io/m1k1o/neko/firefox:3.1.4' "$DEPLOY_DIR/compose.yaml" || fail 'Neko image is not pinned to 3.1.4'
fi
grep -Fq 'caddy:2.11.4-alpine' "$DEPLOY_DIR/compose.yaml" || fail 'Caddy image is not pinned to 2.11.4-alpine'
pass "preflight completed successfully for $MODE"
```

#### Reference helper script: Post-deployment validation

Intended optional runtime filename: `/usr/local/sbin/neko-cloud-validate`.

```bash
#!/usr/bin/env bash
# Read-only post-deployment checks for one deployment mode.
set -euo pipefail

usage() {
  cat >&2 <<'EOF'
Usage: sudo ./scripts/validate.sh <neko|desktop|combined> [deployment-directory]

The deployment directory defaults to /opt/neko-cloud. The check does not install,
start, stop, restart, or otherwise mutate services.
EOF
  exit 2
}

fail() { printf 'FAIL: %s\n' "$*" >&2; exit 1; }
pass() { printf 'PASS: %s\n' "$*"; }

require_command() {
  command -v "$1" >/dev/null 2>&1 || fail "required command is missing: $1"
}

dotenv_value() {
  awk -F= -v wanted="$1" '
    $1 == wanted { value = substr($0, index($0, "=") + 1) }
    END { sub(/\r$/, "", value); print value }
  ' "$ENV_FILE"
}

service_is_running() {
  docker compose --env-file "$ENV_FILE" -f "$DEPLOY_DIR/compose.yaml" \
    ps --status running --services | grep -Fqx "$1"
}

tcp_listeners() {
  ss -H -lnt | awk -v suffix=":$1" '$4 ~ (suffix "$")'
}

udp_listeners() {
  ss -H -lnu | awk -v suffix=":$1" '$4 ~ (suffix "$")'
}

require_tcp_listener() {
  [ -n "$(tcp_listeners "$1")" ] || fail "TCP $1 is not listening"
}

require_udp_listener() {
  [ -n "$(udp_listeners "$1")" ] || fail "UDP $1 is not listening"
}

require_no_tcp_listener() {
  [ -z "$(tcp_listeners "$1")" ] || {
    tcp_listeners "$1" >&2
    fail "TCP $1 must not be listening on the host"
  }
}

require_loopback_only_tcp() {
  local port=$1 listeners local_address
  listeners=$(tcp_listeners "$port")
  [ -n "$listeners" ] || fail "TCP $port is not listening"
  while IFS= read -r line; do
    local_address=$(printf '%s\n' "$line" | awk '{ print $4 }')
    case "$local_address" in
      127.0.0.1:"$port"|'[::1]':"$port") ;;
      *) printf '%s\n' "$line" >&2; fail "TCP $port is exposed beyond loopback" ;;
    esac
  done <<EOF
$listeners
EOF
}

require_no_public_udp() {
  local port=$1 listeners local_address
  listeners=$(udp_listeners "$port")
  [ -z "$listeners" ] && return
  while IFS= read -r line; do
    local_address=$(printf '%s\n' "$line" | awk '{ print $4 }')
    case "$local_address" in
      127.0.0.1:"$port"|'[::1]':"$port") ;;
      *) printf '%s\n' "$line" >&2; fail "UDP $port is exposed beyond loopback" ;;
    esac
  done <<EOF
$listeners
EOF
}

https_code() {
  local host=$1 code attempt
  for ((attempt = 1; attempt <= 18; attempt++)); do
    code=$(curl --silent --output /dev/null --write-out '%{http_code}' \
      --noproxy '*' --connect-timeout 5 --max-time 15 \
      --resolve "$host:443:127.0.0.1" "https://$host/" 2>/dev/null || true)
    case "$code" in
      [1-5][0-9][0-9]) printf '%s' "$code"; return ;;
    esac
    sleep 5
  done
  printf '000'
}

wait_for_neko_backend() {
  local attempt
  for ((attempt = 1; attempt <= 18; attempt++)); do
    if docker compose --env-file "$ENV_FILE" -f "$DEPLOY_DIR/compose.yaml" \
      exec -T caddy wget -qO- http://neko:8080/ >/dev/null 2>&1; then
      return
    fi
    sleep 5
  done
  return 1
}

[ "$#" -ge 1 ] && [ "$#" -le 2 ] || usage
MODE=$1
case "$MODE" in
  neko|desktop|combined) ;;
  *) usage ;;
esac
[ "$(id -u)" -eq 0 ] || fail 'run this read-only check with sudo'

if [ "$#" -eq 2 ]; then
  DEPLOY_DIR=$2
else
  DEPLOY_DIR=/opt/neko-cloud
fi
[ -d "$DEPLOY_DIR" ] || fail "deployment directory does not exist: $DEPLOY_DIR"
DEPLOY_DIR=$(CDPATH= cd -- "$DEPLOY_DIR" && pwd -P)
ENV_FILE=$DEPLOY_DIR/.env
[ -f "$ENV_FILE" ] || fail "missing $ENV_FILE"
[ "$(stat -c '%a' "$ENV_FILE")" = 600 ] || fail "$ENV_FILE must have mode 0600"
[ "$(stat -c '%u' "$ENV_FILE")" -eq 0 ] || fail "$ENV_FILE must be owned by root"

for command_name in awk curl docker dpkg-query getent grep id loginctl passwd runuser ss stat systemctl; do
  require_command "$command_name"
done
docker compose --env-file "$ENV_FILE" -f "$DEPLOY_DIR/compose.yaml" config --quiet
pass 'Compose configuration is valid'

service_is_running caddy || fail 'Caddy container is not running'
docker compose --env-file "$ENV_FILE" -f "$DEPLOY_DIR/compose.yaml" \
  exec -T caddy caddy validate --config /etc/caddy/Caddyfile >/dev/null
caddy_version=$(docker compose --env-file "$ENV_FILE" -f "$DEPLOY_DIR/compose.yaml" \
  exec -T caddy caddy version)
case "$caddy_version" in
  v2.11.4*) ;;
  *) fail "expected Caddy 2.11.4, running $caddy_version" ;;
esac
pass 'Caddy 2.11.4 is running and its loaded configuration validates'

require_tcp_listener 80
require_tcp_listener 443
require_no_tcp_listener 8080
require_no_tcp_listener 5901
require_no_tcp_listener 3389
pass 'gateway ports are present and prohibited host TCP ports are absent'

if [ "$MODE" = neko ] || [ "$MODE" = combined ]; then
  service_is_running neko || fail 'Neko container is not running'
  require_tcp_listener 59000
  require_udp_listener 59000
  wait_for_neko_backend || fail 'Neko backend did not become ready before the timeout'
  NEKO_HOST=$(dotenv_value NEKO_HOST)
  neko_code=$(https_code "$NEKO_HOST")
  case "$neko_code" in
    2??|3??) ;;
    *) fail "Neko HTTPS endpoint returned HTTP $neko_code" ;;
  esac
  pass "Neko HTTPS and WebRTC mux listeners passed ($neko_code)"
else
  require_no_tcp_listener 59000
  [ -z "$(udp_listeners 59000)" ] || fail 'UDP 59000 must be absent in desktop-only mode'
fi

if [ "$MODE" = desktop ] || [ "$MODE" = combined ]; then
  desktop_entry=$(getent passwd desktop || true)
  [ -n "$desktop_entry" ] || fail 'locked Linux account desktop does not exist'
  desktop_home=$(printf '%s\n' "$desktop_entry" | awk -F: '{ print $6 }')
  [ "$desktop_home" = /home/desktop ] || fail "desktop account home must be /home/desktop, found $desktop_home"
  [ -d /home/desktop ] || fail '/home/desktop does not exist'
  [ "$(stat -c '%U:%G' /home/desktop)" = desktop:desktop ] || fail '/home/desktop must be owned by desktop:desktop'
  password_state=$(passwd -S desktop | awk '{ print $2 }')
  [ "$password_state" = L ] || fail 'Linux password for desktop must remain locked'
  desktop_groups=" $(id -nG desktop) "
  for forbidden_group in adm sudo docker lxd; do
    case "$desktop_groups" in
      *" $forbidden_group "*) fail "desktop must not belong to $forbidden_group" ;;
    esac
  done

  [ -f /home/desktop/.kasmpasswd ] || fail '/home/desktop/.kasmpasswd is missing'
  [ "$(stat -c '%a' /home/desktop/.kasmpasswd)" = 600 ] || fail '/home/desktop/.kasmpasswd must have mode 0600'
  [ "$(stat -c '%U:%G' /home/desktop/.kasmpasswd)" = desktop:desktop ] || fail '.kasmpasswd must be owned by desktop:desktop'
  [ -f /home/desktop/.vnc/kasmvnc.yaml ] || fail 'installed kasmvnc.yaml is missing'
  [ -x /home/desktop/.vnc/xstartup ] || fail 'installed xstartup is missing or not executable'
  [ -f '/home/desktop/.config/systemd/user/kasmvncserver@.service' ] || fail 'installed KasmVNC user unit is missing'
  [ -f /etc/systemd/system/kasmvnc-socat.service ] || fail 'installed KasmVNC bridge unit is missing'
  [ -f /home/desktop/.vnc/kasmvnc.crt ] || fail 'KasmVNC placeholder certificate is missing'
  [ -f /home/desktop/.vnc/kasmvnc.key ] || fail 'KasmVNC placeholder private key is missing'
  for private_file in \
    /home/desktop/.vnc/kasmvnc.yaml \
    /home/desktop/.vnc/kasmvnc.crt \
    /home/desktop/.vnc/kasmvnc.key; do
    [ "$(stat -c '%a' "$private_file")" = 600 ] || fail "$private_file must have mode 0600"
    [ "$(stat -c '%U:%G' "$private_file")" = desktop:desktop ] || fail "$private_file must be owned by desktop:desktop"
  done
  [ "$(stat -c '%a %U:%G' /home/desktop/.vnc/xstartup)" = '755 desktop:desktop' ] || fail 'xstartup must be mode 0755 and owned by desktop:desktop'
  [ "$(stat -c '%a %U:%G' '/home/desktop/.config/systemd/user/kasmvncserver@.service')" = '644 desktop:desktop' ] || fail 'KasmVNC user unit must be mode 0644 and owned by desktop:desktop'
  [ "$(stat -c '%a %U:%G' /etc/systemd/system/kasmvnc-socat.service)" = '644 root:root' ] || fail 'KasmVNC bridge unit must be mode 0644 and owned by root:root'
  grep -Fq 'Environment=XDG_RUNTIME_DIR=%t' '/home/desktop/.config/systemd/user/kasmvncserver@.service' || fail 'Kasm user unit must use systemd %t'
  if grep -Eq '/run/user/[0-9]+' '/home/desktop/.config/systemd/user/kasmvncserver@.service'; then
    fail 'Kasm user unit contains a hardcoded numeric UID'
  fi
  [ "$(loginctl show-user desktop -p Linger --value)" = yes ] || fail 'systemd linger is not enabled for desktop'
  [ "$(systemctl get-default)" = multi-user.target ] || fail 'headless host default target must be multi-user.target'
  [ "$(systemctl is-enabled gdm3.service 2>/dev/null || true)" = masked ] || fail 'gdm3.service must remain masked'
  [ "$(dpkg-query -W -f='${Version}' kasmvncserver 2>/dev/null || true)" = 1.4.0-1 ] || fail 'installed KasmVNC package must be 1.4.0-1'

  [ -S /run/kasmvnc/kasm.sock ] || fail '/run/kasmvnc/kasm.sock is missing or is not a Unix socket'
  [ "$(stat -c '%a' /run/kasmvnc/kasm.sock)" = 660 ] || fail 'KasmVNC bridge socket must have mode 0660'
  systemctl is-active --quiet kasmvnc-socat.service || fail 'kasmvnc-socat.service is not active'
  systemctl is-active --quiet gnome-remote-desktop.service || fail 'GNOME handover service is not active'

  desktop_uid=$(id -u desktop)
  runuser -u desktop -- env \
    HOME=/home/desktop USER=desktop LOGNAME=desktop \
    XDG_RUNTIME_DIR="/run/user/$desktop_uid" \
    DBUS_SESSION_BUS_ADDRESS="unix:path=/run/user/$desktop_uid/bus" \
    systemctl --user is-active --quiet 'kasmvncserver@:1.service' \
    || fail 'desktop KasmVNC user service is not active'

  require_loopback_only_tcp 8444
  require_no_public_udp 8444
  kasm_code=$(curl --silent --show-error --output /dev/null --write-out '%{http_code}' \
    --noproxy '*' --connect-timeout 5 --max-time 15 \
    --unix-socket /run/kasmvnc/kasm.sock \
    http://localhost/)
  [ "$kasm_code" = 401 ] || fail "private KasmVNC bridge returned HTTP $kasm_code instead of 401"

  DESKTOP_HOST=$(dotenv_value DESKTOP_HOST)
  desktop_code=$(https_code "$DESKTOP_HOST")
  [ "$desktop_code" = 401 ] || fail "desktop HTTPS endpoint returned HTTP $desktop_code instead of 401"
  pass 'GNOME/KasmVNC service, private bridge, TLS, and authentication challenge passed'
else
  require_no_tcp_listener 8444
  require_no_public_udp 8444
fi

pass "post-deployment validation completed successfully for $MODE"
```

#### Reference helper script: Original repository checker

Original helper filename: `check-repository.sh`. It is preserved as readable source and is not required by the standalone tutorial.

```bash
#!/usr/bin/env bash
# Static, read-only checks for the publishable repository.
set -euo pipefail

fail() { printf 'FAIL: %s\n' "$*" >&2; exit 1; }
pass() { printf 'PASS: %s\n' "$*"; }
warn() { printf 'WARN: %s\n' "$*" >&2; }

SCRIPT_DIR=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd -P)
REPO_ROOT=$(CDPATH= cd -- "$SCRIPT_DIR/.." && pwd -P)
cd "$REPO_ROOT"

mapfile -d '' SHELL_FILES < <(
  find scripts -type f -name '*.sh' -print0 | sort -z
)
SHELL_FILES+=(deploy/xstartup)
[ "${#SHELL_FILES[@]}" -gt 0 ] || fail 'no shell files found'
for file in "${SHELL_FILES[@]}"; do
  bash -n "$file"
done
pass "Bash syntax passed for ${#SHELL_FILES[@]} files"

if command -v shellcheck >/dev/null 2>&1; then
  shellcheck --severity=warning "${SHELL_FILES[@]}"
  pass 'ShellCheck passed'
elif [ "${REQUIRE_SHELLCHECK:-0}" = 1 ]; then
  fail 'ShellCheck is required but unavailable'
else
  warn 'ShellCheck is unavailable; CI requires it'
fi

command -v docker >/dev/null 2>&1 || fail 'Docker is required for Compose validation'
docker compose version >/dev/null 2>&1 || fail 'Docker Compose v2 is required'

for mode in neko desktop combined; do
  compose_file="deploy/$mode.compose.yaml"
  env_file="deploy/$mode.env.example"
  docker compose --env-file "$env_file" -f "$compose_file" config --quiet
  services=$(docker compose --env-file "$env_file" \
    -f "$compose_file" config --services | sort | tr '\n' ' ')
  case "$mode:$services" in
    'neko:caddy neko '|'desktop:caddy '|'combined:caddy neko ') ;;
    *) fail "unexpected Compose services for $mode: $services" ;;
  esac
done
pass 'all three Compose variants parsed with the expected services'

if grep -RInE --exclude-dir=.git --exclude-dir=.local \
  '(-----BEGIN [A-Z ]*PRIVATE KEY-----|AKIA[0-9A-Z]{16})' .; then
  fail 'private-key or access-key material is present'
fi

if grep -RInE '(/home/ubuntu|/run/user/[0-9]+)' deploy; then
  fail 'deployable assets contain a host-specific home or numeric runtime UID'
fi

if grep -RInE 'NEKO_WEBRTC_NAT1TO1:.*[0-9]{1,3}\.' deploy; then
  fail 'a deployable Neko configuration contains a rendered public IP'
fi

grep -Fq 'interface: 127.0.0.1' deploy/kasmvnc.yaml \
  || fail 'KasmVNC must bind to loopback'
grep -Fq 'Environment=XDG_RUNTIME_DIR=%t' 'deploy/kasmvncserver@.service' \
  || fail 'the user unit must use systemd %t'
grep -Fq 'WantedBy=default.target' 'deploy/kasmvncserver@.service' \
  || fail 'the user unit has no default.target install target'
grep -Fq 'WantedBy=multi-user.target' deploy/kasmvnc-socat.service \
  || fail 'the bridge unit has no multi-user.target install target'
grep -Fq 'exec /usr/bin/gnome-session --session=ubuntu' deploy/xstartup \
  || fail 'xstartup does not launch the Ubuntu GNOME session'
pass 'KasmVNC and systemd template invariants passed'

forbidden_file=$(find . -path './.git' -prune -o -path './.local' -prune -o \
  -type f \( -name '.env' -o -name '.kasmpasswd' -o -name '*.key' \
  -o -name '*.pem' -o -name '*.tfstate' -o -name '*.tfvars' \) -print -quit)
if [ -n "$forbidden_file" ]; then
  printf '%s\n' "$forbidden_file" >&2
  fail 'a credential/rendered-state filename is present outside ignored local storage'
fi

if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  tracked_local=$(git ls-files -- .local)
  [ -z "$tracked_local" ] || {
    printf '%s\n' "$tracked_local" >&2
    fail '.local must not be tracked by Git'
  }
fi
pass 'secret-boundary and portability scans passed'

if command -v python3 >/dev/null 2>&1; then
  PYTHON=python3
elif command -v python >/dev/null 2>&1; then
  PYTHON=python
else
  fail 'Python 3 is required for Markdown validation'
fi

"$PYTHON" - "$REPO_ROOT" <<'PY'
from __future__ import annotations

import re
import sys
import urllib.parse
from pathlib import Path

root = Path(sys.argv[1]).resolve()
markdown_files = sorted(
    path for path in root.rglob("*.md")
    if ".git" not in path.parts and ".local" not in path.parts
)
errors: list[str] = []

def slugify(value: str) -> str:
    value = value.strip().lower()
    value = re.sub(r"[^\w\- ]", "", value, flags=re.UNICODE)
    return re.sub(r"[ ]+", "-", value)

heading_cache: dict[Path, set[str]] = {}
def headings(path: Path) -> set[str]:
    if path not in heading_cache:
        result: set[str] = set()
        seen: dict[str, int] = {}
        for line in path.read_text(encoding="utf-8").splitlines():
            match = re.match(r"^#{1,6}\s+(.+?)\s*#*\s*$", line)
            if not match:
                continue
            base = slugify(match.group(1))
            count = seen.get(base, 0)
            seen[base] = count + 1
            result.add(base if count == 0 else f"{base}-{count}")
        heading_cache[path] = result
    return heading_cache[path]

link_pattern = re.compile(r"(?<!!)\[[^\]]*\]\(([^)\s]+)(?:\s+['\"][^'\"]*['\"])?\)")
for source in markdown_files:
    text = source.read_text(encoding="utf-8")
    if sum(1 for line in text.splitlines() if line.startswith("```")) % 2:
        errors.append(f"{source.relative_to(root)}: unbalanced fenced code block")
    for raw in link_pattern.findall(text):
        target = raw.strip("<>")
        parsed = urllib.parse.urlsplit(target)
        if parsed.scheme or target.startswith(("mailto:", "#")):
            continue
        path_part = urllib.parse.unquote(parsed.path)
        destination = (source.parent / path_part).resolve() if path_part else source
        try:
            destination.relative_to(root)
        except ValueError:
            errors.append(f"{source.relative_to(root)}: link escapes repository: {target}")
            continue
        if not destination.exists():
            errors.append(f"{source.relative_to(root)}: missing link target: {target}")
            continue
        if parsed.fragment and destination.is_file() and destination.suffix.lower() == ".md":
            fragment = urllib.parse.unquote(parsed.fragment).lower()
            if fragment not in headings(destination):
                errors.append(
                    f"{source.relative_to(root)}: missing heading #{fragment} in "
                    f"{destination.relative_to(root)}"
                )

if errors:
    raise SystemExit("\n".join(errors))
print(f"PASS: {len(markdown_files)} Markdown files have balanced fences and valid local links")
PY

if [ "${CHECK_CADDY:-0}" = 1 ]; then
  for mode in neko desktop combined; do
    env_file="$REPO_ROOT/deploy/$mode.env.example"
    caddyfile="$REPO_ROOT/deploy/$mode.Caddyfile"
    docker run --rm --env-file "$env_file" \
      -v "$caddyfile:/etc/caddy/Caddyfile:ro" \
      caddy:2.11.4-alpine caddy validate --config /etc/caddy/Caddyfile
  done
  pass 'all three Caddyfiles parsed in the pinned Caddy image'
else
  warn 'Caddy container parsing skipped; set CHECK_CADDY=1 to enable it'
fi

pass 'repository validation completed successfully'
```

### Reference: Security policy

This policy is part of the guide; no separate policy file is required.

This repository publishes templates and documentation, not a hosted service. Operators
remain responsible for their cloud account, VM, domain, credentials, backups, updates,
and compliance obligations.


#### Reference security: Report a vulnerability

For a public GitHub repository, use its private **Security -> Report a vulnerability**
workflow. Do not open a public issue containing an exploitable detail, live hostname,
public IP, username, log bundle, credential, or private key. If private reporting has not
been configured, contact the repository owner through a private channel first.

Vulnerabilities in Neko, KasmVNC, Caddy, Docker, Ubuntu, or a cloud provider should also
be reported through that upstream project's security process.


#### Reference security: Deployment boundary

- Public by design: TCP 80/443; TCP and UDP 59000 only when Neko is enabled.
- Restricted to trusted administrator CIDRs: SSH TCP 22.
- Never public: Neko 8080, KasmVNC 8444, raw VNC 5901, and RDP 3389.
- The desktop OS account must be locked, non-sudo, and outside privileged groups.
- `.env`, `.kasmpasswd`, SSH keys, cloud CLI state, backups, and rendered inventories
  must never be committed.

Application-password-only access is supported but is not the strongest posture. For
internet-facing or sensitive deployments, place an identity-aware proxy, VPN, or
provider-native access layer in front and enable its MFA controls.


#### Reference security: Supported versions

The compatibility table in [this guide](#reference-compatibility-and-support-matrix)
records what the project was last tested against. Security fixes may require upgrading
outside that matrix; test first, retain rollback state, and then update the table.


### Reference: Optional GitHub Actions workflow

The original workflow is preserved verbatim as a readable optional example. It targeted
the former folder-based checker and is not active in the four-file documentation repo.
Materialize and adapt it only if you intentionally recreate that automation layout.

#### Reference workflow: Original validation job

Original workflow source, shown for reproducibility:

```yaml
name: Validate tutorial repository

on:
  push:
  pull_request:
  workflow_dispatch:

permissions:
  contents: read

jobs:
  validate:
    runs-on: ubuntu-24.04
    timeout-minutes: 15

    steps:
      - name: Check out the repository
        uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
        with:
          persist-credentials: false

      - name: Validate documentation and deployment assets
        env:
          CHECK_CADDY: '1'
          REQUIRE_SHELLCHECK: '1'
        run: bash scripts/check-repository.sh
```
