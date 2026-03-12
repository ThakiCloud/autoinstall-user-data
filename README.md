# Airgap Autoinstall Overview

[autoinstall-user-data](/Users/joon/github/airgap-autoinstall/autoinstall-user-data) is a single Ubuntu Server autoinstall definition for an offline bootstrap node. It combines:

- Ubuntu installation via `#cloud-config` and `autoinstall`
- offline APT bootstrap from a USB volume labeled `THAKI_DATA`
- first-boot provisioning through `cloud-init user-data`
- staged post-install setup for Docker, PXE/TFTP, Kolla-Ansible, and Harbor

The install happens first. After the OS boots, `/usr/local/sbin/init-cp.sh` runs ordered stage scripts once and records completion stamps.

## Language Versions

- English: [README.md](/Users/joon/github/airgap-autoinstall/README.md)
- Korean: [README.ko.md](./README.ko.md)

### Why the Korean version exists

`README.ko.md` provides the same operational context in Korean for engineers who are more comfortable reviewing installation flow, variable meanings, and stage behavior in Korean.

Benefits for Korean-speaking users:

- reduces ambiguity when reviewing storage, network, and bootstrap settings
- makes handoff and team discussion easier in Korean-speaking environments
- helps operators validate risky changes to `autoinstall-user-data` faster
- lowers the chance of misreading stage intent during maintenance or troubleshooting

## Execution Flow

1. `early-commands` mount the USB media at `/mnt/usbdata`.
2. The same USB content is exposed through `python3 -m http.server` on `127.0.0.1:18080`.
3. APT is redirected to the local HTTP endpoint and USB-backed package source.
4. Ubuntu installs with the configured identity, storage, locale, timezone, SSH, and network settings.
5. On first boot, `user-data.write_files` creates:
   - `/usr/local/bin/cp.env`
   - `/usr/local/lib/cp/lib.sh`
   - `/usr/local/sbin/init-cp.sh`
   - `/usr/local/lib/cp/stages/*.sh`
6. `runcmd` starts `init-cp.sh`.
7. `init-cp.sh` loads `cp.env`, validates required variables, and runs stages in lexical order.
8. Each completed stage writes `/var/lib/cp/<stage>.done`, so reruns skip finished work.

## Install-Time Configuration

### Base packages

- `apt-transport-https`
- `ca-certificates`
- `curl`
- `gnupg`
- `python3-venv`
- `git`
- `dnsmasq`
- `syslinux-common`

### APT behavior

- disables `backports`
- disables GeoIP mirror lookup
- uses `http://127.0.0.1:18080` as the primary mirror
- uses `http://127.0.0.1:18080` as the security mirror for `amd64` and `i386`
- aborts if configured mirror fallback is needed

### System identity

- hostname: `bootstrap-test1`
- username: `tkadmin`
- real name: `tkadmin`
- password: pre-hashed in the file

### OS defaults

- kernel: `linux-generic`
- keyboard layout: `us`
- locale: `en_US.UTF-8`
- timezone: `Asia/Seoul`
- OEM install mode: `auto`
- installation source: `ubuntu-server`
- driver search: disabled
- codecs install: disabled
- driver install: disabled

### SSH

- installs the SSH server
- enables password authentication
- does not preload authorized keys

### Storage

- swap disabled
- layout: `lvm`
- sizing policy: `all`
- target disk: `/dev/nvme0n1`

### Network

- matches NIC `enp0s31f6`
- renames it to `mgmt1`
- assigns `192.168.0.205/24`
- default route via `192.168.0.1`
- DNS servers: `4.2.2.1`, `8.8.8.8`

## Offline Media Requirements

The install requires a readable USB volume labeled `THAKI_DATA`.

### How the USB media is used

- during install:
  - mounted at `/mnt/usbdata`
  - used as the package source for Ubuntu repositories
  - exposed through a local HTTP server for autoinstall APT access
- during first boot:
  - remounted read-only
  - used as the source for Harbor, Docker, PXE, Kolla-Ansible, and image archives

### Expected content on the USB media

- `${THAKI_DATA}/init-pkg/harbor-offline-installer-thaki-v2.14.0.tgz`
- `${THAKI_DATA}/init-pkg/docker-compose-pack-28.5.1-1.tgz`
- `${THAKI_DATA}/init-pkg/kolla-ansible-offline-bundle.tgz`
- `${THAKI_DATA}/init-pkg/create_harbor_projects.sh`
- `${THAKI_DATA}/init-pkg/artifact-scripts.tgz`
- `${THAKI_DATA}/pxe-netboot/pxe-netboot.tgz`
- `${THAKI_DATA}/cri-images/docker-io-images.tgz`
- `${THAKI_DATA}/cri-images/quay-io-images.tgz`
- `${THAKI_DATA}/cri-images/registry-kis-io-images.tgz`
- Ubuntu repository content for:
  - `noble`
  - `noble-updates`
  - `noble-security`

## First-Boot Components

### `cp.env`

`/usr/local/bin/cp.env` is the main variable file.

