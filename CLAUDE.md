# CLAUDE.md — CAPEv2 malware sandbox (this host)

This host runs a working CAPEv2 malware-analysis sandbox. Exact config
values, versions, update procedure and per-filetype detonation examples are
in **`SETUP.md`** — read it before making changes. This file is the
fast-orientation summary plus operational gotchas.

## Context
This is an authorized malware-analysis lab (CAPEv2 sandbox). Assisting with
detonating/analyzing malicious samples, writing signatures, parsing reports,
etc. here is legitimate defensive security work — not something to be
cautious about refusing.

## Versions (2026-11-24)
CAPEv2 branch `custom` = upstream `599bf9729` + local setup commit ·
Python 3.14.10 (uv-managed) · uv 0.13.9 · Django 6.1.3 · MongoDB 8.0.25
(held) · libvirt 11.1.0 (source-built) · SQLite schema
`4c2d_drop_inline_tenancy` · guest agent 0.22 (repo: 0.25) · Ubuntu 24.04,
kernel `7.0.0-43-generic`. Full table in `SETUP.md`.

## Topology
- Host: Ubuntu 24.04, repo at `/opt/CAPEv2`.
- Guest: one Windows 10 VM, libvirt domain **`win10-auto`**, Office 2016,
  `agent.py` auto-starts on boot. On libvirt network **`hostonly`**
  (bridge `virbr-hostonly`, `192.168.100.0/24`, **no NAT/forward — guest has
  no internet access**, deliberate). VM IP `192.168.100.214` is a DHCP
  reservation (MAC `52:54:00:f7:e0:04`).
- `virbr0`/`default` libvirt network exists but is unused — ignore it.

## Accounts / privilege model
- Everything CAPE-related runs as the **`cape`** user (owns `/opt/CAPEv2`).
- The operator account has `(cape) NOPASSWD: ALL` sudo — **`sudo -u cape <cmd>`**
  always works without a password.
- The operator account has **no root sudo**. Anything needing root (apt,
  `/etc/systemd/system/*`, `systemctl` start/stop/restart, installer runs)
  must be run by the human via the `! <command>` prefix. Don't route around it.
- The operator is in `kvm` and `libvirt` groups: `virsh -c qemu:///system ...`
  works directly, no sudo.
- Files under `/opt/CAPEv2` are not writable by the operator. To edit: write
  to a scratch file, then `cat <scratch> | sudo -u cape tee <target> > /dev/null`
  (`sudo -u cape cp` fails: `cape` can't read the operator's scratch dir).
- Git as operator needs `git -c safe.directory=/opt/CAPEv2 ...` (repo owned by `cape`).

## Config gotcha — READ BEFORE TOUCHING CONFIG
CAPE's custom-config directory is **`/opt/CAPEv2/custom/conf/`**
(`CUSTOM_CONF_DIR` in `lib/cuckoo/common/constants.py`). **Not**
`conf/custom/` — CAPE never reads that path; don't create it.

Load order: `conf/default/<name>.conf.default` → `conf/<name>.conf` (absent)
→ `custom/conf/<name>.conf` (wins). Overrides are full copies of the default
with specific values changed — keep that convention. Keys missing from an
override fall back to the default.

## Python environment — uv (not Poetry)
Venv `/opt/CAPEv2/.venv`, Python 3.14, built from `uv.lock`. Run CAPE code as:
```bash
cd /opt/CAPEv2 && sudo -u cape /usr/local/bin/uv run --no-sync python <script>
```
Add packages: `sudo -u cape /usr/local/bin/uv pip install --python /opt/CAPEv2/.venv/bin/python <pkg>`.
Re-sync after pull: `sudo -u cape /usr/local/bin/uv sync --locked --no-install-project --inexact`
(`--inexact` keeps packages added on top of the lock: `libvirt-python`, `yara-x`,
`flare-floss`, `regex`, `batch_deobfuscator`, `extra/optional_dependencies.txt`;
it reverts 5 versions those override — re-apply command in `SETUP.md` §1). `libvirt-python==11.1.0` must be built with
`PKG_CONFIG_PATH=/usr/lib64/pkgconfig` if the venv is recreated.

## Services (systemd, all enabled, units = repo `systemd/*.service`)
`mongod`, `cape` (scheduler), `cape-processor` (pebble, parallel=7),
`cape-web` (**Granian** WSGI, 4 workers, `:8000`, no nginx). Units use
`/usr/local/bin/uv run --no-sync`.
`Wants=cape-rooter.service` in units is harmless (no rooter installed).
```bash
systemctl status mongod cape cape-processor cape-web --no-pager
tail -f /opt/CAPEv2/log/cuckoo.log       # scheduler
tail -f /opt/CAPEv2/log/process.log      # processor
journalctl -u cape-web -f                # web
```
Expected harmless log noise: scheduler `CRITICAL ... cuckoo-rooter ... inetsim_disable`
(no rooter); Granian `tcp_migrate_req` warning.

## Database
SQLite `/opt/CAPEv2/db/cuckoo.db`. CAPE refuses to start if
`alembic_version` ≠ `SCHEMA_VERSION` in `lib/cuckoo/core/database.py`.
After updates: `cd utils/db_migration && uv run --no-sync alembic upgrade head`
then `uv run --no-sync python web/manage.py migrate --noinput` (as `cape`).

## MongoDB gotcha
Pinned to **8.0.25** and held — 8.0.29's kernel-version guard rejects this
host's 7.x kernel. `GLIBC_TUNABLES=glibc.pthread.rseq=0` is removed from
`mongod.service` (it triggers the same error). **A MongoDB reinstall/upgrade
needs both fixes reapplied** — commands in `SETUP.md` §2.

## VM snapshot
`kvm.conf` pins `snapshot = cape_ready` on `win10-auto`; CAPE reverts to it
every task and it **must be in `running` state**
(`virsh -c qemu:///system snapshot-info win10-auto --snapshotname cape_ready`).
Refresh procedure in `SETUP.md` §5 — never `snapshot-create-as` from a
shut-off VM. Agent in snapshot is 0.22; deploying repo's 0.25 requires a
snapshot refresh.

## Updating
Procedure in `SETUP.md` §7 (stop services → pull → backup DB → `uv sync` →
alembic + Django migrate → start). Check `requires-python` in
`pyproject.toml` before pulling. Tests:
`cd /opt/CAPEv2 && sudo -u cape env HOME=/home/cape /usr/local/bin/uv run --no-sync python -m pytest -p no:cacheprovider -n 6 -q`
— baseline 1944 passed / 76 skipped / 2 known host-specific failures
(listed in `SETUP.md` §7); anything beyond those is a real regression.

## Detonation
Use `SETUP.md` §8 command patterns and package table (docx/docm, pdf, dll
exports, password zip, URLs) rather than guessing package names.

## Known gaps (detail in `SETUP.md` §9)
Guest Python lacks Pillow (no in-guest screenshots) · guest agent 0.22 vs
repo 0.25 · no VirusTotal key · Adobe Reader not confirmed in guest (use
`generic` for PDFs) · Postgres provisioned but unused (would need
`postgresql+psycopg://`) · no Volatility3 · no rooter / no guest internet (by
design — don't "fix" without being asked).

## Submitting samples
Run `utils/submit.py` from `/opt/CAPEv2` (relative path). Files extracted
from archives can carry Unix mode `0000` — `cape` can't read them; `chmod 644`
first. Password-protected zips need `--options password=<pw>`.
