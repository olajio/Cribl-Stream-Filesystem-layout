**Base install path** — `$CRIBL_HOME` defaults to `/opt/cribl`. Everything below is relative to it.

**`bin/`** — All CLI binaries. The main `cribl` executable drives everything: `start`, `stop`, `restart`, `reload`, `status`, `diag`, `git`, and the mode-switching commands (`mode-master`, `mode-worker`, `mode-single`).

**`default/`** — Factory-shipped read-only configs. You can view the default configurations by looking in the `default` directory instead of `local`. Pack configuration resides within a separate subdirectory for each Pack at `$CRIBL_HOME/default/<pack_name>`. Never edit these directly.

**`local/`** — Where all your live config lives. As on most *nix systems, Cribl configurations in `local` take precedence over those in `default`. There is no layering of configuration files.

Key files under `local/`:
- `local/_system/instance.yml` — mode declaration (`master` for leader, `worker` for worker)
- `local/cribl/auth/<guid>.dat` — a GUID Cribl generates on first run, stored in `CRIBL_HOME/local/cribl/auth`. Remove before baking a VM image.
- `local/cribl/` — all the core config YAMLs: `inputs.yml`, `outputs.yml`, `cribl.yml`, `logger.yml`, `limits.yml`, `groups.yml`, `mappings.yml`, `notifications.yml`, `certificates.yml`, `leader.yml` (HA only), `outpost.yml`, `jobs.yml`, `breakers.yml`, `regexes.yml`, `schemas.yml`, `samples.yml`, `vars.yml`, `scripts.yml`, `parsers.yml`, `iometrics.yml`, `persistent-queue.yml`

---

**Leader-exclusive paths**

- **`local/cribl/groups/<group-name>/`** — all paths are relative to `$CRIBL_HOME/groups/<group-name>/` in a distributed deployment. One directory per Worker Group, containing `inputs.yml`, `outputs.yml`, `pipelines/`, `routes.yml`, `jobs.yml`, `lookups/`, etc.
- **`.git/`** — embedded git repo on the leader that tracks every config commit and powers the commit/deploy workflow.
- **`local/cribl/config-bundles/`** — versioned bundles the leader pushes down to workers on deploy.
- **`/tmp/cribl-*/`** — socket files for inter-process communication (IPC) between the Leader and distributed processes. These sockets are essential for ensuring that Edge/Worker Nodes successfully connect to the Leader and for certain metrics services. Protect these from OS tmpfile cleanup (e.g. via `/etc/tmpfiles.d`).
- **Ports**: `9000` (UI/API) and `4200` (worker communication and bootstrap endpoint).

---

**Worker-exclusive paths**

- `local/cribl/` — deployed group config from the leader. You can manually change the local configuration on a Worker Node, but changes won't persist on the filesystem. To permanently modify the configuration, save, commit, and deploy it from the Leader Node.
- `data/` — persistent queue buffers, staging directories for non-streaming destinations, and sample captures.
- `state/` — runtime state including collector state, metrics snapshots, and PQ references.

---

**Present on both**

- `log/` — `cribl.log`, `worker-*.log`, `api.log`, and related operational logs.
- `local/cribl/auth/` — instance GUID, TLS keys, `cribl.secret`.
- The full `local/cribl/*.yml` config file set (though workers receive theirs from the leader rather than editing them manually).