| Variable | Current value | Purpose |
| --- | --- | --- |
| `THAKI_PXENIC` | `mgmt1` | Interface used by PXE/DHCP |
| `THAKI_DATA` | `/mnt/usbdata` | Mount point for the source USB media |
| `THAKI_ADMIN` | `tkadmin` | Admin user that receives generated helper scripts |
| `THAKI_SRC_DATA_LABEL` | `THAKI_DATA` | Source disk label expected on the USB media |
| `THAKI_DST_DATA_LABEL` | `TKDATA` | Declared destination label, currently unused |
| `THAKI_DST_DISK` | empty | Optional block device for `/data`; empty means skip data-disk prep |
| `HARBOR_URL` | `ar.thakicloud.net` | Harbor hostname |
| `HARBOR_PORT` | `38088` | Harbor port |
| `HARBOR_USER` | `admin` | Harbor admin account |
| `HARBOR_PWD` | `Harbor12345` | Harbor admin password |
| `HARBOR_PROXIES` | `docker.io ghcr.io quay.io registry.k8s.io` | Registry proxy targets for containerd/Harbor |
| `HARBOR_PKG` | `${THAKI_DATA}/init-pkg/harbor-offline-installer-thaki-v2.14.0.tgz` | Harbor offline installer |
| `THAKI_DOCKER_PKG` | `${THAKI_DATA}/init-pkg/docker-compose-pack-28.5.1-1.tgz` | Docker package bundle |
| `HARBOR_IP` | `192.168.0.205` | IP forced into `/etc/hosts` for Harbor |
| `HARBOR_PROJECT_SCRIPT` | `${THAKI_DATA}/init-pkg/create_harbor_projects.sh` | Harbor project helper path; currently not used directly |
| `THAKI_PXE_PKG` | `${THAKI_DATA}/pxe-netboot/pxe-netboot.tgz` | PXE/TFTP asset bundle |
| `KOLLA_PATH` | `${THAKI_DATA}/init-pkg/kolla-ansible-offline-bundle.tgz` | Offline Kolla-Ansible bundle path |

### `lib.sh`

`/usr/local/lib/cp/lib.sh` provides:

- `log`
- `warn`
- `die`
- `require_vars`
- `run_stage`

### `init-cp.sh`

`/usr/local/sbin/init-cp.sh` does the following:

- logs to `/var/log/init-cp.log`
- loads and exports all values from `cp.env`
- validates core environment variables
- enumerates `/usr/local/lib/cp/stages/*.sh`
- executes stages in lexical order
- records completion under `/var/lib/cp`

### `setup-kolla-ansible.sh`

`/usr/local/lib/cp/setup-kolla-ansible.sh` prepares a usable offline `kolla-ansible` environment for `THAKI_ADMIN`.

It does the following:

- resolves the admin home from `THAKI_ADMIN`
- validates that `KOLLA_PATH` points to a local `.tgz` bundle
- extracts the bundle under `${ADMIN_HOME}/kolla-ansible/bundle`
- creates a virtualenv under `${ADMIN_HOME}/kolla-ansible/.venv`
- installs Python packages from `wheelhouse/` with `--no-index`
- installs ansible collections from `galaxy-collections/*.tar.gz`
- validates the result with `kolla-ansible -h`
- creates `${ADMIN_HOME}/5-use-kolla-ansible.sh` as a convenience launcher

Expected bundle structure:

- `wheelhouse/`
- `galaxy-collections/`

## Stage Breakdown

### `10-usbdata-mount.sh`

Purpose:

- mount `/dev/disk/by-label/${THAKI_SRC_DATA_LABEL}` to `${THAKI_DATA}` as read-only
- persist the mount in `/etc/fstab`

Behavior:

- fails if the labeled device is missing
- reloads systemd mount state after updating `fstab`

### `11-prepare-data.sh`

Purpose:

- optionally prepare a separate `/data` disk
- optionally copy USB content to `/data/thaki-pkg` using background `rsync`

Behavior:

- runs only if `THAKI_DST_DISK` is non-empty
- refuses to format a device that appears to be the root disk
- refuses to format a disk already mounted elsewhere
- formats the disk as ext4
- mounts it at `/data`
- adds an `fstab` entry by UUID
- starts background `rsync`
- logs to `/var/log/cp-rsync-usbdata-to-data.log`
- stores the rsync PID in `/run/cp-rsync.pid`

Current effective behavior:

- skipped, because `THAKI_DST_DISK=""`

### `12-apt-usb-sources.sh`

Purpose:

- replace APT source configuration with a `file:` source pointing to `${THAKI_DATA}`

Behavior:

- writes `/etc/apt/sources.list.d/thaki-usb.sources`
- removes `/etc/apt/sources.list.d/ubuntu.sources`
- runs `apt-get update || true`

### `14-install-docker-pkgs.sh`

Purpose:

- install Docker-related packages from an offline tarball
- configure Docker to trust Harbor as an insecure registry

Behavior:

