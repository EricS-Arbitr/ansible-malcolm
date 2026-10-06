# ansible-malcolm

Ansible roles and playbooks that install and configure
[Malcolm](https://github.com/cisagov/Malcolm), CISA's open-source network
traffic analysis suite, on a fresh Ubuntu host without interactive prompts.

Malcolm's own installer is interactive. These roles set its configuration files
directly, use only the non-interactive modes of Malcolm's control script, and
run the stack as a systemd service. The goal is a deployment you can rebuild
from scratch with one command. It is aimed at cyber ranges and training labs,
including air-gapped ones.

## What you get

After `deploy_malcolm.yml` runs, the target has:

- **A dedicated data disk** partitioned, formatted (ext4) and mounted at
  `/data`, with Malcolm installed under it so PCAP and OpenSearch data stay off
  the OS disk.
- **Host tuning** from Malcolm's host configuration docs: `vm.max_map_count`,
  swappiness, inotify limits, file/memlock/nproc ulimits, sleep targets masked,
  GNOME idle suspend off, and unattended upgrades off.
- **Docker CE and the Compose v2 plugin** from Docker's official apt repo, with
  log rotation configured.
- **Malcolm v26.08.0**, pinned and checksum-verified, running as an unprivileged
  `malcolm` service user.
- **Live capture** on a dedicated NIC that has no IP address, promiscuous mode on
  and offloads off. Zeek and Suricata analyse traffic live, and netsniff-ng
  writes rotated PCAP for Arkime.
- **Basic authentication** with an admin account whose password comes from
  Ansible Vault. TLS certificates and internal credentials are generated on the
  first run.
- **`malcolm.service`**, which starts Malcolm at boot and stops it cleanly at
  shutdown.
- **Memory-aware sizing**: the OpenSearch and Logstash heaps, Zeek worker count
  and the Strelka pipeline are set from the host's RAM.

Every role is idempotent: a second run reports `changed=0`.

## Tested on

| Component | Version |
|---|---|
| Target | Ubuntu 22.04 Desktop (jammy), VMware Workstation VM, 16 GB RAM, 8 vCPU, 250 GB data disk |
| Controller | Ubuntu 24.04 (WSL2), ansible-core 2.21, Python 3.12 |
| Malcolm | 26.08.0 |

The roles also declare Ubuntu 24.04 (noble) support. `docker_engine` derives
its apt suite from the target's facts and works on both releases.
`malcolm_host_prep` assumes **NetworkManager** for the capture interface, which
is the default on Ubuntu Desktop. Ubuntu Server uses systemd-networkd and would
need a netplan variant (not yet written).

## Requirements

### Controller

- ansible-core (tested with 2.21)
- The collections in `requirements.yml`, pinned:
  - `community.docker` 5.3.0
  - `community.general` 13.4.0
  - `ansible.posix` 2.2.2
- An SSH keypair at `~/.ssh/id_ed25519` (used by `bootstrap.yml`)
- A vault password file at `~/.ansible/vault_pass.txt` (set in `ansible.cfg`)

### Target

- Ubuntu 22.04 or 24.04, x86_64
- **16 GB RAM minimum.** Malcolm is memory-bound; see [Sizing](#sizing).
- A **second disk** for Malcolm data (recommended; 250 GB or more)
- About **30 GB free on the OS disk** for container images. Docker keeps them
  in `/var/lib/docker`, which stays on the OS disk; only Malcolm's data goes to
  the data disk.
- A **third NIC** for capture, if you want live capture. It must not be the
  interface Ansible connects through or the one with the default route.
- `openssh-server` installed. Ubuntu Desktop doesn't ship it:
  `sudo apt install openssh-server`. This is the only manual step on the target.
- Internet access to Docker's apt repo, GitHub releases and `ghcr.io`, unless
  you point those at local mirrors (see [Air-gapped deployments](#air-gapped-deployments)).

## Repository layout

```
.
├── ansible.cfg                  # inventory, vault password file, timeouts
├── requirements.yml             # pinned collections
├── inventories/
│   └── lab/                     # example inventory (the reference lab)
│       ├── hosts.yml
│       └── group_vars/
│           ├── all.yml
│           └── malcolm/
│               ├── vars.yml     # disk, NIC, install dir, admin user
│               └── vault.yml    # encrypted secrets
├── playbooks/
│   ├── bootstrap.yml            # one-time: automation user, SSH keys, sudo
│   └── deploy_malcolm.yml       # host prep, Docker, Malcolm
└── roles/
    ├── malcolm_host_prep/
    ├── docker_engine/
    └── malcolm/
```

## Quick start

These steps assume you're creating your own inventory rather than using the
reference lab's. The `lab` inventory and its `vault.yml` belong to the original
environment; you can't decrypt that vault, so create your own.

### 1. Install the collections

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Create an inventory

Copy the example and edit it:

```bash
cp -r inventories/lab inventories/myrange
rm inventories/myrange/group_vars/malcolm/vault.yml
```

`inventories/myrange/hosts.yml`:

```yaml
all:
  children:
    malcolm:
      hosts:
        malcolm01:
          ansible_host: 10.0.0.50
```

`inventories/myrange/group_vars/malcolm/vars.yml`:

```yaml
ansible_user: ansible

docker_engine_users:
  - ansible

# Data disk to partition and mount at /data. Check `lsblk` on the target.
malcolm_host_prep_data_device: /dev/sdb

# Capture NIC. Must NOT be the management or default-route interface.
# Leave empty ("") for PCAP-upload-only deployments.
malcolm_host_prep_capture_interface: ens38

malcolm_install_dir: /data/malcolm
malcolm_capture_interface: "{{ malcolm_host_prep_capture_interface }}"
malcolm_admin_username: analyst
malcolm_admin_password: "{{ vault_malcolm_admin_password }}"
malcolm_arkime_secret: "{{ vault_malcolm_arkime_secret }}"
```

Either change `inventory` in `ansible.cfg` to point at your inventory, or pass
`-i inventories/myrange/hosts.yml` on every command.

### 3. Create the vault

```bash
mkdir -p ~/.ansible
openssl rand -base64 32 > ~/.ansible/vault_pass.txt
chmod 600 ~/.ansible/vault_pass.txt

ansible-vault create inventories/myrange/group_vars/malcolm/vault.yml
```

Contents:

```yaml
vault_malcolm_admin_password: "a strong password, 8+ characters"
vault_malcolm_arkime_secret: "a long random string"
```

Keep the vault password file out of the repository. If the repository is
public, use a strong vault password: anyone can copy the encrypted file and try
to crack it offline.

### 4. Set the authorized SSH keys

Edit `automation_pubkeys` in `playbooks/bootstrap.yml`. The first entry reads
the controller's own `~/.ssh/id_ed25519.pub`. **Replace the second entry**
(the reference lab's key) with any other controller keys you need, or remove
it.

> **Warning:** the key list is exclusive. Any key on the target's `ansible`
> account that isn't in `automation_pubkeys` is removed on the next bootstrap
> run.

### 5. Bootstrap the target

Run this once per fresh target, as the target's initial admin user (prompts for
its SSH and sudo passwords):

```bash
ansible-playbook playbooks/bootstrap.yml -e ansible_user=<admin-user> -k -K
```

This creates an `ansible` account with no password, SSH-key-only login and
passwordless sudo. Use `-e ansible_user=...`, not `-u`: `vars.yml` sets
`ansible_user: ansible`, and inventory variables take precedence over `-u`.

### 6. Deploy

```bash
ansible-playbook playbooks/deploy_malcolm.yml
```

The first run downloads the release bundle and Malcolm's container images
(about 24 GB on disk), then starts Malcolm and waits for its API to answer. Expect 20–40
minutes depending on bandwidth. On first boot Malcolm pins every CPU for
several minutes while OpenSearch and Logstash initialise.

### 7. Log in

Browse to `https://<target>/` and sign in with `malcolm_admin_username` and the
vaulted password. The certificate is self-signed.

Expect empty dashboards for the first ~10 minutes. See
[Indexing delay](#indexing-delay).

## Roles

### `malcolm_host_prep`

Prepares the operating system. Each section can be skipped independently.

| Variable | Default | Purpose |
|---|---|---|
| `malcolm_host_prep_data_device` | `""` | Whole disk to partition and mount (e.g. `/dev/sdb`). Empty skips disk setup. |
| `malcolm_host_prep_data_mount` | `/data` | Mount point |
| `malcolm_host_prep_data_fstype` | `ext4` | Filesystem type |
| `malcolm_host_prep_data_mount_opts` | `defaults,noatime` | Mount options |
| `malcolm_host_prep_sysctls` | see defaults | Kernel parameters, written to `/etc/sysctl.d/60-malcolm.conf` |
| `malcolm_host_prep_limits` | see defaults | PAM limits, written to `/etc/security/limits.d/60-malcolm.conf` |
| `malcolm_host_prep_disable_sleep` | `true` | Mask systemd sleep/suspend/hibernate targets |
| `malcolm_host_prep_disable_gnome_idle` | `true` | Disable GNOME idle suspend and screen blank (skipped if GNOME is absent) |
| `malcolm_host_prep_disable_auto_upgrades` | `true` | Turn off APT periodic updates and unattended-upgrades |
| `malcolm_host_prep_capture_interface` | `""` | NIC to configure for capture. Empty skips capture setup. |
| `malcolm_host_prep_capture_offloads_off` | `gro lro gso tso sg rxvlan txvlan` | NIC offloads to disable |

Safety checks:

- **The data disk is never reformatted.** The role creates a filesystem only if
  the partition has none, and it refuses to touch the disk holding `/`.
- **Capture setup refuses to strip the IP from a working interface.** It fails
  if the capture NIC holds the default route or the address Ansible connects
  to.
- The capture NIC is configured with a NetworkManager keyfile: no IPv4 or IPv6,
  `accept-all-mac-addresses=1` (promiscuous mode), offloads off. NetworkManager
  is also told not to auto-create a DHCP profile on it.

### `docker_engine`

Installs Docker CE from `download.docker.com`, not Ubuntu's `docker.io`
package. It removes conflicting distro packages first.

| Variable | Default | Purpose |
|---|---|---|
| `docker_engine_repo_url` | `https://download.docker.com/linux/ubuntu` | Apt repository (point at a mirror for air-gapped use) |
| `docker_engine_gpg_key_url` | `<repo_url>/gpg` | Repo signing key |
| `docker_engine_repo_suite` | target's release codename | Apt suite |
| `docker_engine_daemon_config` | json-file logs, 50 MB × 3 | Written to `/etc/docker/daemon.json` |
| `docker_engine_users` | `[]` | Users added to the `docker` group |

### `malcolm`

Stages, configures and runs a pinned Malcolm release.

| Variable | Default | Purpose |
|---|---|---|
| `malcolm_version` | `26.08.0` | Release to install |
| `malcolm_release_checksum` | sha256 of the v26.08.0 bundle | Must be updated with `malcolm_version` |
| `malcolm_release_url` | GitHub release asset | Bundle download URL |
| `malcolm_image_registry` | `ghcr.io/idaholab/malcolm` | Container registry; image references are rewritten if changed |
| `malcolm_install_dir` | `/opt/malcolm` | Install root. All of Malcolm's data paths are relative to it, so put it on the data disk (`/data/malcolm`). |
| `malcolm_user` / `malcolm_group` | `malcolm` | Service account. Its UID/GID become Malcolm's `PUID`/`PGID`. |
| `malcolm_capture_interface` | `""` | Live capture NIC. Empty means PCAP upload only. |
| `malcolm_node_name` | inventory hostname | Node name shown in Arkime and dashboards |
| `malcolm_admin_username` | `analyst` | Basic auth admin user |
| `malcolm_admin_password` | (required) | From the vault; at least 8 characters |
| `malcolm_arkime_secret` | (required) | From the vault |
| `malcolm_auth_force` | `false` | Re-run auth setup, e.g. after changing the admin password |
| `malcolm_opensearch_heap` / `malcolm_logstash_heap` | scaled from RAM | See [Sizing](#sizing) |
| `malcolm_zeek_live_workers` | `1` below 24 GB, else `0` (Malcolm's auto) | Zeek live worker processes |
| `malcolm_pipeline_enabled` | `false` below 24 GB | Strelka file-scanning pipeline |
| `malcolm_env_extra` | `{}` | Extra `config/*.env` settings (see below) |
| `malcolm_ready_timeout` | `1200` | Seconds to wait for the API after start |
| `malcolm_no_log` | `true` | Hide password-bearing task output |

The role works through these steps:

1. **Stage.** It downloads `malcolm-<version>-docker_install.zip`, verifies the
   checksum, and extracts the runtime tree into `malcolm_install_dir`, owned by
   the `malcolm` user. It writes `.malcolm-version` and refuses to install over
   a different version.
2. **Configure.** It creates each `config/*.env` from its `.example` if
   missing, then sets only the keys the role owns with `lineinfile`. Settings
   you change by hand in other keys are left alone.
3. **Pull images.** It runs `docker compose pull` only when an image is missing.
4. **Authenticate.** It runs `control.py --auth-noninteractive` only when the
   htpasswd file, TLS certificates or OpenSearch credentials are missing, or
   when `malcolm_auth_force` is set.
5. **Run.** It installs and enables `malcolm.service`, starts it, and waits for
   `https://localhost/mapi/ping`. When a config change is made, the handler
   restarts the service, but only if it was already running before the play
   started.

#### Extra Malcolm settings

Any `config/*.env` key can be set with `malcolm_env_extra`. Values are merged
over the role's own settings:

```yaml
malcolm_env_extra:
  zeek.env:
    ZEEK_EXTRACTOR_MODE: interesting
  upload-common.env:
    PCAP_PIPELINE_POLLING: "true"
```

Check the `.env.example` files in the Malcolm release for available keys. Keys
change between Malcolm versions.

## Live capture

With local OpenSearch, which is what these roles deploy, Malcolm's supported
live-capture combination is:

| Component | Mode |
|---|---|
| netsniff-ng | Captures the NIC and writes rotated PCAP files |
| Arkime | Indexes the rotated PCAP (`ARKIME_ROTATED_PCAP=true`, `ARKIME_LIVE_CAPTURE=false`) |
| Zeek | Analyses the NIC live (`ZEEK_LIVE_CAPTURE=true`) |
| Suricata | Analyses the NIC live (`SURICATA_LIVE_CAPTURE=true`) |

Arkime's own live-capture mode requires a remote OpenSearch cluster and is not
used. This mirrors what Malcolm's installer configures
(`installer/utils/custom_transforms.py`).

**Promiscuous mode must also be allowed by the hypervisor or switch.** The role
enables it in the guest, but that's not enough on its own:

- **VMware Workstation:** put the capture adapter on a dedicated custom VMnet,
  or bridge it to a wired adapter. VMware may prompt to allow promiscuous mode.
- **ESXi / vSphere:** set *Promiscuous mode: Accept* on the port group.
- **Physical:** connect the NIC to a SPAN/mirror port or a network tap.

A bridged adapter on a normal switch port only sees the host's own traffic plus
broadcast and multicast. To see other hosts' unicast traffic, you need a SPAN
port, a tap, or a shared virtual network.

To check the NIC from the controller:

```bash
ansible malcolm -b -m shell -a 'cat /sys/class/net/ens38/carrier /sys/class/net/ens38/statistics/rx_packets'
```

`carrier` should be `1`, and `rx_packets` should climb between runs.

## Sizing

Memory is the binding constraint. Malcolm's own defaults (10 GB OpenSearch heap,
3 GB Logstash, Zeek workers = CPUs − 4, Strelka on) overloaded a 16 GB test
host to a load average of 29, with heavy swapping. The role sizes from
`ansible_facts['memtotal_mb']`:

| Host RAM | OpenSearch heap | Logstash heap | Zeek live workers | Strelka pipeline |
|---|---|---|---|---|
| < 20 GB | 6g | 2g | 1 | off |
| 20–24 GB | 10g | 3g | 1 | off |
| 24–28 GB | 10g | 3g | auto | on |
| 28–48 GB | 12g | 4g | auto | on |
| 48–96 GB | 24g | 4g | auto | on |
| ≥ 96 GB | 31g | 6g | auto | on |

On a 16 GB Ubuntu Desktop host, the steady state is about 13 GB used and 2 GB
available. GNOME accounts for about 2 GB of that; use Ubuntu Server for
anything beyond a lab.

Override any of these in `group_vars`, e.g. `malcolm_opensearch_heap: 8g`.

## Operating Malcolm

Malcolm runs as a systemd service:

```bash
sudo systemctl status malcolm
sudo systemctl restart malcolm
sudo systemctl stop malcolm      # takes ~80 s; OpenSearch shuts down gracefully
```

Every Malcolm compose service has `restart: "no"`, so without this unit
Malcolm would not come back after a reboot. The unit also stops the stack
before Docker shuts down, so OpenSearch gets its full shutdown grace period.

Container status and logs:

```bash
cd /data/malcolm
sudo -u malcolm docker compose --profile malcolm ps
sudo -u malcolm docker compose --profile malcolm logs -f logstash
```

### Indexing delay

New traffic does not show up immediately:

- Live PCAP rotates every 10 minutes (`PCAP_ROTATE_MINUTES`) before Arkime
  indexes it.
- Zeek and Suricata logs are picked up and indexed in batches.

Allow about 10 minutes after a start before expecting new events. An empty
result shortly after a restart is not a failure.

### Changing the admin password

```bash
ansible-vault edit inventories/<env>/group_vars/malcolm/vault.yml
ansible-playbook playbooks/deploy_malcolm.yml -e malcolm_auth_force=true
```

### Rotating the vault password

```bash
ansible-vault rekey inventories/<env>/group_vars/malcolm/vault.yml
# then update ~/.ansible/vault_pass.txt to the new password
```

## Air-gapped deployments

Every external download location is a variable. Point them at internal
mirrors:

```yaml
docker_engine_repo_url: https://mirror.range.local/docker/linux/ubuntu
docker_engine_gpg_key_url: https://mirror.range.local/docker/gpg

malcolm_release_url: https://mirror.range.local/malcolm/malcolm-26.08.0-docker_install.zip
malcolm_image_registry: registry.range.local/idaholab/malcolm
```

You also need:

- An Ubuntu apt mirror. The roles install `parted`, `python3-dotenv`,
  `python3-requests`, `python3-ruamel.yaml`, `openssl`, `apache2-utils` and
  `unzip`.
- The Malcolm images for the pinned version pushed to your registry. When
  `malcolm_image_registry` differs from the default, the role rewrites the
  compose file's image references.
- The Ansible collections, installed on the controller from a tarball or
  internal Galaxy server.

## Limitations

- **Upgrades are not automated.** The role refuses to install over a different
  Malcolm version. To upgrade, follow Malcolm's upgrade procedure by hand, or
  rebuild the host.
- **Capture NIC configuration requires NetworkManager.** Ubuntu Server
  (systemd-networkd / netplan) is not handled yet.
- **Authentication is basic auth only.** Malcolm also supports LDAP and
  Keycloak; neither is wired up.
- **Single node.** No remote OpenSearch, no Hedgehog Linux sensors.
- **Promiscuous mode outside the guest** (hypervisor, switch) is outside
  Ansible's reach.

## Troubleshooting

**`ansible-playbook` fails to connect after a rollback or rebuild.** Check
whether the target still has the `ansible` account and keys. If not, re-run
`bootstrap.yml` as the admin user (step 5).

**Apt tasks hang for minutes on Ubuntu Desktop.** PackageKit or
unattended-upgrades is holding the dpkg lock. The apt tasks wait up to
10 minutes for it (`*_apt_lock_timeout`). After the first run,
`malcolm_host_prep` disables automatic upgrades.

**Timeouts on `sudo` during the first start.** Malcolm's first boot saturates
the CPUs, so SSH and sudo can respond slowly. `ansible.cfg` sets `timeout = 30`,
and the readiness check runs without `become`. Raise `malcolm_ready_timeout` on
slow hosts.

**The capture NIC has carrier but `rx_packets` stays at 0.** The hypervisor
isn't delivering frames. On VMware Workstation, this happens when VMnet0 is
bridged to a host adapter that is down (Wi-Fi off, or "Automatic" picking the
wrong adapter). Bridge VMnet0 to the wired adapter by name in the Virtual
Network Editor, then disconnect and reconnect the VM's adapter.

**A variable seems to be ignored.** Run `ansible-inventory --host <name>` to see
what actually resolves for the host.

**Config written but never applied.** `ansible.cfg` sets
`force_handlers = True` so that a failure later in the play doesn't drop
pending restarts. If you run with a different config and a play fails after
changing `config/*.env`, later runs won't re-notify the handler; restart
`malcolm.service` by hand.

## Development

- Use fully-qualified collection names for all non-builtin modules.
- Reference facts as `ansible_facts['...']`, not the injected `ansible_*`
  variables.
- Prefix role variables with the role name.
- Make every role idempotent: a second run must report `changed=0`.
- Run `ansible-lint` before committing.

To test, roll the target back to a snapshot taken after bootstrap, run
`deploy_malcolm.yml` twice, and check that the second run reports no changes.
