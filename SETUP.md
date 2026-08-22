# CAPEv2 Setup — This Host

This document records exactly how this CAPEv2 install was brought from a fresh
checkout to a working malware-analysis sandbox, so the setup can be understood,
reproduced, or extended later. It also covers day-to-day usage.

Environment:
- Host: Ubuntu 24.04 (noble), Python 3.12.3, KVM/QEMU + libvirt (source-built,
  v11.1.0, via `installer/kvm-qemu.sh`), CAPEv2 checked out at `/opt/CAPEv2`.
- Guest: a single Windows 10 VM (`win10-auto`), libvirt domain running under
  QEMU/KVM, MS Office 2016, with `agent.py` (CAPE Agent v0.22) auto-starting
  on boot.
- Service account: `cape` (created by the CAPEv2 installer). All CAPE
  processes run as `cape`; an operator account with sudoers rules
  (`(cape) NOPASSWD: ALL`, plus `kvm`/`libvirt` group membership) is used to
  impersonate `cape` and to manage the VM.

---

## 1. Python dependencies — Poetry

CAPE's dependencies are managed with Poetry, installed to a dedicated prefix
(matching the official installer, `installer/cape2.sh`):

```bash
curl -sSL https://install.python-poetry.org | sudo POETRY_HOME=/etc/poetry python3 -
sudo /etc/poetry/bin/poetry self add poetry-plugin-shell
sudo chmod -R a+rX /etc/poetry
```

This installs Poetry to `/etc/poetry` (root-owned, but world-readable/executable
so the `cape` user can invoke `/etc/poetry/bin/poetry`).

Project dependencies, installed as `cape` from `/opt/CAPEv2`:

```bash
sudo -u cape /etc/poetry/bin/poetry install
```

Poetry creates its virtualenv under
`/home/cape/.cache/pypoetry/virtualenvs/capev2-<hash>-py3.12`. All `ExecStart`
lines in the systemd units (see §5) and all commands in this document invoke
CAPE via `poetry run` from `/opt/CAPEv2`, which resolves to that venv.

### `libvirt-python`

`libvirt-python` isn't a normal PyPI wheel — it's a C extension that links
against the *installed* libvirt shared library, so it has to be built locally
against this host's source-built libvirt (v11.1.0), whose headers/pkg-config
file live under a non-default path:

```bash
sudo -u cape bash -c '
  export PKG_CONFIG_PATH=/usr/lib64/pkgconfig
  /etc/poetry/bin/poetry run pip install libvirt-python==11.1.0
'
```

Verify:

```bash
cd /opt/CAPEv2 && sudo -u cape /etc/poetry/bin/poetry run python -c \
  "import libvirt; print(libvirt.getVersion())"
# -> 11001000
```

---

## 2. Databases

**PostgreSQL** — already installed and running, with a `cape` database/role
already provisioned. CAPE is left on **SQLite** (`conf/custom` → `[database]
connection =` blank) since this is a single-node lab setup; Postgres is
present but currently unused (safe to ignore, or wire up later by setting
`connection = postgresql://cape:<password>@localhost:5432/cape`).

**MongoDB** — required for full report storage and the web UI's analysis
views. Installed from MongoDB's official apt repo:

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pgp.mongodb.com/server-8.0.asc | sudo gpg --dearmor -o /etc/apt/keyrings/mongo.gpg --yes
echo "deb [signed-by=/etc/apt/keyrings/mongo.gpg arch=amd64] https://repo.mongodb.org/apt/ubuntu $(lsb_release -cs)/mongodb-org/8.0 multiverse" \
  | sudo tee /etc/apt/sources.list.d/mongodb.list
sudo apt-get update
sudo apt-get install -y mongodb-org
```

**Important caveat on this host:** MongoDB 8.0.29 (the latest build at setup
time) has a hardcoded startup check that refuses to run on Linux kernel
6.19+ (`SERVER-121912`) — this host runs `7.0.0-29-generic`, which trips that
guard. Versions 8.0.8 through 8.0.25 all start cleanly on this kernel. The
package was therefore pinned:

```bash
sudo apt-get install --allow-downgrades -y \
  mongodb-org=8.0.25 mongodb-org-database=8.0.25 \
  mongodb-org-database-tools-extra=8.0.25 mongodb-org-server=8.0.25 \
  mongodb-org-mongos=8.0.25 mongodb-org-tools=8.0.25