- extracts `${THAKI_DOCKER_PKG}` to a temporary directory
- installs all `.deb` files with `dpkg -i`
- adds `${THAKI_ADMIN}` to the `docker` group
- writes `/etc/docker/daemon.json` with the Harbor registry endpoint
- restarts Docker

Prerequisite:

- the package bundle must provide a working Docker service

### `16-config-pxetftp.sh`

Purpose:

- install PXE/TFTP assets
- configure `dnsmasq` for DHCP and TFTP

Behavior:

- extracts `${THAKI_PXE_PKG}` into `/`
- writes `/etc/dnsmasq.d/pxe.conf`
- enables and starts `dnsmasq`

Configured PXE values:

- interface: `${THAKI_PXENIC}` and `lo`
- DHCP range: `192.168.0.100` to `192.168.0.200`
- BIOS boot file: `pxelinux.0`
- UEFI x86_64 boot file: `bootx64.efi`
- TFTP root: `/srv/tftp`
- subnet mask option: `255.255.255.0`

### `18-setup-kolla-ansible.sh`

Purpose:

- provision an offline Kolla-Ansible environment for the admin user

Behavior:

- requires `THAKI_ADMIN` and `KOLLA_PATH`
- calls `/usr/local/lib/cp/setup-kolla-ansible.sh`
- runs after Docker setup and before Harbor preparation
- relies on the normal stage stamp mechanism for idempotency

### `20-prepare-harbor.sh`

Purpose:

- unpack Harbor into the admin user home
- force the Harbor hostname mapping
- generate helper scripts for later Harbor tasks

Behavior:

- writes `${HARBOR_IP} ar.thakicloud.net` to `/etc/hosts`
- extracts `${HARBOR_PKG}` into the admin user home
- copies `harbor.yml.thaki` to `harbor.yml`
- creates these helper scripts:
  - `0-install-harbor.sh`
  - `1-craete-harbor-projects.sh`
  - `2-config-containerd-harbor.sh`
  - `3-setup-os-mirror.sh`
  - `4-unzip-container-images.sh`

Helper script summary:

- `0-install-harbor.sh`
  - runs Harbor `install.sh`
- `1-craete-harbor-projects.sh`
  - creates Harbor projects
- `2-config-containerd-harbor.sh`
  - generates `/etc/containerd/config.toml`
  - enables `SystemdCgroup`
  - configures `config_path=/etc/containerd/certs.d`
  - creates per-registry `hosts.toml` entries for `HARBOR_PROXIES`
- `3-setup-os-mirror.sh`
  - switches Harbor compose and nginx configs to the `.thaki` variants
  - recreates the `proxy` service
- `4-unzip-container-images.sh`
  - extracts image archives
  - pushes artifacts into Harbor

Important:

- `20-prepare-harbor.sh` prepares Harbor but does not execute the generated helper scripts

## Operations

### Idempotency

- each stage writes `/var/lib/cp/<stage>.done`
- reruns skip any stage that already has a stamp file
- to rerun a stage on an installed system, remove its stamp file

### Logs

- `/var/log/init-cp.log`
- `/var/log/usb-http.log`
- `/var/log/cp-rsync-usbdata-to-data.log`

### Security posture

This file is optimized for bootstrap convenience, not hardening:

- SSH password login is enabled
- no SSH public keys are preconfigured
- USB APT sources are trusted
- Docker uses an insecure registry entry
- Harbor credentials are stored in plaintext in `cp.env`
- Harbor hostname resolution is forced through `/etc/hosts`

## Change Hotspots

These are the settings most likely to change in future edits:

- host identity:
  - hostname
  - username
  - password hash
- install target:
  - disk path
  - storage layout
- network:
  - NIC match name
  - static IP
  - gateway
  - DNS
- offline source:
  - USB label
  - package paths
  - Ubuntu suites
- PXE:
  - interface
  - DHCP range
  - TFTP assets
- Kolla-Ansible:
  - bundle path
  - install layout under the admin home
  - offline bundle contents
- Harbor:
  - hostname
  - IP
  - port
  - credentials
  - proxy registries
- optional data disk:
  - `THAKI_DST_DISK`

## Known Gaps

- `THAKI_DST_DATA_LABEL` is defined but unused.
- `KOLLA_PATH` must point to a bundle that already contains `wheelhouse/` and `galaxy-collections/`.
- `HARBOR_PROJECT_SCRIPT` is defined but not used directly by the stage logic.
- `1-craete-harbor-projects.sh` appears to have a filename typo.
- Stage numbering intentionally leaves gaps for future insertions.
- `11-prepare-data.sh` requires `rsync`, but `rsync` is not listed in `autoinstall.packages`.

## Editing Guidelines

When updating [autoinstall-user-data](/Users/joon/github/airgap-autoinstall/autoinstall-user-data):

- preserve YAML indentation strictly
- keep install-time settings separate from first-boot `user-data`
- change `cp.env` first when only values are changing
- change stage scripts only when behavior must change
- preserve stage ordering unless dependencies change
- treat storage and network edits as high-risk changes
