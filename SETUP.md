# CAPEv2 Setup — This Host

Reference for how this CAPEv2 install is configured and operated.

## Versions (as of 2026-11-24)

| Component | Version / detail |
|---|---|
| Host OS | Ubuntu 24.04 (noble), kernel `7.0.0-43-generic` |
| CAPEv2 | branch `custom` = upstream `599bf9729` (2026-11-24) + local setup commit; remote `origin/custom` |
| Python | 3.14.10, uv-managed, at `/usr/local/share/uv/python/cpython-3.14-linux-x86_64-gnu` |
| Package manager | uv 0.13.9 at `/usr/local/bin/uv`; venv `/opt/CAPEv2/.venv` (from `uv.lock`) |
| Key Python packages | Django 6.1.3, SQLAlchemy 2.0.41, alembic 1.9.4, pymongo 4.19.0, granian 2.8.8, flare-capa 10.0.0, flare-floss 3.2.0, CAPE-parsers 0.1.61, SFlock2 0.3.86, yara-python 4.5.4, yara-x 1.24.0, libvirt-python 11.1.0, psycopg 3.3.6, regex 2026.11.23 |
| libvirt / QEMU | 11.1.0, source-built via `installer/kvm-qemu.sh` (pkg-config in `/usr/lib64/pkgconfig`) |
| MongoDB | 8.0.25, `apt-mark hold` (see §2) |
| CAPE DB | SQLite `/opt/CAPEv2/db/cuckoo.db`, alembic schema `4c2d_drop_inline_tenancy` |
| Web server | Granian (WSGI), 4 workers, `0.0.0.0:8000` |
| Guest | Windows 10, libvirt domain `win10-auto`, Office 2016, CAPE Agent **0.22** (repo ships 0.25, not deployed — see §5) |
| Service account | `cape` (owns `/opt/CAPEv2`); operator has `(cape) NOPASSWD: ALL`, `kvm`/`libvirt` groups, no root sudo |

---

## 1. Python environment — uv

All CAPE code runs from `/opt/CAPEv2/.venv` (Python 3.14) via uv:

```bash
cd /opt/CAPEv2 && sudo -u cape /usr/local/bin/uv run --no-sync python <script>
```

`--no-sync` prevents uv from re-resolving the env on every call (same flag the
systemd units use).

Installed by `sudo bash installer/cape2.sh migrate-uv`, which installs uv +
Python 3.14, creates `.venv`, runs
`uv sync --locked --no-install-project --inexact --python 3.14` (includes the
`dev` group: pytest etc.), builds `libvirt-python` against the local libvirt,
installs `yara-x`, and syncs systemd units from `systemd/`.

Extra packages installed into the venv on top of `uv.lock`: `flare-floss`,
`regex`, `batch_deobfuscator` (git), and everything in
`extra/optional_dependencies.txt` (XLMMacroDeobfuscator, pyattck, pyclamd,
ImageHash, socks5man, peepdf-3 git build, pydot/graphviz, urlextract, …).
That set overrides 5 locked versions (rich 13.9.4, peepdf-3 git, maxminddb,
wcwidth, yarl); a later `uv sync --inexact` reverts those 5 to the lock.
Re-apply after a sync with:

```bash
cd /opt/CAPEv2 && sudo -u cape /usr/local/bin/uv pip install --python .venv/bin/python \
  -r extra/optional_dependencies.txt git+https://github.com/DissectMalware/batch_deobfuscator
```

Install further packages with:

```bash
cd /opt/CAPEv2 && sudo -u cape /usr/local/bin/uv pip install --python .venv/bin/python <pkg>
```

Re-sync after a `git pull` (root needed only for the installer wrapper;
as `cape` this works directly):

```bash
cd /opt/CAPEv2 && sudo -u cape /usr/local/bin/uv sync --locked --no-install-project --inexact
```

`uv sync --inexact` keeps manually installed packages; without `--inexact` they
are removed. `libvirt-python` must be rebuilt if the venv is ever recreated:

```bash
cd /opt/CAPEv2 && sudo -u cape env PKG_CONFIG_PATH=/usr/lib64/pkgconfig \
  /usr/local/bin/uv pip install --python .venv/bin/python libvirt-python==11.1.0
sudo -u cape .venv/bin/python -c "import libvirt; print(libvirt.getVersion())"   # -> 11001000
```

