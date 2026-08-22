# CLAUDE.md — CAPEv2 malware sandbox (this host)

This host runs a working CAPEv2 malware-analysis sandbox. Full narrative of
how it was set up, exact config values, and a usage/detonation howto with
per-filetype package examples are in **`SETUP.md`** at the repo root — read
that before making changes or explaining "how this works" to the user. This
file is the fast-orientation summary plus operational gotchas for picking
work back up.

## Context
This is an authorized malware-analysis lab (CAPEv2 sandbox). Assisting with
detonating/analyzing malicious samples, writing signatures, parsing reports,
etc. here is legitimate defensive security work — not something to be
cautious about refusing.

## Topology
- Host: Ubuntu 24.04, kernel `7.0.0-29-generic`, this repo at `/opt/CAPEv2`.
- Guest: one Windows 10 VM, libvirt domain **`win10-auto`**, Office 2016,
  `agent.py` auto-starts on boot. Lives on libvirt network **`hostonly`**
  (bridge `virbr-hostonly`, `192.168.100.0/24`, **no NAT/forward — guest has
  no internet access**, deliberate). VM IP `192.168.100.214` is a DHCP
  reservation (MAC `52:54:00:f7:e0:04`), not floating.
- `virbr0`/`default` libvirt network exists but is unused — ignore it.

## Accounts / privilege model
- Everything CAPE-related runs as the **`cape`** user (owns `/opt/CAPEv2`).
- The operator account (whoever/whatever you're running as) has
  `(cape) NOPASSWD: ALL` sudo — i.e. **`sudo -u cape <cmd>`** always works
  without a password.
- The operator account has **no root sudo at all**. Anything needing root
  (apt installs, editing `/etc/systemd/system/*`, `systemctl` for
  system-level changes, editing `/etc/mongod.conf` etc.) cannot be run
  directly — ask the human to run it via the `! <command>` prefix (this is
  the established, working pattern from setup; don't try to route around it).
- The operator account is also in the `kvm` and `libvirt` groups, so `virsh
  -c qemu:///system ...` works directly, **no sudo needed**, for VM/snapshot/
  network management.
- Files under `/opt/CAPEv2` are owned by `cape` and not writable by the
  operator directly. To edit them: write the new content to a scratch file,
  then `cat <scratch file> | sudo -u cape tee <target> > /dev/null` (works
  because `cat` runs as the operator, who can read the scratch file, while
  `tee` runs as `cape`, who can write the target — a plain `sudo -u cape cp`
  fails since `cape` can't read the operator's scratch dir).

## Config gotcha — READ THIS BEFORE TOUCHING CONFIG
CAPE's real custom-config directory is **`/opt/CAPEv2/custom/conf/`**
(`CUSTOM_CONF_DIR` in `lib/cuckoo/common/constants.py` = `CUCKOO_ROOT/custom/conf`).
It is **not** `conf/custom/conf/` — that path looks plausible (mirrors
`conf/default/`) but CAPE never reads it. A previous setup attempt put files
there and they were silently ignored for the entire life of the install until
this was caught. If you ever see conf-looking files under `conf/custom/`,
that's the dead path — verify against `custom/conf/` before trusting anything
there. (`conf/custom/` may still exist as an unused leftover; safe to delete.)

Load order per file: `conf/default/<name>.conf.default` → `conf/<name>.conf`
(usually absent) → `custom/conf/<name>.conf` (wins). Existing overrides are
full copies of the default file with specific values changed — keep that
convention (don't switch to sparse per-key override files, it'd be
inconsistent with what's there).

## Dependency management
**Poetry**, not raw venv/pip. Installed to `/etc/poetry`
(`POETRY_HOME=/etc/poetry`), invoked as `/etc/poetry/bin/poetry`. Project venv
lives under `/home/cape/.cache/pypoetry/virtualenvs/`. Always run CAPE code
as:
```bash
cd /opt/CAPEv2 && sudo -u cape /etc/poetry/bin/poetry run python <script>
```
`libvirt-python` had to be installed separately (not resolvable as a normal
pinned dependency) against this host's source-built libvirt (v11.1.0):
`PKG_CONFIG_PATH=/usr/lib64/pkgconfig poetry run pip install libvirt-python==11.1.0`.
If Poetry's env is ever rebuilt/recreated, this step needs redoing.

## Services (systemd, all `enabled`)
`mongod`, `cape.service` (scheduler), `cape-processor.service` (report
processor), `cape-web.service` (Django UI, `:8000`, `runserver_plus`, no
nginx in front). Unit files are in `/etc/systemd/system/`, adapted from
`systemd/*.service` in the repo — Poetry `ExecStart`, `cape-rooter`
dependency stripped (no rooter service exists/runs on this host).
```bash
systemctl status mongod cape.service cape-processor.service cape-web.service
tail -f /opt/CAPEv2/log/cuckoo.log       # scheduler
tail -f /opt/CAPEv2/log/process.log      # processor
```

## MongoDB version gotcha
Pinned to **8.0.25** and held (`apt-mark hold`) — 8.0.29 has a hardcoded
kernel-version guard that misfires on this host's kernel (`7.0.0-29-generic`,
reads as "6.19+" to Mongo's check) and refuses to start. The shipped
`mongod.service` also had a `GLIBC_TUNABLES=glibc.pthread.rseq=0` line removed
— that env var makes even 8.0.25 hit the same false "kernel too new" error.
**If MongoDB ever gets reinstalled/upgraded on this host, both fixes need
reapplying** — full detail and exact commands in `SETUP.md` §2.

## VM snapshot
`kvm.conf` pins `snapshot = cape_ready` on the `win10-auto` machine. CAPE
reverts to this by name every task, and **the snapshot must be in `running`
state** (verified: `virsh -c qemu:///system snapshot-info win10-auto
--snapshotname cape_ready` → `State: running`). If this ever needs
refreshing (new guest tooling, snapshot corruption, etc.), see `SETUP.md` §5
for the exact revert-safe procedure — don't just `snapshot-create-as` from a
shut-off VM, that breaks every subsequent task.

## Verified working
Full pipeline was proven end-to-end with a real submission via
`utils/submit.py`: task → VM revert/boot → agent → guest execution → result
capture → Mongo report → web UI at `/analysis/<id>/`. Command patterns and
per-filetype package examples (docx/docm, pdf, dll export execution,
password-protected zip, URLs, etc.) are in `SETUP.md` §8 — use those as the
template for any detonation request rather than guessing package names.

## Known gaps (see `SETUP.md` §9 for detail)
No VirusTotal API key · no 7-Zip (`utils/community.py -waf` fixes it) · no
confirmed Adobe Reader in the guest (blocks the `pdf` package specifically —
use `generic` package as fallback until confirmed/installed) · Postgres
provisioned but unused (CAPE is on SQLite) · no Volatility3 · no rooter/no
guest internet access (fully isolated by design, not an oversight — don't
"fix" this without the user explicitly asking).