sudo apt-mark hold mongodb-org mongodb-org-database mongodb-org-database-tools-extra \
  mongodb-org-server mongodb-org-mongos mongodb-org-tools
```

The shipped `mongod.service` unit also sets `GLIBC_TUNABLES=glibc.pthread.rseq=0`
(a workaround for an unrelated, older glibc/rseq bug). On this host that
tunable causes mongod's kernel-compat probe to misfire the same "kernel too
new" error even on 8.0.25. It was removed:

```bash
sudo sed -i '/GLIBC_TUNABLES=glibc.pthread.rseq=0/d' /usr/lib/systemd/system/mongod.service
sudo systemctl daemon-reload
sudo systemctl enable --now mongod
```

If MongoDB is ever upgraded/reinstalled on this host, re-check both of the
above (the apt hold prevents an accidental `apt upgrade` from reintroducing
the 8.0.29 issue; a fresh package reinstall would reintroduce the
`GLIBC_TUNABLES` line and need the same `sed` fix again).

Finally, Mongo reporting was enabled in CAPE's config (see §3).

---

## 3. CAPE configuration (`custom/conf/`)

CAPE reads config as: `conf/default/<name>.conf.default` → `conf/<name>.conf`
(if present) → **`custom/conf/<name>.conf`** (highest precedence) — per
`lib/cuckoo/common/constants.py`'s `CUSTOM_CONF_DIR = CUCKOO_ROOT/custom/conf`.
That's `/opt/CAPEv2/custom/conf/`, **not** `conf/custom/`. All overrides below
live there, each as a full copy of the matching `conf/default/*.conf.default`
with the specific values below changed (keeps every other option visible and
documented in one place, matching upstream's file layout).

### `custom/conf/cuckoo.conf`
- `[cuckoo] machinery = kvm`
- `[cuckoo] categories = static, pcap, url, file`
- `[resultserver] ip = 192.168.100.1` — the host's address on the VM's
  network (`virbr-hostonly`), i.e. the address the guest can actually reach.
- `[database] connection =` (blank → SQLite, see §2)

### `custom/conf/kvm.conf`
```ini
[kvm]
machines = win10-auto
interface = virbr-hostonly
dsn = qemu:///system

[win10-auto]
label = win10-auto
platform = windows
ip = 192.168.100.214
tags = win10
snapshot = cape_ready
arch = x86
```
- `label` must equal the actual libvirt domain name (`win10-auto`) — CAPE
  calls `virsh`/libvirt directly using this value.
- `ip` is the VM's address on `virbr-hostonly`, now a fixed DHCP reservation
  (see §4).
- `interface` is the tcpdump-capture bridge, `virbr-hostonly` (not the unused
  default NAT bridge `virbr0`).
- `snapshot = cape_ready` pins the exact snapshot CAPE reverts to per task
  (see §5) rather than relying on "whatever is current."
- `tags = win10` — required so CAPE can auto-select Windows-10-appropriate
  packages/behavior for submissions tagged/targeted that way.

### `custom/conf/auxiliary.conf`
- `[auxiliary_modules] windows_static_route_gateway = 192.168.100.1`
  (only takes effect if `windows_static_route` is turned on; kept consistent
  with the real gateway either way).
- `[sniffer] interface = virbr-hostonly`, `host = root@192.168.100.1`
  (tcpdump capture interface, matching the real guest bridge).
- `[gateways]` left empty/commented — the guest network is fully isolated
  (see below), so no INetSim/real-gateway routing is configured.

### `custom/conf/reporting.conf`
- `[mongodb] enabled = yes` (only line changed from default) — everything
  else (host `127.0.0.1`, port `27017`, db `cuckoo`) is upstream's default.

### Networking/isolation decision
The guest (`win10-auto`) sits on libvirt's `hostonly` network
(`virbr-hostonly`, `192.168.100.0/24`), which has **no NAT/forward** — the
guest has zero outbound internet access. This is deliberate: safest default
for handling live malware. There is no rooter service running and no
`[gateways]` entries. (If simulated internet is wanted later, INetSim +
CAPE's `utils/rooter.py` + a `cape-rooter.service` would need to be added and
`[gateways]` populated — not done here.)

---

## 4. Libvirt networking

The VM (`win10-auto`, MAC `52:54:00:f7:e0:04`) was getting a plain DHCP lease
on `virbr-hostonly` with no guarantee of stability across reboots. A DHCP
host reservation was added so its IP is now fixed:

```bash
virsh -c qemu:///system net-update hostonly add ip-dhcp-host \
  "<host mac='52:54:00:f7:e0:04' ip='192.168.100.214'/>" --live --config
```

(Run as the operator account — libvirt group membership is enough, no `sudo`
needed for `qemu:///system` here.)

---

## 5. VM snapshot (`cape_ready`)

CAPE's KVM machinery (`lib/cuckoo/common/abstracts.py`, `LibVirtMachinery`)
reverts to the snapshot named in `kvm.conf`'s `snapshot =` option (or, if
unset, the most recently created one) **and that snapshot must be in the
`running` state** — revert restores memory + disk together; there's no boot
step. A snapshot taken while the VM is off will make every analysis fail
immediately with *"Your snapshot MUST BE in running state!"*.

The golden snapshot was created like this:

```bash
# 1. Make sure the VM is up and the agent is answering
virsh -c qemu:///system list --all
curl -m 5 http://192.168.100.214:8000/
# -> {"message": "CAPE Agent!", "version": "0.22", ...}

# 2. Snapshot it, live
virsh -c qemu:///system snapshot-create-as --domain win10-auto \
  --name cape_ready \
  --description "CAPE-ready state: agent.py running, Defender disabled, clean" \
  --atomic
```

Verified with a real revert cycle:

```bash
virsh -c qemu:///system snapshot-revert win10-auto --snapshotname cape_ready --running
# VM comes back running, agent immediately reachable on :8000
```

**To refresh this snapshot later** (e.g. after installing new guest tooling):
get the VM into the desired state and running, confirm the agent responds,
delete the old `cape_ready` snapshot (`virsh -c qemu:///system snapshot-delete
win10-auto cape_ready`), then repeat the `snapshot-create-as` command above.

---

## 6. systemd services

Three services (adapted from CAPE's shipped templates in `systemd/`, using
Poetry, with the `cape-rooter` dependency removed since no rooter is running
— see §3):

**`/etc/systemd/system/cape.service`** — main scheduler:
```ini
[Unit]
Description=CAPE
Documentation=https://github.com/kevoreilly/CAPEv2

[Service]
WorkingDirectory=/opt/CAPEv2/
ExecStart=/etc/poetry/bin/poetry run python cuckoo.py
User=cape
Group=cape
Restart=always
RestartSec=5m
TimeoutStopSec=4m
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
```

**`/etc/systemd/system/cape-processor.service`** — report processor:
```ini
[Unit]
Description=CAPE report processor
Documentation=https://github.com/kevoreilly/CAPEv2
Wants=cape.service

[Service]
Type=exec
WorkingDirectory=/opt/CAPEv2
Environment=CAPE_PROCESSING_TIMEOUT=900
Environment=CAPE_PARALLEL=7
Environment=CAPE_MAXTASKSPERCHILD=7
Environment=CAPE_ID=auto
EnvironmentFile=-/opt/CAPEv2/conf/cape-processor.env
EnvironmentFile=-/opt/CAPEv2/custom/conf/cape-processor.env
ExecStart=/etc/poetry/bin/poetry run python utils/process.py
User=cape
Group=cape
Restart=always
RestartSec=10
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
```

**`/etc/systemd/system/cape-web.service`** — Django web UI:
```ini
[Unit]
Description=CAPE WSGI app
Documentation=https://github.com/kevoreilly/CAPEv2

[Service]
WorkingDirectory=/opt/CAPEv2/web
ExecStart=/etc/poetry/bin/poetry run python manage.py runserver_plus 0.0.0.0:8000 --traceback --keep-meta-shutdown
User=cape
Group=cape
Restart=always
RestartSec=5m

[Install]
WantedBy=multi-user.target
```

Installed and enabled with:

```bash
sudo cp cape.service cape-processor.service cape-web.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now cape.service cape-processor.service cape-web.service
```

Before first start, Django's own (separate, SQLite-backed) auth database
needs its migrations applied once:

```bash
cd /opt/CAPEv2/web && sudo -u cape /etc/poetry/bin/poetry run python manage.py migrate
```

Check status any time with:
```bash
systemctl status cape.service cape-processor.service cape-web.service
journalctl -u cape.service -f      # live scheduler log
tail -f /opt/CAPEv2/log/cuckoo.log
tail -f /opt/CAPEv2/log/process.log
```

---

## 7. End-to-end verification performed

A harmless test file was submitted through the real pipeline and confirmed:
task queued → `win10-auto` reverted to `cape_ready` and booted → agent
connected → sample executed on the guest → result server captured behavior
→ processor generated a report into MongoDB → visible in the web UI at
`/analysis/<id>/`. See §8 for how to do this yourself.

---

## 8. Day-to-day usage

### Check everything is up
```bash
systemctl is-active mongod cape.service cape-processor.service cape-web.service
virsh -c qemu:///system list --all      # win10-auto should show "shut off" when idle —
                                         # CAPE powers it down between tasks and reverts
                                         # + boots it fresh per task.
curl -s http://192.168.100.214:8000/    # only responds while a task has the VM running
```

### Submit a sample from the CLI
This is the exact method used to validate the whole pipeline above:

```bash
cd /opt/CAPEv2
sudo -u cape /etc/poetry/bin/poetry run python utils/submit.py /path/to/sample.bin
```

Notes:
- **File extension matters.** CAPE skips a hardcoded list of "junk" extensions
  outright — `.txt .md .yml .yaml .yar .yara .gitignore` and a few filenames
  (`readme`, `license`, `makefile`, ...) — regardless of content. Don't name
  or rename a real sample with one of these.
- Watch progress live:
  ```bash
  tail -f /opt/CAPEv2/log/cuckoo.log      # scheduler / VM lifecycle
  tail -f /opt/CAPEv2/log/process.log     # report generation
  ```
- Task IDs print on submission (`Success: File "..." added as task with ID N`).
  Once processing finishes ("Reports generation completed for Task #N" in
  `process.log`), the report is at:
  ```
  http://<host>:8000/analysis/N/
  ```

### Using the web UI
Browse to `http://<host>:8000/`:
- **Submit** page — drag/drop or browse to a file, pick a package/options,
  submit.
- **Recent** / analysis list — see completed tasks.
- **`/analysis/<id>/`** — full report: behavior, dropped files, network
  traffic, screenshots, signatures.

(First-time login: create a Django admin user with
`sudo -u cape /etc/poetry/bin/poetry run python web/manage.py createsuperuser`
if authentication is enforced in `web.conf`.)

### Forcing a specific analysis package
CAPE auto-detects a package from the file's type/magic in most cases, but you
can force one with `--package`, and pass package-specific options with
`--options "key=value,key2=value2"`. Packages live in
`analyzer/windows/modules/packages/`; useful ones for this guest (Windows 10
+ Office 2016):

| File type | Package | Notes |
|---|---|---|
| `.docx` / `.docm` / `.doc` (Word) | `doc` or `doc2016` | `doc2016` explicitly targets the Office 2016 `WINWORD.EXE` path and opens with `/q /dde /n` (suppresses DDE prompts) — recommended on this guest. |
| `.xlsx` / `.xlsm` / `.xls` (Excel) | `xls` or `xls2016` | same idea as doc/doc2016. |
| `.pptx` / `.pptm` / `.ppt` (PowerPoint) | `ppt` or `ppt2016` | same idea. |
| `.pdf` | `pdf` | Opens with Adobe Reader/Acrobat — **requires Adobe Reader or Acrobat installed in the guest**, which is not currently confirmed present (only Office 2016 was mentioned as installed). If missing, either install Adobe Reader in the guest and refresh `cape_ready`, or fall back to `generic`, which just uses the file's default OS association. |
| `.exe` / `.dll` | `exe` / `dll` | `dll` supports `--options "function=<export>"` or `--options "dllloader=<path>"` to control which export runs. |
| `.js` / `.vbs` / `.ps1` / `.hta` / `.wsf` | `js`, `vbs`, `ps1`, `hta`, `wsf` | script packages, run via `wscript`/`powershell`/`mshta` respectively. |
| `.lnk` | `lnk` | |
| `.zip` / `.rar` (archives, incl. password-protected) | `zip` / `archive` | `--options "password=<pw>"` for protected archives; CAPE's demux logic will usually auto-extract and pick a package for the inner file anyway. |
| URL | `--url` flag, no package needed | see below. |
| `.msi` | `msi` | |
| `.one` (OneNote) | `one` | relevant for OneNote-attachment phishing lures. |
| `.chm` | `chm` | |
| shellcode (raw) | `shellcode` / `shellcode_x64` | |

Examples:

```bash
# Office doc, forcing the Office-2016-aware package
poetry run python utils/submit.py --package doc2016 /path/to/invoice.docm

# PDF
poetry run python utils/submit.py --package pdf /path/to/lure.pdf

# DLL, calling a specific export
poetry run python utils/submit.py --package dll --options "function=DllRegisterServer" /path/to/sample.dll

# A URL instead of a file
poetry run python utils/submit.py --url http://example.test/path

# Password-protected zip
poetry run python utils/submit.py --package zip --options "password=infected" /path/to/sample.zip

# Custom timeout (seconds) and tagging the target OS explicitly
poetry run python utils/submit.py --timeout 300 --tags win10 /path/to/sample.bin
```

(All of the above assume `cd /opt/CAPEv2 && sudo -u cape /etc/poetry/bin/poetry run ...`
as shown in the first example — omitted from the rest for brevity.)

### Common useful `--options` keys
- `enable_family_signatures=1` / general CAPE signature options — see
  `docs/` and `modules/signatures/` for the full set; most need no manual
  flag and just run.
- `procmemdump=1` — dump process memory for CAPE payload extraction.
- `route=inetsim` / `route=vpn0` — only usable if a rooter + routing target is
  configured (not the case on this host today, see §3).
- `full_memory_dump=1` (needs `cuckoo.conf [cuckoo] memory_dump = on`, off by
  default here) — full VM memory snapshot on shutdown, for Volatility-style
  analysis (Volatility3 itself is **not installed** on this host currently).

---

## 9. Known gaps / optional follow-ups (not required for basic operation)

- **VirusTotal integration** has no API key configured — lookups fail
  harmlessly (`VT: Request failed` in `process.log`). Add a key in
  `custom/conf/integrations.conf` (`[virustotal]`) to enable it.
- **7-Zip** isn't installed, so some archive-extraction paths log `Missed
  7zip executable`. Fix with:
  ```bash
  sudo -u cape /etc/poetry/bin/poetry run python utils/community.py -waf
  ```
- **Adobe Reader/Acrobat** is not confirmed installed in the guest — the
  `pdf` package needs one of them; without it, PDF samples should use the
  `generic` package instead (or install a reader in the guest and refresh
  the `cape_ready` snapshot).
- **PostgreSQL** is installed and has a `cape` database ready to go, but
  CAPE is currently on SQLite. Fine for single-node/lab use; switch by
  setting `[database] connection` in `custom/conf/cuckoo.conf` if this ever
  needs to scale or support concurrent web+scheduler access more robustly.
- **Volatility3** is not installed — only needed if memory-forensics
  analysis (`memory_dump`) is turned on.
- **No rooter / no simulated or real internet for the guest** — intentional
  (see §3). Add `utils/rooter.py` as a service + INetSim if outbound traffic
  needs to be observed/simulated for detonations that require network
  callbacks to "succeed."
- A stray, unused directory `/opt/CAPEv2/conf/custom/` may still exist from
  before the config-path fix in §3 — CAPE never reads it; safe to
  `rm -rf` if you want it gone.