---

## 2. Databases

**SQLite** — CAPE task DB at `/opt/CAPEv2/db/cuckoo.db`
(`[database] connection =` blank in `custom/conf/cuckoo.conf`). Schema is
managed by alembic; CAPE refuses to start on a version mismatch. After an
update that adds migrations:

```bash
cd /opt/CAPEv2/utils/db_migration && sudo -u cape /usr/local/bin/uv run --no-sync alembic upgrade head
cd /opt/CAPEv2 && sudo -u cape /usr/local/bin/uv run --no-sync python web/manage.py migrate --noinput
sqlite3 /opt/CAPEv2/db/cuckoo.db 'select * from alembic_version'   # must equal SCHEMA_VERSION in lib/cuckoo/core/database.py
```

**PostgreSQL** — installed, `cape` database/role provisioned, unused. To use
it: `connection = postgresql+psycopg://cape:<password>@localhost:5432/cape`
(psycopg3 driver; the old `postgresql://` psycopg2 form no longer works).

**MongoDB 8.0.25** — report storage for the web UI. Installed from
`repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0`. Constraints on this host:

- Pinned to 8.0.25 and held: 8.0.29 has a startup kernel-version guard
  (`SERVER-121912`) that rejects this host's 7.x kernel. 8.0.8–8.0.25 work.
  ```bash
  sudo apt-get install --allow-downgrades -y \
    mongodb-org=8.0.25 mongodb-org-database=8.0.25 \
    mongodb-org-database-tools-extra=8.0.25 mongodb-org-server=8.0.25 \
    mongodb-org-mongos=8.0.25 mongodb-org-tools=8.0.25
  sudo apt-mark hold mongodb-org mongodb-org-database mongodb-org-database-tools-extra \
    mongodb-org-server mongodb-org-mongos mongodb-org-tools
  ```
- The `Environment=GLIBC_TUNABLES=glibc.pthread.rseq=0` line is removed from
  `/usr/lib/systemd/system/mongod.service` (it triggers the same false
  "kernel too new" error). A package reinstall restores it:
  ```bash
  sudo sed -i '/GLIBC_TUNABLES=glibc.pthread.rseq=0/d' /usr/lib/systemd/system/mongod.service
  sudo systemctl daemon-reload && sudo systemctl restart mongod
  ```

---

## 3. CAPE configuration (`custom/conf/`)

Load order: `conf/default/<name>.conf.default` → `conf/<name>.conf` (absent)
→ **`/opt/CAPEv2/custom/conf/<name>.conf`** (wins; `CUSTOM_CONF_DIR` in
`lib/cuckoo/common/constants.py`). Not `conf/custom/` — CAPE never reads that
path; don't create it.

Overrides are full copies of the default file with specific values changed.
Keys added upstream later and missing from an override fall back to the
default, so new upstream keys need no action unless their value must differ.

### `custom/conf/cuckoo.conf`
- `[cuckoo] machinery = kvm`
- `[cuckoo] categories = static, pcap, url, file`
- `[resultserver] ip = 192.168.100.1`
- `[database] connection =` (blank → SQLite)

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

### `custom/conf/auxiliary.conf`
- `[auxiliary_modules] windows_static_route_gateway = 192.168.100.1`
- `[sniffer] interface = virbr-hostonly`, `host = root@192.168.100.1`
- `[gateways]` empty

### `custom/conf/reporting.conf`
- `[mongodb] enabled = yes` (only change from default)

### Env files
`conf/cape-processor.env`, `conf/cape-web.env`, `conf/guac-web.env` — seeded
from `conf/default/*.env.default`, all-comment (no overrides). Per-host
overrides for the systemd units go in `custom/conf/<unit>.env`.

### Network isolation
Guest is on libvirt network `hostonly` (bridge `virbr-hostonly`,
`192.168.100.0/24`, no NAT/forward): no internet access, by design. No rooter
service, no `[gateways]`. The scheduler logs
`CRITICAL: Unable to passthrough root command (/tmp/cuckoo-rooter) ... inetsim_disable`
at startup because of this; it is harmless.

---

## 4. Libvirt networking

