# CLAUDE.md

Guidance for Claude Code working in this repository.

## Goal

Automate the installation and configuration of **Malcolm**, the open-source
network traffic analysis suite (https://github.com/cisagov/Malcolm), with
Ansible. The end product is a set of roles and playbooks that can stand up a
working Malcolm instance on a fresh Ubuntu host repeatably and without
interactive prompts.

These roles are intended to be reusable for cyber range and training
environments, including air-gapped deployments, so avoid solutions that depend
on this specific lab (hardcoded IPs, one-off manual steps, interactive scripts).

## Topology

| Role | Machine | Notes |
|---|---|---|
| Ansible controller | WSL Ubuntu 24.04 on `Eric-Gaming-Rig` | Repo lives here at `~/malcolm-ansible`. All `ansible-playbook` runs happen here. |
| Malcolm target | `malcolm01` — Ubuntu 22.04 **Desktop** VM in VMware Workstation | `192.168.153.130`, hostname `malcolm`, 16 GB RAM, 8 vCPU. NICs: `ens33` = VMware NAT, DHCP, default route (internet); `ens34` = management (`192.168.153.130`, what Ansible uses); `ens38` = capture (no IP, bridged, see below). Data disk `sdb` (250 GB) → `/data`. VM files: `C:\Users\erics\malcolm_test\` (`/mnt/c/Users/erics/malcolm_test/` from WSL). |
| Legacy controller | Ubuntu 24.04 VM (`eric-ubuntu-24-04`) | No longer the primary controller. Kept as a staging / air-gap testbed. Shares the WSL controller's SSH keypair (copied over), so it is authorized on the target by the same key. |

The controller is **not** in the inventory. Only managed hosts are listed.

### Capture network (lab)

- `ens38` is VMware Network Adapter 3 (`ethernet2` in the `.vmx`), **Bridged**
  (VMnet0) to the workstation's **wired** Ethernet adapter (Wi-Fi is off),
  with "Replicate physical network connection state" enabled. LAN is
  `192.168.40.0/24`; the workstation is `192.168.40.109`.
- Verified 2026-09-24: carrier up, PROMISC, 0 kernel drops; captured unicast
  frames between other MACs, i.e. promiscuous mode works in guest and VMware.
- It sees the **workstation's own traffic** (including WSL, which NATs out
  through it) plus LAN broadcast/multicast. Other LAN hosts' unicast needs a
  switch SPAN port or tap. Treat captures as containing personal traffic.
- For range use, the intended layout is a dedicated **Custom** VMnet (host
  adapter and DHCP off) shared with the lab VMs; no Ansible change needed, the
  interface stays `ens38`.
- Quick check from the controller:
  `ansible malcolm -b -m shell -a 'cat /sys/class/net/ens38/carrier /sys/class/net/ens38/statistics/rx_packets' </dev/null >out 2>&1`

## Access model

- Ansible connects to targets as the **`ansible`** service account.
- That account has **no password** (locked in `/etc/shadow`). SSH key only.
- Passwordless sudo via `/etc/sudoers.d/90-ansible`.
- `playbooks/bootstrap.yml` is the source of truth for authorized keys and uses
  `exclusive: true`. **Any key not listed in `automation_pubkeys` is removed on
  the next run.** If you add a key by hand, add it to the playbook too or it
  will be wiped.
- The WSL controller's own key is pulled in with
  `lookup('file', '~/.ssh/id_ed25519.pub')`. Keys belonging to other machines
  must be pasted as literal strings, since the lookup runs on the controller.
- Both controllers (WSL and the legacy VM) use the **same** ed25519 keypair.
  The literal key in `automation_pubkeys` is identical to the looked-up one; it
  is kept so the playbook still authorizes that key if run from a controller
  with a different `~/.ssh/id_ed25519.pub`. If the legacy controller ever gets
  its own keypair, add its public key as a new literal entry.

## Repo layout

```
malcolm-ansible/
├── ansible.cfg
├── requirements.yml
├── inventories/
│   └── lab/
│       ├── hosts.yml
│       └── group_vars/
│           ├── all.yml          # currently empty
│           └── malcolm/
│               ├── vars.yml     # host, capture, malcolm settings; refs vault vars
│               └── vault.yml    # ansible-vault encrypted secrets
├── roles/
│   ├── malcolm_host_prep/
│   ├── docker_engine/
│   └── malcolm/
└── playbooks/
    ├── bootstrap.yml            # run once per fresh target
    └── deploy_malcolm.yml       # applies the roles below
```

Collections in `requirements.yml`: `community.docker`, `community.general`,
`ansible.posix`.

## Roles: built and planned

- [x] `bootstrap.yml` playbook — automation user, SSH keys, passwordless sudo
- [x] `docker_engine` — Docker CE from Docker's official apt repo (not
  `docker.io`), Compose v2 plugin, `daemon.json` log rotation, docker group
  membership. Must work on both jammy (target) and noble (if run against the
  legacy controller) — derive the repo suite from `ansible_facts['distribution_release']`,
  never hardcode it. Make the apt repo URL and GPG key URL role defaults so an
  air-gapped deployment can point them at a local mirror.
- [x] `malcolm_host_prep` — data disk partition/format/mount (never reformats
  an existing filesystem), `vm.max_map_count` and other sysctls, file and
  memlock ulimits, mask sleep targets, GNOME idle/suspend off via system dconf,
  APT periodic updates off, capture NIC via a NetworkManager keyfile (no IP,
  promiscuous, offloads off). Capture setup refuses to touch the default-route
  or management interface. Assumes NetworkManager (Desktop); Server/networkd
  hosts would need a netplan variant.
- [x] `malcolm` — v26.08.0 pinned (`malcolm_version` + sha256 of the
  `docker_install.zip` release asset). Unpacks the runtime tarball to
  `malcolm_install_dir` (`/data/malcolm` in the lab, so every relative data
  path lands on `sdb`), owned by a `malcolm` system user (home
  `/var/lib/malcolm`, in the docker group; its UID/GID become PUID/PGID).
  Creates `config/*.env` from the examples, then sets only the keys the role
  owns via `lineinfile` (heaps, auth mode, node name, live-capture set, Zeek
  workers, pipeline); extra keys via `malcolm_env_extra`. Pulls images only
  when missing. Auth via `control.py --auth-noninteractive` only when certs /
  htpasswd / OpenSearch creds are missing (or `malcolm_auth_force`). Starts via
  `control.py --start --quiet` only when nothing is running; config changes
  restart via handler. Waits for `/mapi/ping`. Refuses to overwrite a different
  installed version (upgrades not automated yet). Image registry is a variable
  for air-gapped mirrors.

## Malcolm-specific constraints

- **Do not drive Malcolm's `install.py` / `configure` scripts interactively.**
  They prompt. The role sets `config/*.env` keys itself and uses only the
  non-interactive modes of `scripts/control.py` (`--auth-noninteractive`,
  `--start/--restart --quiet`). Plain `docker compose up` is **not** enough:
  `start` also creates the OpenSearch keystore, bind-mount dirs, placeholder
  auth files and fixes permissions, and refuses to run until auth files exist.
- **Live capture with local OpenSearch** = netsniff-ng writes rotated PCAP for
  Arkime (`ARKIME_LIVE_CAPTURE=false`, `ARKIME_ROTATED_PCAP=true`) while Zeek
  and Suricata analyse the NIC live (`*_LIVE_CAPTURE=true`,
  `*_ROTATED_PCAP=false`). Arkime's own live mode needs remote OpenSearch.
  Mirrors `installer/utils/custom_transforms.py`.
- **Small hosts (< 24 GB):** Malcolm's defaults (OpenSearch 10g, Logstash 3g,
  Zeek live workers = CPUs − 4, Strelka pipeline on) drove `malcolm01` to load
  29 and swap. The role sets 6g/2g, 1 Zeek worker and `PIPELINE_DISABLED=true`
  there. Steady state is ~13 GB used, ~2 GB available.
- **Pin the Malcolm release** in `group_vars`. Configuration options change
  between versions.
- **Memory is the binding constraint.** Malcolm wants 16 GB minimum; the target
  has exactly that and also runs a GNOME desktop, so it is tight. Size the
  OpenSearch and Logstash heaps from `ansible_facts['memtotal_mb']` rather than
  hardcoding:

  | Host RAM | OpenSearch heap | Logstash heap |
  |---|---|---|
  | 16 GB | 6–8 GB | 2–3 GB |
  | 24 GB | 10 GB | 3 GB |
  | 32 GB | 12–14 GB | 4 GB |
  | 64 GB | 24–31 GB | 4–6 GB |

  Heap does not benefit above ~31 GB. On a 16 GB **Desktop** host (like
  `malcolm01`), use the low end of the row (~6 GB / ~2 GB): GNOME takes ~2 GB
  and Arkime, Zeek, Suricata, dashboards and page cache need the rest.
- **Data on a separate disk.** Malcolm's PCAP and OpenSearch data should live on
  a secondary VMDK mounted by `malcolm_host_prep`, not on the OS disk. Verify
  the disk exists (`lsblk`) before assuming it is present.
- **Live capture** needs promiscuous mode enabled both in the guest and in
  VMware. The VMware side is outside Ansible's reach.

## Conventions

- Fully-qualified collection names for all non-builtin modules
  (`ansible.posix.mount`, not `mount`).
- Roles must be idempotent: a second run reports `changed=0`.
- Reference facts as `ansible_facts['distribution']`, not the injected
  top-level `ansible_distribution` form (deprecated; removed in ansible-core 2.24).
- Role variables are prefixed with the role name (`docker_engine_*`).
- Collection versions are pinned in `requirements.yml`; bump deliberately.
- Tunables go in `roles/<role>/defaults/main.yml`; environment-specific values
  go in `inventories/lab/group_vars/`.
- Secrets live in `inventories/lab/group_vars/malcolm/vault.yml` (encrypted,
  committed): `vault_malcolm_admin_password`, `vault_malcolm_arkime_secret`,
  referenced from `vars.yml`. The vault password file is
  `~/.ansible/vault_pass.txt` (outside the repo, set in `ansible.cfg`); never
  commit it. The current vault and admin passwords are temporary lab values
  and are to be changed (`ansible-vault rekey`, then edit + `malcolm_auth_force`).
- Run `ansible-lint` before committing.
- Commit after each working role.

## Testing

`malcolm01` has a VMware snapshot taken after `bootstrap.yml` ran. Roll back to
it to test roles against a clean system. The `ansible` account, its keys and
its sudo rule are part of the snapshot, so roles can be run immediately after a
rollback without re-bootstrapping.

VMware snapshots include virtual hardware. The existing snapshot predates the
capture NIC and the host prep, so rolling back may remove Network Adapter 3.
Take a fresh snapshot before testing the `malcolm` role.

On a fresh target (or a snapshot older than bootstrap), run bootstrap as the
initial admin user:

```
ansible-playbook playbooks/bootstrap.yml -e ansible_user=eric -k -K
```

Use `-e ansible_user=eric`, not `-u eric`: `group_vars/malcolm/vars.yml` sets
`ansible_user: ansible`, and inventory vars take precedence over `-u`, so `-u`
would silently try to connect as the not-yet-existing `ansible` user.

**Manual prerequisite:** Ubuntu Desktop ships without an SSH server, so
`sudo apt install openssh-server` must be done on a fresh target before
bootstrap can reach it. This is the only accepted manual step.

## Gotchas hit so far

- Empty `group_vars` files fail silently. If a variable seems ignored, `cat` the
  file before assuming a path or precedence problem.
- `ansible-inventory --host <name>` is the fastest way to see what variables
  actually resolve for a host.
- The target is Ubuntu **Desktop**, not Server: no SSH server by default, GNOME
  consumes ~2 GB RAM, and PackageKit/unattended-upgrades can hold the apt lock
  mid-run. Apt tasks set `lock_timeout` to wait it out.
- From Claude Code's Bash tool, `ansible`/`ansible-playbook` abort with
  "requires blocking IO on stdin/stdout/stderr". Run them with
  `</dev/null >file 2>&1` and read the file.
- Ad-hoc `-a` strings are templated, so `{{ }}` (e.g. `docker --format`) must
  be avoided or escaped.
- `ansible_managed` is only defined inside templates; using it in `copy`
  `content:` fails.
- `nmcli general reload` rereads NM config only. New/changed connection
  keyfiles need `nmcli connection reload`.
- Handlers from a failed run are dropped unless `force_handlers = True`
  (now set in `ansible.cfg`); otherwise a config file can be written but never
  applied, and later runs won't re-notify.
- Hot-adding a vNIC in VMware can briefly drop SSH to the guest.
- VMware bridging troubleshooting: a bridged vNIC with carrier but
  `rx_packets=0`, or NO-CARRIER with link state propagation on, means VMnet0
  is bridged to an adapter that is down (e.g. Wi-Fi off, or "Automatic").
  In the Virtual Network Editor click **Change Settings** first (otherwise it
  is read-only and changes are silently discarded), set VMnet0 to the wired
  adapter by name, then untick/re-tick **Connected** on the VM's adapter.
  `vmware.log` in the VM directory shows `MACVNetLinkStateEventHandler ... up:1`
  when the bridge link is really up. Don't use **Restore Defaults**: it can
  renumber VMnet1/VMnet8 and break management and NAT.
- On ansible-core 2.19+, a var built with `{% for %}` blocks renders as a
  **string**, not a list; use nested `include_tasks` loops or filters instead.
- Malcolm's first boot pins all 8 vCPUs for ~5 min (API) to ~10 min
  (Logstash healthy). `sudo` on the target can exceed Ansible's 10 s default,
  so `ansible.cfg` sets `timeout = 30` and the readiness check runs without
  become.
- Malcolm API aggregation URL is `/mapi/agg?fields=event.provider&from=...`;
  `/mapi/agg/<field>` redirects to plain HTTP and loses the field.
- Windows interop (`powershell.exe`) is disabled in this WSL instance; Windows
  host state can only be inspected via files under `/mnt/c`.
