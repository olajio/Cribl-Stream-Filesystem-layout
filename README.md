# Cribl Stream — Filesystem & Directory Structure Reference

> **`$CRIBL_HOME`** defaults to `/opt/cribl` on Linux installs.  
> All paths below are relative to `$CRIBL_HOME` unless otherwise noted.

---

## Table of Contents

- [Base Layout (All Nodes)](#base-layout-all-nodes)
- [bin/ — CLI & Executables](#bin--cli--executables)
- [default/ — Factory Defaults](#default--factory-defaults)
- [local/ — User Configuration](#local--user-configuration)
  - [local/_system/](#local_system)
  - [local/cribl/ — Core Config Files](#localcribl--core-config-files)
  - [local/cribl/auth/](#localcriblauth)
  - [local/cribl/certificates/](#localcriblcertificates)
  - [local/cribl/lookups/](#localcribllookups)
- [log/ — Internal Logs](#log--internal-logs)
- [Leader Node — Exclusive Paths](#leader-node--exclusive-paths)
  - [local/cribl/groups/\<group\>/](#localcriblgroupsgroup)
  - [.git/ — Version Control](#git--version-control)
  - [local/cribl/config-bundles/](#localcriblconfig-bundles)
  - [IPC Socket Files](#ipc-socket-files)
  - [Leader Ports](#leader-ports)
- [Worker Node — Exclusive Paths](#worker-node--exclusive-paths)
  - [data/ — Persistent Queues & Staging](#data--persistent-queues--staging)
  - [state/ — Runtime State](#state--runtime-state)
  - [Worker Ports](#worker-ports)
- [Key Config Files Reference](#key-config-files-reference)
- [Environment Variables](#environment-variables)
- [Notes & Gotchas](#notes--gotchas)

---

## Base Layout (All Nodes)

```
$CRIBL_HOME/                   (/opt/cribl by default)
├── bin/                       # Executables and CLI
├── default/                   # Read-only factory defaults (do not edit)
├── local/                     # User-managed config (overrides default/)
│   ├── _system/               # Instance-level system settings
│   └── cribl/                 # Core Cribl config files
├── data/                      # Persistent queues, samples, lookups (runtime)
├── state/                     # Runtime state, metrics, collector state
└── log/                       # Internal operational logs
```

> **Precedence rule:** `local/` always takes precedence over `default/`. There is no layered merging — the file in `local/` fully replaces the one in `default/`.

---

## bin/ — CLI & Executables

```
$CRIBL_HOME/bin/
├── cribl                      # Main binary (all CLI commands)
└── cribl.sh                   # Legacy shell wrapper
```

### Common CLI Commands

| Command | Description |
|---|---|
| `./cribl start` | Start the Cribl service |
| `./cribl stop` | Stop the Cribl service |
| `./cribl restart` | Restart (cancels in-flight collection jobs) |
| `./cribl reload` | Soft reload — applies route/pipeline/function changes |
| `./cribl status` | Show process status |
| `./cribl mode-master` | Switch instance to Leader mode |
| `./cribl mode-worker -H <leader> -p <port> -u <token>` | Join a Leader as a Worker |
| `./cribl mode-single` | Revert to standalone single-instance mode |
| `./cribl diag` | Collect diagnostic bundle |
| `./cribl git` | Version control operations |
| `./cribl vars` | Manage global variables |
| `./cribl limits` | Manage system limits |
| `./cribl keys` | Manage encryption keys |
| `./cribl node` | Node management |
| `./cribl version` | Display version info |

> **Reload vs Restart:** Use `reload` after changing routes, pipelines, or functions. Use `restart` after changing inputs, outputs, or system-level settings.

---

## default/ — Factory Defaults

```
$CRIBL_HOME/default/
├── cribl/                     # Default stream configs for all features
└── <pack-name>/               # One directory per shipped Pack
```

- These files are **read-only** — do not edit them directly.
- They are overwritten on every upgrade.
- To customize, copy the file to the corresponding path under `local/` and edit it there.

---

## local/ — User Configuration

```
$CRIBL_HOME/local/
├── _system/
│   ├── instance.yml           # Mode, distributed settings, API config
│   └── service.yml            # Process/service-level settings
└── cribl/
    ├── cribl.yml              # Core system config (API, TLS, ports)
    ├── logger.yml             # Log level and log output settings
    ├── limits.yml             # System resource limits
    ├── notifications.yml      # Notification channel config
    ├── iometrics.yml          # I/O metrics configuration
    ├── certificates.yml       # TLS certificate references
    ├── groups.yml             # Worker Group definitions (leader)
    ├── mappings.yml           # Worker-to-group mapping rules
    ├── fleet-mappings.yml     # Fleet mapping rules (Edge)
    ├── outpost.yml            # Outpost configuration
    ├── leader.yml             # HA Leader settings (HA deployments only)
    ├── jobs.yml               # Collector job definitions
    ├── job-limits.yml         # Job concurrency limits
    ├── breakers.yml           # Event breaker rules
    ├── inputs.yml             # Source definitions
    ├── outputs.yml            # Destination definitions
    ├── regexes.yml            # Named regex library
    ├── schemas.yml            # Data schema definitions
    ├── samples.yml            # Sample data references
    ├── parsers.yml            # Parser definitions
    ├── vars.yml               # Global variable definitions
    ├── scripts.yml            # Script definitions
    ├── persistent-queue.yml   # PQ global settings
    ├── messages.yml           # Custom message templates
    ├── auth/                  # Auth and identity files
    ├── certificates/          # TLS cert and key files
    ├── lookups/               # Lookup table files
    ├── roles/                 # RBAC role definitions
    └── users/                 # User account definitions
```

### local/_system/

| File | Purpose |
|---|---|
| `instance.yml` | Declares mode (`master`/`worker`/`single`), distributed settings, Leader URL, API bind address/port |
| `service.yml` | Process ownership, service manager settings |

**Example `instance.yml` for Leader:**
```yaml
distributed:
  mode: master
api:
  host: 0.0.0.0
  port: 9000
```

**Example `instance.yml` for Worker:**
```yaml
distributed:
  mode: worker
  master:
    host: criblleader.mycompany.com
    port: 4200
```

### local/cribl/ — Core Config Files

> In a **distributed deployment**, Worker Group-specific configs live under  
> `local/cribl/groups/<group-name>/` on the Leader, not directly in `local/cribl/`.

### local/cribl/auth/

```
local/cribl/auth/
├── <guid>.dat                 # Instance GUID (generated on first run)
├── cribl.secret               # Master secret key for encryption
└── ssh/
    └── git.key                # SSH key for remote Git repo auth
```

> ⚠️ **VM/image baking warning:** Delete `<guid>.dat` before snapshotting. Cribl regenerates it on next start. Duplicate GUIDs across Workers cause management issues.

> ⚠️ **Security:** `cribl.secret` and `ssh/git.key` are sensitive. Secure your Git remote repo — these files are included in commits if version control is enabled.

### local/cribl/certificates/

```
local/cribl/certificates/
├── *.pem / *.crt              # TLS certificates
└── *.key                      # Private keys
```

Used for API/UI TLS, mTLS between leader and workers, and source/destination TLS.

### local/cribl/lookups/

```
local/cribl/lookups/
├── *.csv                      # CSV lookup tables
└── *.mmdb                     # MaxMind GeoIP databases
```

> **Important:** The `data/` directory also needs to be synced between instances in a standalone multi-instance deployment — it contains lookup files used at runtime.

---

## log/ — Internal Logs

```
$CRIBL_HOME/log/
├── cribl.log                  # Main operational log (all processes)
├── api.log                    # API process log
├── worker-0.log               # Worker process 0 log
├── worker-N.log               # Worker process N log
└── ...
```

Log levels are configured in `local/cribl/logger.yml`.

---

## Leader Node — Exclusive Paths

### local/cribl/groups/\<group\>/

This is the most important Leader-exclusive path. Each Worker Group gets its own subdirectory containing the full configuration for that group.

```
$CRIBL_HOME/local/cribl/groups/
└── <worker-group-name>/
    ├── default/               # Group-level default overrides
    └── local/
        ├── cribl/
        │   ├── inputs.yml     # Sources for this group
        │   ├── outputs.yml    # Destinations for this group
        │   ├── routes.yml     # Routing rules
        │   ├── pipelines/     # Pipeline definitions
        │   │   └── <pipeline-name>.json
        │   ├── jobs.yml       # Collector jobs
        │   ├── lookups/       # Group-scoped lookup files
        │   ├── breakers.yml   # Event breakers
        │   ├── regexes.yml    # Named regexes
        │   ├── schemas.yml    # Schemas
        │   ├── parsers.yml    # Parsers
        │   ├── samples.yml    # Samples
        │   ├── vars.yml       # Group-level variables
        │   ├── scripts.yml    # Scripts
        │   └── persistent-queue.yml
        └── _system/
            └── instance.yml   # Group-level instance overrides
```

> **Key behavior:** When you commit and deploy from the Leader, Cribl packages these files into a config bundle and pushes them to all Workers in the group. Workers then apply the config. Any manual edits on a Worker are **overwritten** on the next deploy.

### .git/ — Version Control

```
$CRIBL_HOME/.git/              # Embedded git repository (Leader only)
```

- Tracks all configuration changes across all Worker Groups.
- Powers the **Commit** and **Deploy** workflow in the UI.
- Can be connected to a remote GitHub/GitLab repo for backup and GitOps.
- Requires `git` to be installed on the Leader host for commit history markers to appear in the Monitoring dashboard.

### local/cribl/config-bundles/

```
$CRIBL_HOME/local/cribl/config-bundles/
└── <group-name>-<version>.tar.gz   # Versioned config bundles
```

These are the packaged bundles that Workers download and apply when the Leader deploys a configuration.

### IPC Socket Files

```
/tmp/cribl-*/                  # Default location (OS temp dir)
```

- Created at runtime for inter-process communication between Leader API and config helper processes.
- **Risk:** Linux `systemd-tmpfiles` can clean `/tmp` periodically (often every 10 days), breaking the Leader UI pages for Workers/Fleets and Monitoring.
- **Fix option 1:** Add `/etc/tmpfiles.d/cribl.conf` with: `X /tmp/cribl-*`
- **Fix option 2:** Move to a protected dir via UI → Settings > Global > Distributed Settings > Leader Settings > **Helper processes socket dir** (e.g. `/var/tmp/cribl`)

### Leader Ports

| Port | Protocol | Purpose |
|---|---|---|
| `9000` | TCP | UI, REST API, inbound management traffic |
| `4200` | TCP | Worker communication; bootstrap API (`/init/install-worker.sh`) |

---

## Worker Node — Exclusive Paths

### data/ — Persistent Queues & Staging

```
$CRIBL_HOME/data/
├── <destination-id>/          # Persistent Queue (PQ) buffers per destination
├── staging/                   # Temp staging for non-streaming destinations
└── captures/                  # Packet captures (if used)
```

> If PQ directories are configured to a custom path outside `$CRIBL_HOME`, exclude those paths from AV scanning and ensure they are on the same filesystem as `$CRIBL_HOME` (or a dedicated volume).

### state/ — Runtime State

```
$CRIBL_HOME/state/
├── jobs/                      # Collector job state (resumable jobs)
├── metrics/                   # Local copy of monitoring metrics
└── ...
```

> If the Worker Node goes down, in-flight data in memory is lost. Persistent Queues on Sources and Destinations mitigate this risk.

### Worker Ports

| Port | Protocol | Purpose |
|---|---|---|
| `9000` | TCP | Worker local UI (if teleport/remote access is enabled) |
| `4200` | TCP | Outbound to Leader (Workers initiate connection to Leader) |
| Source ports | varies | Configured per Source (e.g. 9997 for Splunk TCP, 8088 for HEC, 514 for Syslog) |

---

## Key Config Files Reference

| File | Scope | What it controls |
|---|---|---|
| `local/_system/instance.yml` | Node | Mode, Leader URL, API host/port |
| `local/cribl/cribl.yml` | Global | API TLS, system-wide settings |
| `local/cribl/groups.yml` | Leader | Worker Group definitions and settings |
| `local/cribl/mappings.yml` | Leader | Worker-to-group auto-mapping rules |
| `local/cribl/leader.yml` | Leader (HA) | HA failover configuration |
| `local/cribl/outpost.yml` | Leader | Outpost connection settings |
| `local/cribl/inputs.yml` | Group | Source definitions |
| `local/cribl/outputs.yml` | Group | Destination definitions |
| `local/cribl/routes.yml` | Group | Data routing rules |
| `local/cribl/pipelines/*.json` | Group | Pipeline function chains |
| `local/cribl/breakers.yml` | Group | Event breaker rules |
| `local/cribl/regexes.yml` | Group | Named regex library |
| `local/cribl/vars.yml` | Group | Global variables |
| `local/cribl/jobs.yml` | Group | Collector job definitions |
| `local/cribl/lookups/*.csv` | Group | Lookup enrichment files |
| `local/cribl/auth/<guid>.dat` | Node | Instance GUID |
| `local/cribl/auth/cribl.secret` | Node | Master encryption secret |
| `local/cribl/logger.yml` | Node | Log level and output config |
| `local/cribl/limits.yml` | Node | CPU/memory/throughput limits |
| `local/cribl/notifications.yml` | Node | Notification targets |
| `local/cribl/certificates.yml` | Node | TLS cert references |
| `local/cribl/persistent-queue.yml` | Group | PQ global settings |

---

## Environment Variables

| Variable | Description |
|---|---|
| `CRIBL_HOME` | Internal variable pointing to the Cribl binary/install location |
| `CRIBL_VOLUME_DIR` | Override for writable data directories (overrides `CRIBL_HOME` for data paths) |
| `CRIBL_CONF_DIR` | Override for config directory (used in HA failover mode) |
| `CRIBL_DIST_MODE` | Sets instance mode: `worker` or `leader` (internally `master`) |
| `CRIBL_DIST_LEADER_URL` | Leader URL for Workers (format: `tcp://criblmaster@<host>:<port>`) |
| `CRIBL_TMP_DIR` | Override for temp staging directory (default: OS temp dir) |
| `CRIBL_BOOTSTRAP` | Bootstrap config URL, file path, or YAML string applied on first start |
| `CRIBL_BOOTSTRAP_HOST` | Override hostname shown in bootstrap script modal |

---

## Notes & Gotchas

**1. Config precedence**  
`local/` → fully overrides `default/`. No merging. If a file exists in `local/`, the `default/` version is ignored entirely.

**2. Worker config is read-only in practice**  
Manual edits on a Worker Node's `local/cribl/` are overwritten on the next Leader deploy. Use the Leader UI/CLI for all persistent changes.

**3. `internal` mode naming**  
Leader mode is internally called `master` (legacy name). You'll see `master` in CLI commands (`mode-master`), `instance.yml` keys, environment variables (`CRIBL_DIST_MODE=leader` is the new name but `master` still works), and Helm charts.

**4. Protect socket files**  
IPC sockets in `/tmp/cribl-*` are cleaned by OS tmpfile daemons. Configure exclusions or move the socket dir.

**5. Same filesystem requirement**  
`/opt/cribl` and all its subdirectories must reside on the same device. Do not mount separate volumes inside `/opt/cribl`. For PQs and lookups requiring dedicated storage, use a separate path outside `$CRIBL_HOME` and reference it in config.

**6. AV/EDR exclusions**  
Exclude the following from antivirus/EDR process scanning:
- PQ and staging directories
- Non-streaming destination staging paths
- `$CRIBL_HOME` subdirectories used for runtime data

**7. Git on Leader**  
`git` must be installed on the Leader host for full version control functionality. On hardened/FIPS systems, add the git-core directory to your fapolicy trust file.

**8. HA Leader**  
In a Leader HA setup, `leader.yml` in `$CRIBL_HOME/local/cribl/` replicates `instance.yml` content (minus failover config) and is synced to the failover volume. Settings changed via UI in failover mode are written to `leader.yml` on the failover volume, not the local `instance.yml`.

---

## References

- [Cribl Stream Configuration Files](https://docs.cribl.io/stream/configuration-files/)
- [Distributed Deployment](https://docs.cribl.io/stream/deploy-distributed/)
- [Set Up Leader and Worker Nodes](https://docs.cribl.io/stream/setting-up-leader-and-worker-nodes/)
- [Bootstrap Workers from Leader](https://docs.cribl.io/stream/deploy-workers/)
- [Environment Variables](https://docs.cribl.io/stream/environment-variables/)
- [CLI Reference](https://docs.cribl.io/stream/cli-reference/)
- [Leader High Availability](https://docs.cribl.io/stream/deploy-add-second-leader/)
- [Version Control](https://docs.cribl.io/stream/version-control/)