- Network `hostonly`, bridge `virbr-hostonly`, host IP `192.168.100.1`.
- DHCP reservation: MAC `52:54:00:f7:e0:04` → `192.168.100.214`
  ```bash
  virsh -c qemu:///system net-update hostonly add ip-dhcp-host \
    "<host mac='52:54:00:f7:e0:04' ip='192.168.100.214'/>" --live --config
  ```
- `default`/`virbr0` exists, unused.

Operator runs `virsh -c qemu:///system ...` without sudo (libvirt group).

---

## 5. VM snapshot (`cape_ready`)

CAPE reverts to the snapshot named in `kvm.conf` (`cape_ready`) for every
task. It **must be in `running` state** (memory + disk); a shut-off snapshot
makes every task fail with *"Your snapshot MUST BE in running state!"*.

```bash
virsh -c qemu:///system snapshot-info win10-auto --snapshotname cape_ready | grep State   # running
```

Guest agent in the snapshot: **0.22**. Repo `agent/agent.py` is **0.25**
(includes guest `/status` connection-backlog fixes). The host does not
enforce an agent version; 0.22 works. To deploy 0.25: copy
`agent/agent.py` into the guest over the existing auto-start agent, then
refresh the snapshot as below.

Refresh procedure:
```bash
# VM running in the desired state, agent answering:
curl -m 5 http://192.168.100.214:8000/        # {"message": "CAPE Agent!", "version": "...", ...}
virsh -c qemu:///system snapshot-delete win10-auto cape_ready
virsh -c qemu:///system snapshot-create-as --domain win10-auto --name cape_ready \
  --description "CAPE-ready state: agent.py running, Defender disabled, clean" --atomic
virsh -c qemu:///system snapshot-revert win10-auto --snapshotname cape_ready --running   # verify
```

---

## 6. systemd services

Units in `/etc/systemd/system/` are the repo's `systemd/*.service` installed
as-is by the installer.
All enabled, all run as `cape`:

| Unit | ExecStart |
|---|---|
| `cape.service` | `uv run --no-sync python cuckoo.py` (`KillMode=mixed`, `TimeoutStopSec=4m`) |
| `cape-processor.service` | `uv run --no-sync python utils/process.py` (pebble engine, `CAPE_PARALLEL=7`, `CAPE_PROCESSING_TIMEOUT=900`) |
| `cape-web.service` | `collectstatic` pre-start, then `uv run --no-sync granian --interface wsgi ... web.wsgi:application` on `GRANIAN_HOST=0.0.0.0`, `GRANIAN_PORT=8000`, `GRANIAN_WORKERS=4`; static files from `web/static_collected` |
| `mongod.service` | MongoDB (see §2) |

The units contain `Wants=/After=cape-rooter.service`; that unit is not
installed, which systemd tolerates. Granian settings are overridable in
`custom/conf/cape-web.env` (e.g. `GRANIAN_HOST=127.0.0.1`).
Granian logs a warning that `net.ipv4.tcp_migrate_req` is off; optional fix:
`sudo sysctl -w net.ipv4.tcp_migrate_req=1` (persist in `/etc/sysctl.d/`).

```bash
systemctl status cape cape-processor cape-web mongod --no-pager
journalctl -u cape-web -f
tail -f /opt/CAPEv2/log/cuckoo.log     # scheduler
tail -f /opt/CAPEv2/log/process.log    # processor
```

---

## 7. Updating CAPE

```bash
sudo systemctl stop cape cape-processor cape-web
cd /opt/CAPEv2 && sudo -u cape git pull                     # branch custom
sudo -u cape cp db/cuckoo.db db/cuckoo.db.bak
sudo -u cape /usr/local/bin/uv sync --locked --no-install-project --inexact
# then the alembic + Django migrations from §2
sudo systemctl start cape cape-processor cape-web
```

`sudo bash installer/cape2.sh update` does the same but also runs a plain
`git pull`, re-syncs systemd units from the repo, and seeds new
`conf/*.env` files. Git needs
`git -c safe.directory=/opt/CAPEv2 ...` when run as a non-`cape` user.

Check for new upstream Python requirements (`requires-python` in
`pyproject.toml`) before pulling.

### Tests

```bash
cd /opt/CAPEv2 && sudo -u cape env HOME=/home/cape \
  /usr/local/bin/uv run --no-sync python -m pytest -p no:cacheprovider -n 6 -q
```

