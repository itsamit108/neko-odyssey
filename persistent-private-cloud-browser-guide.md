# Building a Persistent Private Cloud Browser and Personal Cloud Computer

*A simple, end-to-end guide to Neko, cloud virtual machines, persistence, automatic start/stop, storage, networking, and cost control.*

> Prices, free-tier limits, VM availability, and product rules change. Use the formulas and architecture in this guide, then confirm the current values in your own cloud account before deployment.

---

## Table of contents

1. [What we are trying to build](#1-what-we-are-trying-to-build)
2. [Important terms](#2-important-terms)
3. [The main ways to solve the problem](#3-the-main-ways-to-solve-the-problem)
4. [What Neko is](#4-what-neko-is)
5. [Neko versus a full remote desktop](#5-neko-versus-a-full-remote-desktop)
6. [Cloud VM pricing explained](#6-cloud-vm-pricing-explained)
7. [Persistent, temporary, and ephemeral storage](#7-persistent-temporary-and-ephemeral-storage)
8. [Spot VMs](#8-spot-vms)
9. [Network traffic and egress](#9-network-traffic-and-egress)
10. [Three practical architectures](#10-three-practical-architectures)
11. [Recommended automatic architecture](#11-recommended-automatic-architecture)
12. [What data must be saved](#12-what-data-must-be-saved)
13. [Neko persistence configuration](#13-neko-persistence-configuration)
14. [Using Oracle Object Storage](#14-using-oracle-object-storage)
15. [The automatic Start button](#15-the-automatic-start-button)
16. [Detecting 10 minutes of inactivity](#16-detecting-10-minutes-of-inactivity)
17. [The automatic shutdown flow](#17-the-automatic-shutdown-flow)
18. [Networking, domains, HTTPS, and WebRTC](#18-networking-domains-https-and-webrtc)
19. [Cost formulas and examples](#19-cost-formulas-and-examples)
20. [Performance and video quality](#20-performance-and-video-quality)
21. [Adding a full personal cloud desktop](#21-adding-a-full-personal-cloud-desktop)
22. [Using several cloud subscriptions](#22-using-several-cloud-subscriptions)
23. [Using a home laptop, Mac, or phone instead](#23-using-a-home-laptop-mac-or-phone-instead)
24. [Security checklist](#24-security-checklist)
25. [A sensible build plan](#25-a-sensible-build-plan)
26. [Common mistakes](#26-common-mistakes)
27. [Final recommended design](#27-final-recommended-design)
28. [Official references](#28-official-references)

---

# 1. What we are trying to build

The goal is a private browser or cloud computer that feels like a laptop stored on the internet.

The desired experience is:

```text
Open one permanent website
        |
        v
Click "Start"
        |
        v
A cloud VM starts or is created
        |
        v
Neko starts with the same profile as last time
        |
        v
Use the browser from any laptop
        |
        v
Leave the Neko tab
        |
        v
After 10 minutes, data is saved and compute stops
```

The system should remember:

- Website logins and cookies
- Bookmarks and history
- Extensions and browser settings
- Open tabs that the browser can restore
- Selected downloads and personal files
- The same Neko user password
- The same public URL

The system should not require:

- Installing software on a friend's laptop
- Keeping an expensive VM running all day
- Manually opening the Azure portal each time
- Reinstalling Neko after every session
- Losing the browser profile when the VM is stopped

This is possible, but it needs several parts working together.

---

# 2. Important terms

## Virtual machine

A virtual machine, or VM, is a rented computer in a cloud data centre.

It has:

- Virtual CPU cores
- RAM
- An operating system
- One or more disks
- A network interface
- Usually a public or private IP address

Traditional VPS products and cloud VMs are broadly the same kind of product. The main difference is packaging. A traditional VPS often includes storage and bandwidth in one monthly price. A hyperscale cloud usually bills compute, disk, IP address, and bandwidth separately.

## Neko

Neko is a self-hosted browser or Linux desktop that runs inside Docker and streams its screen through WebRTC.

Your local device receives a video stream and sends keyboard and mouse input back to Neko.

## Persistence

Persistence means data survives after the current process, container, or VM stops.

Examples:

- A browser profile on a persistent disk
- A backup in object storage
- A Docker volume stored on a managed disk

RAM is not persistent. A temporary disk is not persistent. A container's writable layer should not be treated as persistent.

## Deallocated

For an ordinary Azure VM, **Stopped (deallocated)** means Azure has released the CPU and RAM allocation. Compute billing stops, although disks and some networking resources can continue to cost money.

Stopping the operating system from inside Linux is not always enough. Always confirm that the Azure state is **Stopped (deallocated)**.

## Ephemeral OS disk

An ephemeral OS disk is stored on the physical host's local storage rather than normal remote managed storage.

It is fast and can avoid a managed OS disk charge, but it is intended for stateless workloads. It does not provide the normal persistent stop/deallocate/restart model.

For an ephemeral VM, the usual workflow is:

```text
Create -> use -> save external state -> delete
```

not:

```text
Create -> deallocate -> start later with the same OS disk
```

## Spot VM

A Spot VM uses spare cloud capacity at a discount. The provider can evict it when capacity is needed.

Spot is useful when the workload can recover from interruption. It is not suitable as the only copy of important data.

## Object storage

Object storage stores files as objects in buckets. Examples include:

- Oracle Object Storage
- Amazon S3
- Cloudflare R2
- Google Cloud Storage
- Azure Blob Storage

Object storage is excellent for backups. It is not normally a direct replacement for a VM's boot disk.

## Ingress and egress

- **Ingress**: data entering the cloud provider
- **Egress**: data leaving the cloud provider

Ingress is commonly free. Egress is commonly charged after a free allowance.

---

# 3. The main ways to solve the problem

## Option A: Use an old laptop at home

Run Neko, Chrome Remote Desktop, or another remote-access tool on an old laptop.

### Advantages

- No VM compute bill
- Full control
- Local storage is naturally persistent
- Good performance if the hardware is decent
- No Spot eviction

### Disadvantages

- Electricity use
- Home internet upload speed matters
- Power cuts and router failures
- Heat and battery management
- Port forwarding, TURN, or another remote-access service may be needed
- Fully automatic recovery after a complete power failure can be difficult

This is often the cheapest long-term option.

## Option B: Use a normal cloud VM with a managed disk

The VM keeps its persistent OS disk when deallocated.

### Advantages

- Simple persistence
- Fast restart
- Easy to understand
- The whole Linux installation survives
- Good choice for a full cloud desktop

### Disadvantages

- The managed disk is charged while the VM is off
- A retained public IP can also be charged
- Spot eviction is still possible if Spot is used

This is the best first version because it is simple.

## Option C: Use a disposable VM and external object storage

Use a Spot VM with an ephemeral OS disk. Save the Neko profile to object storage before deleting the VM.

### Advantages

- Very low idle cost
- No persistent Azure OS disk charge
- Easy to recreate from code
- Good research and automation project

### Disadvantages

- More moving parts
- Longer cold start
- Must restore data on every start
- Must save data before every deletion
- Spot eviction can lose changes made since the last checkpoint
- A full desktop is harder to back up this way

This is the lowest-cost but most complex design.

## Option D: Buy a simple VPS

A normal VPS provider often bundles CPU, RAM, storage, IP, and traffic.

### Advantages

- Predictable bill
- Persistent disk included
- Simple Linux server
- No need to design a custom start controller

### Disadvantages

- Usually charged for the full month
- Less useful for only a few hours per week
- Hardware may be shared heavily
- Some providers have limited locations

A VPS is often better than hyperscale cloud when the server must run continuously.

---

# 4. What Neko is

Neko is a Docker-based, self-hosted virtual browser.

It can run:

- Firefox
- Chromium-based browsers
- A lightweight Xfce or KDE Linux desktop
- Other graphical applications

Neko streams:

- Video
- Audio
- Keyboard input
- Mouse input

through WebRTC.

A normal web browser is enough on the client device.

## What Neko is good for

- Private remote browsing
- A persistent browser available from many devices
- Shared browsing
- Browser isolation
- Watching or controlling a remote browser
- Accessing internal web applications
- A small containerised Linux desktop

## What Neko is not

- A direct copy of your local Chrome profile unless you import it
- A replacement for all features of a full Windows or macOS machine
- A zero-bandwidth remote solution
- A guarantee that every DRM video service will work
- A live-migration system that preserves RAM across different VMs

A Neko container is one remote graphical session. Several clients can connect to the same session, depending on configuration.

---

# 5. Neko versus a full remote desktop

| Requirement | Neko | Full remote desktop |
|---|---|---|
| Remote browser | Excellent | Yes |
| Access from a normal browser | Yes | Yes, with Guacamole or a web gateway |
| Isolated container | Yes | Usually no |
| Full host operating system | No | Yes |
| Existing Linux applications | Limited to container image | Yes |
| Simple browser persistence | Yes | Yes |
| Low resource use | Usually lower | Usually higher |
| Exact VM desktop | No | Yes |
| Multiple applications | Use Xfce/KDE image or custom image | Yes |

For browser-only use, Neko is simpler.

For a complete cloud computer, use:

- Ubuntu Desktop plus a browser-based remote desktop gateway
- Apache Guacamole with RDP or VNC
- KasmVNC
- A Neko Xfce/KDE image if a containerised Linux desktop is enough

The same start, backup, idle, and shutdown logic can control both Neko and a full desktop.

---

# 6. Cloud VM pricing explained

Cloud pricing pages often show a number such as:

```text
$13.22 per month
```

This does not always mean a flat monthly subscription.

For many VM pricing pages, the monthly number is an estimate based on approximately 730 running hours:

```text
hourly price x 730 hours = displayed monthly estimate
```

Therefore:

```text
hourly price = displayed monthly estimate / 730
```

If a VM is shown as $13.22 per month:

```text
$13.22 / 730 = about $0.0181 per running hour
```

If it runs for 90 hours in a month:

```text
$0.0181 x 90 = about $1.63 compute
```

## What is normally billed separately

A VM bill can include:

1. Compute
2. OS disk
3. Additional data disks
4. Disk transactions, depending on the disk type
5. Public IP
6. Outbound data transfer
7. Load balancer
8. Backup and snapshots
9. Logging and monitoring
10. Automation services

Do not compare only the VM compute price.

## Always-on versus on-demand

If a VM is running 24 hours per day, you pay for roughly the full month of compute even when nobody is connected to Neko.

If a VM is running only three hours per day:

```text
3 hours x 30 days = 90 running hours
```

Compute should be based on those 90 hours, provided the VM is really deallocated or deleted after use.

---

# 7. Persistent, temporary, and ephemeral storage

This is one of the most important topics.

## Managed OS disk

A managed OS disk contains:

- Ubuntu
- Docker
- Neko
- Configuration
- Any profile stored on the OS disk

It remains after the VM is deallocated.

### Good for

- Simple persistence
- Fast restarts
- Full desktops
- Important local files

### Cost behaviour

The disk remains allocated while the VM is off, so storage billing continues.

## Managed data disk

An additional persistent disk attached to the VM.

It is optional for a small Neko setup. The Neko profile can live on the managed OS disk.

A separate data disk is useful when:

- You want to rebuild the operating system without touching user data
- You want easier backup or migration
- The profile and downloads are large
- You run both a desktop and Neko

## Temporary local disk

Some VM sizes include a large local temporary disk.

It is useful for:

- Browser cache
- Video buffers
- `/tmp`
- Disposable downloads
- Build files
- Swap, when appropriate

Do not store the only copy of:

- Cookies
- Bookmarks
- Neko profile
- Personal documents
- Backups
- Application databases

A temporary disk can be erased after deallocation, redeployment, host movement, or eviction.

## Ephemeral OS disk

An ephemeral OS disk uses local host storage for the operating system.

It can remove the managed OS disk charge, but the operating system itself becomes disposable.

Use it only when the complete machine can be rebuilt automatically.

## Minimum disk size

For a normal managed Ubuntu Neko VM:

| Disk size | Practical result |
|---|---|
| 16 GiB | Usually too small |
| 32 GiB | Minimum practical |
| 64 GiB | Recommended for Neko and normal updates |
| 128 GiB | Comfortable for Neko plus a full desktop |
| 256 GiB or more | Useful for large downloads or many applications |

For the external-object-storage architecture, the profile backup can often stay below 5-20 GB if caches and temporary files are excluded.

---

# 8. Spot VMs

Spot VMs are cheap because the cloud provider can take the capacity back.

## Advantages

- Large discounts
- Good for personal experiments
- Good for disposable Neko machines
- Useful when state is stored elsewhere

## Risks

- Eviction can happen at any time
- The VM may not restart because capacity is unavailable
- Spot price can change
- A sudden eviction can lose recent unsaved browser changes

## Eviction policies

A normal Spot VM can commonly use:

- **Deallocate**: keep the VM resource and persistent disks, but release compute
- **Delete**: remove the VM after eviction

For an ephemeral OS disk, design around deletion and recreation.

## Safe Spot design

Use all three:

1. Save a checkpoint periodically
2. Save again during a graceful shutdown
3. Monitor the cloud scheduled-events endpoint for eviction warnings

Even with this, accept that the latest few minutes can sometimes be lost.

## When not to use Spot

Avoid Spot when:

- The service must always be available
- You cannot tolerate a restart
- You are in the middle of important unsaved work
- The system contains the only copy of important data
- You need guaranteed capacity

---

# 9. Network traffic and egress

Neko is a remote video stream. Network traffic can cost more than the VM.

## YouTube example

When YouTube runs inside Neko:

```text
YouTube -> cloud VM
```

is ingress to the cloud VM.

Then Neko captures the remote browser and sends:

```text
cloud VM -> your laptop
```

That second stream is egress.

You can therefore pay egress even when the original YouTube stream entering the VM is free.

## Bitrate formula

A useful estimate is:

```text
data in GB = bitrate in Mbps x connected hours x 0.45
```

Examples for 90 connected hours:

| Average Neko bitrate | Approximate outgoing data |
|---:|---:|
| 2 Mbps | 81 GB |
| 3 Mbps | 121.5 GB |
| 4 Mbps | 162 GB |
| 6 Mbps | 243 GB |
| 8 Mbps | 324 GB |

The percentage of time spent on YouTube does not always reduce the traffic proportionally. Neko can continue streaming the desktop while the connection is open.

## How to reduce egress

- Use 720p instead of 1080p
- Use 30 FPS instead of 60 FPS
- Reduce the encoder bitrate
- Disconnect Neko when not using it
- Stop the VM after inactivity
- Avoid leaving a session open in a hidden tab
- Use a region near the user for better latency

---

# 10. Three practical architectures

## Architecture 1: Always-on VM with managed storage

```text
Permanent VM
  |
  +-- Managed OS disk
  +-- Neko
  +-- Public IP
```

### User experience

- Neko is immediately available
- No cold start
- Browser state remains naturally

### Cost

- Full monthly compute
- Full monthly disk
- Full monthly public IP
- Egress when used

### Best for

- Daily heavy use
- Fast access
- Simple setup

## Architecture 2: Managed disk with automatic deallocation

```text
Permanent VM resource
  |
  +-- Persistent managed OS disk
  +-- VM starts on demand
  +-- VM deallocates after 10 minutes idle
```

### User experience

- Start normally takes a few minutes
- All installed software remains
- Browser profile remains
- No restore from object storage is required

### Cost

- Compute only while allocated
- Managed disk all month
- Public IP may cost all month if retained
- Egress when used

### Best for

- Most people
- Full desktop plus Neko
- Reliability and simplicity

## Architecture 3: Disposable Spot VM with external persistence

```text
Control website
  |
  +-- Creates Spot VM with ephemeral OS
  +-- Restores Neko profile from object storage
  +-- Starts Neko
  +-- Saves profile after inactivity
  +-- Deletes VM
```

### User experience

- Longer cold start
- Same browser profile after restore
- The VM itself is disposable

### Cost

- Compute only while VM exists
- No Azure managed OS disk
- External object storage may be free within limits
- Public IP and egress can still cost money

### Best for

- Minimum idle cost
- Learning automation
- A browser-only workload
- Users who accept more complexity

---

# 11. Recommended automatic architecture

For a low-cost, highly automatic Neko system:

```text
control.example.com
        |
        v
Always-available controller
        |
        +-- Authenticates the user
        +-- Creates or starts the Azure VM
        +-- Shows progress
        +-- Checks health
        +-- Returns the Neko URL
        |
        v
Azure Spot VM
        |
        +-- Ubuntu ARM64 or AMD64
        +-- Docker
        +-- Neko Firefox
        +-- Idle monitor
        +-- Backup client
        |
        v
Oracle Object Storage
        |
        +-- Encrypted Neko profile snapshots
        +-- Downloads selected for persistence
        +-- Configuration
```

## Suggested components

### Compute

- Azure Spot VM
- Four vCPUs and 8 GB RAM as a starting point
- Use a supported Neko architecture
- An ARM VM can use Neko's ARM64 images
- Prefer Firefox for the first build

### Persistent storage

- Oracle Object Storage
- Encrypted backup repository
- Keep the profile small by excluding caches

### Controller

Any small always-available service can be used:

- Azure Functions
- GCP Cloud Run or Cloud Run functions
- Cloudflare Workers with a suitable backend
- A small always-on home server
- A tiny VPS

The controller should not run Neko. It only controls the VM lifecycle.

### Domain names

Use two names:

```text
control.example.com
neko.example.com
```

`control.example.com` is always online.

`neko.example.com` points to the currently running VM or an always-on gateway.

---

# 12. What data must be saved

Do not back up the whole disposable VM unless necessary.

Separate data into two groups.

## Re-creatable data

Keep this in Git or infrastructure-as-code:

- Docker Compose file
- Neko image version
- Firewall configuration
- Reverse-proxy configuration
- Startup scripts
- Cloud-init
- Terraform, Bicep, or ARM templates
- Package installation list
- Idle-monitor code

This data should be easy to recreate.

## Mutable persistent data

Back up:

- Browser profile
- Cookies
- History
- Bookmarks
- Browser extensions
- Browser settings
- Selected downloads
- Neko configuration that changes
- User-created files
- Optional TLS state

For Firefox Neko, the main profile path is:

```text
/home/neko/.mozilla/firefox/profile.default
```

A clean host layout can be:

```text
/srv/neko/profile
/srv/neko/downloads
/srv/neko/config
```

## Data to exclude

Exclude:

- Browser cache
- Shader cache
- Temporary video files
- Crash reports
- `/tmp`
- Docker image layers
- Package caches
- Large disposable downloads
- Logs that are not needed

This keeps backups fast and small.

---

# 13. Neko persistence configuration

A persistent browser profile needs two things:

1. A mounted profile directory
2. A browser policy that does not clear the data on shutdown

## Example directory structure

```text
/opt/neko/
├── docker-compose.yml
├── policy.json
└── data/
    ├── profile/
    └── downloads/
```

## Example Docker Compose file

```yaml
services:
  neko:
    image: ghcr.io/m1k1o/neko/firefox:latest
    container_name: neko
    restart: unless-stopped
    shm_size: "2gb"

    ports:
      - "8080:8080"
      - "56000-56100:56000-56100/udp"

    volumes:
      - ./data/profile:/home/neko/.mozilla/firefox/profile.default
      - ./data/downloads:/home/neko/Downloads
      - ./policy.json:/usr/lib/firefox/distribution/policies.json:ro

    environment:
      NEKO_DESKTOP_SCREEN: "1280x720@30"
      NEKO_MEMBER_MULTIUSER_USER_PASSWORD: "${NEKO_USER_PASSWORD}"
      NEKO_MEMBER_MULTIUSER_ADMIN_PASSWORD: "${NEKO_ADMIN_PASSWORD}"
      NEKO_WEBRTC_EPR: "56000-56100"
      NEKO_WEBRTC_ICELITE: "1"
```

For a public deployment, configure the public WebRTC address, firewall, NAT, or TURN service according to the actual network design.

## File ownership

Neko normally uses UID 1000 inside the container.

```bash
sudo mkdir -p /opt/neko/data/profile /opt/neko/data/downloads
sudo chown -R 1000:1000 /opt/neko/data
```

## Firefox policy

```json
{
  "policies": {
    "SanitizeOnShutdown": false,
    "Homepage": {
      "StartPage": "previous-session"
    }
  }
}
```

Without the policy change, mounting a profile alone may not keep cookies and sessions.

## What persistence can restore

Usually:

- Cookies
- Website sessions
- Bookmarks
- History
- Extensions
- Settings
- Restorable tabs

It cannot restore the exact RAM state of the old process.

---

# 14. Using Oracle Object Storage

Oracle Object Storage can act as the external persistent backup store.

Oracle offers an S3-compatible API, so common backup tools can use it.

## Recommended tool: restic

Restic provides:

- Encryption
- Incremental backups
- Deduplication
- Snapshots
- Retention policies
- S3-compatible storage support

## Basic Oracle setup

1. Create an Object Storage bucket.
2. Create a dedicated OCI user for backups.
3. Give that user access only to the required bucket.
4. Create a Customer Secret Key.
5. Save the access key and secret securely.
6. Use the Oracle S3-compatible endpoint for the home region.

Do not put the secret key in:

- Public Git repositories
- Browser JavaScript
- Docker images
- The control website source code

Store it in:

- A secret manager
- Encrypted controller configuration
- A protected VM environment file
- A temporary deployment secret

## Example restic environment

```bash
export AWS_ACCESS_KEY_ID="OCI_ACCESS_KEY"
export AWS_SECRET_ACCESS_KEY="OCI_SECRET_KEY"
export RESTIC_PASSWORD_FILE="/run/secrets/restic-password"
export RESTIC_REPOSITORY="s3:https://NAMESPACE.compat.objectstorage.REGION.oraclecloud.com/BUCKET/neko"
```

Replace the uppercase placeholders with the actual OCI values.

## First-time repository setup

```bash
restic init
```

## Restore before Neko starts

```bash
mkdir -p /opt/neko/data

if restic snapshots >/dev/null 2>&1; then
  restic restore latest --target /
fi

chown -R 1000:1000 /opt/neko/data
```

Use a consistent absolute path in the backup and restore design.

## Save a snapshot

```bash
restic backup /opt/neko/data \
  --exclude '/opt/neko/data/profile/cache2' \
  --exclude '/opt/neko/data/profile/**/cache2' \
  --exclude '/opt/neko/data/profile/**/shader-cache'
```

## Retention

```bash
restic forget \
  --keep-last 7 \
  --keep-daily 7 \
  --keep-weekly 4 \
  --prune
```

Do not run a full repository check after every short session. Run lightweight verification after each backup and a deeper check periodically.

## Checkpoint strategy

Use:

- A final backup during graceful shutdown
- A checkpoint every 30-60 minutes
- An emergency best-effort backup after a Spot eviction warning

---

# 15. The automatic Start button

The control website is the power button.

## The page can show

```text
Private Browser

Status: Offline

[ Start Neko ]
```

After clicking:

```text
Creating VM...
Booting Ubuntu...
Restoring profile...
Starting Neko...
Checking connection...
Ready
```

Then it redirects to:

```text
https://neko.example.com
```

## Controller responsibilities

The controller should:

1. Authenticate the user
2. Prevent two start operations at the same time
3. Check whether a VM already exists
4. Create or start the VM
5. Wait for the cloud deployment to complete
6. Wait for the Neko health endpoint
7. Update DNS or gateway routing
8. Return the final URL
9. Show errors clearly
10. Record start time for cost tracking

## Do not expose cloud credentials to the browser

The browser should call:

```text
POST /api/start
```

The server-side controller then calls the Azure API.

Never send an Azure client secret or subscription credential to client-side JavaScript.

## Identity

The controller needs permission to perform only required actions:

- Create or start the target VM
- Delete or deallocate it
- Read its status
- Read its IP
- Update the relevant DNS record, if needed

Use least-privilege cloud roles.

## Faster start

A fresh VM can take several minutes if it must:

- Install Docker
- Download packages
- Pull Neko
- Install restic
- Restore the profile

To reduce cold-start time:

- Build a reusable image with Docker and restic already installed
- Pin the Neko image version
- Use cloud-init only for secrets and final configuration
- Keep the profile small
- Use a nearby Oracle region when possible
- Avoid restoring caches

---

# 16. Detecting 10 minutes of inactivity

This requirement needs careful design.

A connected Neko WebRTC session is not necessarily active. The user may switch to another browser tab while the connection remains open.

CPU usage is also unreliable. A browser can use CPU in the background, and a user can read a page with almost no CPU activity.

## Better definition of active

Count the session as active when the Neko tab is:

- Visible
- Focused
- Sending a regular heartbeat

Count it as inactive when:

- The user switches tabs
- The browser is minimised
- The user switches to another application
- The page is closed
- The network connection disappears

## Frontend heartbeat

A customised Neko frontend can send a heartbeat:

```javascript
const HEARTBEAT_MS = 15000;

setInterval(async () => {
  const active =
    document.visibilityState === "visible" &&
    document.hasFocus();

  if (!active) return;

  try {
    await fetch("/activity/heartbeat", {
      method: "POST",
      credentials: "include",
      keepalive: true
    });
  } catch {
    // The server-side timeout will handle missing heartbeats.
  }
}, HEARTBEAT_MS);
```

Neko's UI can be customised by building the frontend and mounting it into the container.

## Server-side idle rule

The backend records:

```text
last_visible_heartbeat
```

Every minute:

```text
if current_time - last_visible_heartbeat >= 10 minutes:
    begin graceful shutdown
```

## Protect against accidental shutdown

Add:

- A visible countdown during the final minute
- A `Keep Running` button
- A short grace period when reconnecting
- A maximum session limit as a separate safety rule
- A lock so two shutdown jobs cannot run together

## Watching a video

A visible, focused tab continues sending heartbeats even when the user does not move the mouse. This avoids stopping the VM while watching a video.

---

# 17. The automatic shutdown flow

A correct shutdown should save data before compute stops.

## Managed-disk VM

For a normal managed-disk VM:

```text
1. Block new sessions
2. Warn the user
3. Stop Neko cleanly
4. Flush writes
5. Optional backup
6. Ask Azure to deallocate the VM
7. Confirm Stopped (deallocated)
```

The managed disk keeps the profile.

## Ephemeral VM

For a disposable ephemeral VM:

```text
1. Block new sessions
2. Warn the user
3. Stop Neko cleanly
4. Flush writes
5. Back up the profile to Oracle
6. Verify the new snapshot
7. Notify the controller
8. Delete the VM and related temporary resources
```

## Example shutdown script

```bash
#!/usr/bin/env bash
set -euo pipefail

LOCK_FILE="/run/neko-shutdown.lock"
NEKO_DIR="/opt/neko"

exec 9>"$LOCK_FILE"
flock -n 9 || exit 0

cd "$NEKO_DIR"

docker compose stop neko
sync

restic backup "$NEKO_DIR/data" \
  --exclude "$NEKO_DIR/data/profile/cache2" \
  --exclude "$NEKO_DIR/data/profile/**/cache2"

restic snapshots --latest 1

curl -fsS -X POST \
  -H "Authorization: Bearer $CONTROLLER_TOKEN" \
  "https://control.example.com/api/shutdown-ready"
```

The controller should delete or deallocate the VM only after receiving the verified completion signal.

## Failure handling

If the backup fails:

- Do not immediately delete the VM
- Retry with exponential delay
- Show an error in the control page
- Keep the VM for a limited emergency period
- Alert the owner

A cost-saving system must not silently destroy the only current copy of the profile.

---

# 18. Networking, domains, HTTPS, and WebRTC

Neko needs more than a normal website connection.

## Two traffic paths

1. HTTP/HTTPS and WebSocket for the Neko interface
2. WebRTC media and input traffic

A normal reverse proxy can handle the web interface, but WebRTC still needs a valid network path.

## Basic public ports

A common Neko setup uses:

- TCP 443 for HTTPS
- UDP range for WebRTC
- Optional TCP WebRTC fallback
- SSH only from trusted addresses, if needed

Do not expose Neko's plain HTTP port directly to the internet without protection.

## Reverse proxy

Use:

- Caddy
- Nginx
- Traefik
- Another HTTPS reverse proxy

The proxy forwards:

```text
https://neko.example.com -> http://127.0.0.1:8080
```

## WebRTC public address

Neko must advertise an address that the client can reach.

Depending on the network, use:

- The VM's public IP
- NAT 1-to-1 configuration
- A correctly configured UDP/TCP mux
- A TURN server

## TURN

TURN relays the WebRTC media when a direct connection cannot be established.

TURN improves connectivity but can be expensive because all Neko video traffic passes through it.

Use TURN when:

- UDP is blocked
- The VM is behind an unsuitable NAT
- You need one stable media endpoint
- You switch between several backend clouds

## Stable URL with a changing VM

Options:

### Keep one static public IP

- Simplest
- Fast DNS behaviour
- The IP can cost money while unused

### Create a new IP and update DNS

- Lower retained-resource cost
- DNS change delay
- WebRTC configuration must use the new address
- Use a short DNS TTL

### Use an always-on gateway

- One stable public URL
- Routes requests to the active VM
- Can display a startup page
- WebRTC may still require direct ports or TURN

## HTTPS certificates

For frequently recreated machines:

- Persist the certificate state
- Use DNS-based certificate validation
- Use a stable gateway
- Avoid repeatedly requesting new certificates on every short session

---

# 19. Cost formulas and examples

Use formulas rather than relying on one old screenshot.

## Compute

```text
monthly compute =
displayed full-month price x actual running hours / 730
```

Example:

```text
displayed Spot price: $13.22/month
actual runtime: 90 hours

$13.22 x 90 / 730 = about $1.63
```

## Always-on compute

If the VM runs continuously:

```text
monthly compute is approximately the displayed monthly amount
```

It does not matter that Neko is used for only three hours per day. The VM is still allocated for the other 21 hours.

## Managed disk

A managed disk is normally billed while it exists.

The cost depends on:

- Disk type
- Provisioned size
- Transaction model
- Region
- Redundancy

A 32 GiB disk is a practical minimum. A 64 GiB disk is easier to live with.

## Temporary disk

Included local temporary storage normally has no separate storage charge, but it is not persistent.

## Public IP

A retained public IP can be billed independently of VM runtime.

Check whether the IP remains allocated after deallocation or VM deletion.

## Egress

```text
monthly stream data in GB =
average Mbps x connected hours x 0.45
```

Then:

```text
chargeable GB =
max(0, total egress - provider free allowance)
```

and:

```text
egress cost =
chargeable GB x regional egress price
```

## Three-hours-per-day example

```text
3 hours/day x 30 days = 90 hours
```

At 4 Mbps:

```text
4 x 90 x 0.45 = 162 GB
```

At 6 Mbps:

```text
6 x 90 x 0.45 = 243 GB
```

## Why YouTube percentage is not enough

If YouTube is used for only half the session, Neko may still stream the desktop for the complete connected period.

Measure actual network output rather than estimating only from video-watch time.

## Cost-control rules

- Create budget alerts
- Track VM running time independently
- Delete unused disks and IPs
- Confirm the VM state
- Set a maximum session duration
- Automatically stop after inactivity
- Keep logs small
- Review cost analysis every few days during testing

---

# 20. Performance and video quality

Neko must render a browser and encode the screen in real time.

## CPU-only VM

A four-vCPU, 8 GB VM is a reasonable starting point for:

- Normal browsing
- 720p at 30 FPS
- One user
- Moderate video use

1080p at 60 FPS is much harder, especially with software encoding.

Possible results:

- High CPU use
- Dropped frames
- Poor input latency
- Audio/video stutter
- Reduced browser performance

## Recommended starting settings

```text
Resolution: 1280x720
Frame rate: 30 FPS
Bitrate: 2-4 Mbps
Users: 1
Browser: Firefox
```

Test and increase slowly.

## For better 1080p

Consider:

- More CPU cores
- A faster CPU architecture
- Hardware video encoding
- A supported Intel or Nvidia GPU Neko image
- 1080p at 30 FPS instead of 60 FPS
- A lower bitrate
- A region closer to the user

## ARM compatibility

An ARM Azure VM requires ARM64-compatible images.

Neko publishes ARM64 images for supported applications through the GitHub Container Registry.

Always verify that:

- The chosen Neko browser image supports ARM64
- Any additional binary tools support ARM64
- Docker pulls the correct architecture
- Browser extensions work normally

---

# 21. Adding a full personal cloud desktop

There are two meanings of "full desktop."

## Containerised desktop

Use a Neko Xfce or KDE image.

### Advantages

- Same browser-based access
- Same WebRTC model
- Easy to start with Docker
- Lower complexity

### Limitations

- It is a container, not the whole VM
- Some system-level operations are limited
- Persistence must be mounted carefully

## Actual VM desktop

Install a Linux desktop on the VM and access it through:

- Apache Guacamole
- RDP with xrdp
- VNC behind Guacamole
- KasmVNC

Use two hostnames:

```text
desktop.example.com
neko.example.com
```

The same controller can start the VM for either service.

## Persistence for a full desktop

Backing up only a browser profile is small and fast.

Backing up an entire Linux home directory can be much larger.

For a full personal desktop, a managed OS or data disk is usually simpler than recreating the VM every session.

A balanced design is:

```text
Managed disk for full desktop
Object storage for independent backups
Automatic deallocation after inactivity
```

---

# 22. Using several cloud subscriptions

Several subscriptions cannot normally share one literal VM resource.

The correct model is one **logical machine** built from several equivalent backends.

```text
Permanent controller and domain
          |
          +-- Subscription A VM
          +-- Subscription B VM
          +-- Subscription C VM
          |
          +-- Shared external persistent data
```

## What stays common

- Infrastructure template
- Neko configuration
- Browser profile backup
- Passwords
- SSH public key
- Domain name
- Controller
- User experience

## What changes

- VM resource ID
- Subscription
- Public IP
- Physical host
- Running process
- WebRTC session

## Automatic switch

A controller can:

1. Detect that a budget threshold is near
2. Wait until no user is active
3. Save the latest profile
4. Create the equivalent VM in the next subscription
5. Restore the profile
6. Run health checks
7. Update routing
8. Delete or deallocate the old VM

This can be invisible while the system is offline.

It cannot migrate:

- Live RAM
- Current WebRTC connection
- Unsaved form input
- Running processes
- A playing video's exact state

It is the same logical environment, not the same physical or cloud VM.

---

# 23. Using a home laptop, Mac, or phone instead

## Old Windows or Linux laptop

A good home server for:

- Neko
- Full remote desktop
- Docker
- File storage
- Personal services

Recommended:

- SSD
- 8 GB RAM or more
- Ethernet
- Never sleep while plugged in
- Screen off
- Good cooling
- Automatic service startup
- Strong remote authentication

## Apple Silicon MacBook

An M1 Mac can run ARM64 Neko through Docker.

However:

- A MacBook normally sleeps when the lid closes
- Closed-lid operation needs careful power management
- Docker Desktop starts after user login
- FileVault can prevent unattended recovery after a reboot
- The fanless MacBook Air can become warm under continuous encoding

For full Mac access, a remote-desktop product is simpler than Neko.

## Android phone

An Android phone can run small services through tools such as Termux, but it is not a good Neko host.

Problems include:

- Android background-process limits
- Heat
- Battery ageing
- Docker and kernel limitations
- Unreliable unattended recovery
- Mobile-browser interface
- Poor storage management

Use a phone as a Neko client, not the main server.

## iPhone

An iPhone is not a practical unattended server for this use case.

---

# 24. Security checklist

A private cloud browser contains valuable login sessions.

## Authentication

- Protect the control website with OAuth or another strong login
- Enable two-factor authentication
- Use separate Neko user and admin passwords
- Use long random passwords
- Do not reuse personal account passwords

## Secrets

- Store cloud API credentials server-side
- Use a secret manager when possible
- Never commit secrets to Git
- Never expose secrets in browser JavaScript
- Rotate credentials
- Give each identity only required permissions

## Network

- Use HTTPS
- Allow only necessary ports
- Do not expose Docker's remote API
- Restrict SSH
- Use firewall rules
- Keep the operating system and containers updated
- Use a TURN server only when required

## Data

- Encrypt object-storage backups
- Keep several snapshots
- Test restoration
- Exclude caches
- Do not store the only copy of important files in temporary storage
- Avoid banking or highly sensitive work from an untrusted friend's laptop

Incognito or Guest mode prevents normal local browser data from being retained, but it does not protect against malware or keyloggers already installed on that laptop.

## Logging

Do not log:

- Passwords
- Access tokens
- Cookie databases
- Full secret environment variables

---

# 25. A sensible build plan

Do not begin with the most complicated multi-cloud automatic version.

## Phase 1: Local Neko test

1. Install Docker on a local Linux machine or VM
2. Start Neko
3. Confirm browser access
4. Confirm audio, keyboard, and mouse
5. Test 720p at 30 FPS

## Phase 2: Add local persistence

1. Mount the Firefox profile directory
2. Add the persistent browser policy
3. Log in to a test website
4. Restart the container
5. Confirm the login and bookmarks remain

## Phase 3: Deploy a normal Azure VM

1. Use a managed OS disk
2. Install Docker
3. Run Neko
4. Configure HTTPS
5. Configure WebRTC ports
6. Test from mobile data

## Phase 4: Add automatic deallocation

1. Add activity heartbeat
2. Add the 10-minute timer
3. Stop Neko cleanly
4. Deallocate the VM
5. Start it manually and confirm persistence

This version is already useful.

## Phase 5: Add the control website

1. Add authentication
2. Add a Start button
3. Call the Azure API server-side
4. Show progress
5. Redirect when Neko is ready

## Phase 6: Add Oracle backups

1. Create the OCI bucket
2. Configure a bucket-limited identity
3. Initialise restic
4. Back up the Neko profile
5. Restore it onto a clean test VM
6. Confirm logins and bookmarks

## Phase 7: Move to ephemeral OS

1. Build a disposable VM template
2. Restore automatically during boot
3. Save automatically during shutdown
4. Delete the VM only after verification
5. Test a failed backup
6. Test a Spot eviction simulation

## Phase 8: Add advanced features

- Periodic checkpoints
- Stable gateway
- TURN
- Multi-subscription switching
- Full cloud desktop
- Cost dashboard
- Automatic fallback VM

---

# 26. Common mistakes

## Mistake: treating temporary disk as persistent

Temporary storage can disappear. Use it only for cache and disposable data.

## Mistake: stopping Linux but not deallocating the VM

Compute can remain billable. Confirm the cloud power state.

## Mistake: mounting a profile but keeping Neko's clearing policy

The mounted directory alone may not keep cookies. Change the browser policy.

## Mistake: using only connection count for idle detection

A hidden tab can remain connected. Use visible-tab heartbeats.

## Mistake: deleting the VM before verifying backup

A failed upload can destroy the only current copy of the profile.

## Mistake: putting API keys in the Start page

Cloud credentials must stay on the server side.

## Mistake: assuming a reverse proxy handles all Neko traffic

The web interface may load while WebRTC video fails. Configure UDP/TCP or TURN.

## Mistake: using 1080p60 immediately

Start with 720p30 and measure CPU, latency, and bandwidth.

## Mistake: ignoring egress

Neko is video streaming. Egress can exceed compute cost.

## Mistake: relying on one Spot VM

Spot capacity can disappear. Keep external state and a rebuild path.

## Mistake: backing up the entire browser cache

It wastes time, storage, and API requests.

---

# 27. Final recommended design

## Best simple version

```text
Azure Spot VM with managed 64 GiB OS disk
Neko Firefox
Persistent profile on OS disk
Automatic 10-minute idle deallocation
Permanent authenticated control website
HTTPS and correct WebRTC networking
Oracle Object Storage backup once per day or session
```

Choose this first if reliability matters more than saving a few dollars.

## Lowest-idle-cost experimental version

```text
Authenticated control.example.com
        |
        v
Serverless controller
        |
        v
Azure Spot VM with ephemeral OS disk
        |
        +-- Cloud-init or prebuilt image
        +-- Docker and Neko
        +-- Restore encrypted profile from OCI
        +-- Visible-tab heartbeat
        +-- 10-minute idle monitor
        +-- Periodic checkpoint
        +-- Graceful final backup
        |
        v
Delete VM after verified backup
```

## Expected behaviour

Starting:

```text
Open control URL
-> click Start
-> wait for VM creation
-> restore profile
-> open Neko
```

Stopping:

```text
Leave Neko tab
-> 10 minutes pass
-> stop browser cleanly
-> save profile
-> verify snapshot
-> delete/deallocate VM
-> compute billing stops
```

Returning:

```text
Click Start again
-> restore the latest profile
-> same password
-> same bookmarks, cookies, settings, and restorable tabs
```

This is similar to shutting down a laptop and turning it on later.

It is not the same as suspending RAM. Active video playback, unsaved forms, and running processes do not survive.

---

# 28. Official references

## Neko

- [Neko introduction](https://neko.m1k1o.net/docs/v3/introduction)
- [Neko installation](https://neko.m1k1o.net/docs/v3/installation)
- [Neko Docker images and supported architectures](https://neko.m1k1o.net/docs/v3/installation/docker-images)
- [Neko browser profile persistence and policies](https://neko.m1k1o.net/docs/v3/customization/browsers)
- [Neko UI customisation](https://neko.m1k1o.net/docs/v3/customization/ui)
- [Neko installation examples](https://neko.m1k1o.net/docs/v3/installation/examples)

## Microsoft Azure

- [Azure VM states and billing](https://learn.microsoft.com/en-us/azure/virtual-machines/states-billing)
- [Azure Spot Virtual Machines](https://learn.microsoft.com/en-us/azure/virtual-machines/spot-vms)
- [Spot eviction design guidance](https://learn.microsoft.com/en-us/azure/architecture/guide/spot/spot-eviction)
- [Azure ephemeral OS disks](https://learn.microsoft.com/en-us/azure/virtual-machines/ephemeral-os-disks)
- [Ephemeral OS disk FAQ](https://learn.microsoft.com/en-us/azure/virtual-machines/ephemeral-os-disks-faq)
- [Azure managed disk billing](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-understand-billing)
- [Azure bandwidth pricing](https://azure.microsoft.com/en-us/pricing/details/bandwidth/)
- [Azure public IP pricing](https://azure.microsoft.com/en-us/pricing/details/ip-addresses/)
- [Azure pricing calculator guide](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/pricing-calculator)
- [Azure Dpldsv5 ARM VM series](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/general-purpose/dpldsv5-series)

## Oracle Cloud

- [OCI Always Free resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)
- [OCI Object Storage overview](https://docs.oracle.com/en-us/iaas/Content/Object/Concepts/objectstorageoverview.htm)
- [OCI S3-compatible API](https://docs.oracle.com/en-us/iaas/Content/Object/Tasks/s3compatibleapi.htm)
- [OCI Customer Secret Keys](https://docs.oracle.com/en-us/iaas/Content/Identity/access/working-with-customer-secret-keys.htm)

## Backup tools

- [Restic documentation](https://restic.readthedocs.io/en/latest/)
- [Restic S3-compatible repository setup](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html)
- [Rclone Oracle Object Storage support](https://rclone.org/oracleobjectstorage/)

## Google Cloud control-plane options

- [Google Cloud Free Tier](https://docs.cloud.google.com/free/docs/free-cloud-features)
- [Cloud Run container runtime contract](https://docs.cloud.google.com/run/docs/container-contract)

---

## One-sentence summary

Use the cloud VM as disposable CPU and RAM, keep the browser profile on persistent storage, put a secure controller in front of it, and never stop or delete the machine until the latest profile backup has been verified.