Baseline on this host (2026-11-25): 1944 passed, 76 skipped, 2 failed. Both
failures are host-environment specific:
- `tests/test_autoprocess_yara_prewarm.py` — the live `cape-processor` holds
  `log/process_auto.lock`; passes with the service stopped.
- `tests/test_objects.py::TestFiles::test_get_yara` — the test feeds
  yara-python rules, but with `yara-x` installed `File.get_yara` scans via
  yara-x (`'Rules' object is not an instance of 'Rules'`). Production
  compiles and scans with yara-x consistently.

---

## 8. Day-to-day usage

### Check everything is up
```bash
systemctl is-active mongod cape cape-processor cape-web
virsh -c qemu:///system list --all      # win10-auto is "shut off" when idle
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8000/   # 200
```

### Submit from the CLI
```bash
cd /opt/CAPEv2 && sudo -u cape /usr/local/bin/uv run --no-sync python utils/submit.py /path/to/sample.bin
```

- Extensions `.txt .md .yml .yaml .yar .yara .gitignore` and names like
  `readme`, `license`, `makefile` are skipped as junk regardless of content.
- Submission prints the task ID. When `process.log` shows
  "Reports generation completed for Task #N", the report is at
  `http://<host>:8000/analysis/N/`.

### Web UI
`http://<host>:8000/` — Submit page, Recent list, `/analysis/<id>/` reports.
Create an admin if auth is enforced in `web.conf`:
```bash
cd /opt/CAPEv2 && sudo -u cape /usr/local/bin/uv run --no-sync python web/manage.py createsuperuser
```

### Analysis packages
Force with `--package`, pass options with `--options "k=v,k2=v2"`. Packages:
`analyzer/windows/modules/packages/`.

| File type | Package | Notes |
|---|---|---|
| `.docx` / `.docm` / `.doc` | `doc2016` (or `doc`) | Office 2016 `WINWORD.EXE`, `/q /dde /n` |
| `.xlsx` / `.xlsm` / `.xls` | `xls2016` (or `xls`) | |
| `.pptx` / `.pptm` / `.ppt` | `ppt2016` (or `ppt`) | |
| `.pdf` | `pdf` | needs Adobe Reader/Acrobat in guest (not confirmed); otherwise `generic` |
| `.exe` / `.dll` | `exe` / `dll` | `dll`: `function=<export>` or `dllloader=<path>` |
| `.js` / `.vbs` / `.ps1` / `.hta` / `.wsf` | `js`, `vbs`, `ps1`, `hta`, `wsf` | |
| `.lnk` | `lnk` | |
| `.zip` / `.rar` | `zip` / `archive` | `password=<pw>` |
| `.msi` | `msi` | |
| `.one` | `one` | |
| `.chm` | `chm` | |
| raw shellcode | `shellcode` / `shellcode_x64` | |
| URL | `--url <url>` | |

Examples (prefix each with
`cd /opt/CAPEv2 && sudo -u cape /usr/local/bin/uv run --no-sync`):

```bash
python utils/submit.py --package doc2016 /path/to/invoice.docm
python utils/submit.py --package dll --options "function=DllRegisterServer" /path/to/sample.dll
python utils/submit.py --url http://example.test/path
python utils/submit.py --package zip --options "password=infected" /path/to/sample.zip
python utils/submit.py --timeout 300 --tags win10 /path/to/sample.bin
```

Useful options: `procmemdump=1`; `full_memory_dump=1` (needs
`[cuckoo] memory_dump = on`, off here; Volatility3 not installed);
`route=...` unavailable (no rooter).

---

## 9. Known gaps

- **Guest Python has no Pillow** — analyzer logs `No module named 'PIL'`,
  so in-guest screenshots are disabled (true for every task since setup). Fix:
  `pip install Pillow` for the guest's 32-bit Python
  (`C:\Users\admin\AppData\Local\Programs\Python\Python312-32`), then
  refresh `cape_ready` (§5).
- **Guest agent** 0.22, repo has 0.25 (§5). Works.
- **VirusTotal** — no API key (`VT: Request failed` in `process.log`; set
  `custom/conf/integrations.conf [virustotal]`).
- **Adobe Reader** not confirmed in guest — use `generic` for PDFs.
- **Volatility3** not installed.
- **No rooter / no guest internet** — intentional.
