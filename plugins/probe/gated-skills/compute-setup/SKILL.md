---
name: compute-setup
description: Registers a team's GPUs with Probe so it sees each machine, its GPUs and its jobs. Use when someone asks to register, track, monitor or connect their GPUs, cluster, Kubernetes, SLURM, SSH hosts or cloud machines to Probe.
---
# Compute setup

The steps, the same for every coding agent, are one page. Read it raw and follow it, start to end:

```sh
curl -fsSL https://research.prbe.ai/register-gpus.md
```

A web-fetch tool's summary is not the page: use curl's output.

## Offline: the page could not be read

Follow these instead, spelling every command `npx -y probe-research …`:

1. `whoami`: if its `role` is not `owner` or `admin`, say "Registering GPUs needs a team owner or admin; ask
   one of them to run it." and stop.
2. `compute clusters --json`: if it answers `NOT_ENABLED`, say "Probe compute is not turned on for your
   account yet." and stop.
3. `compute connect --dry-run --json --accept-env`. Tell the person in plain words what each target is and what
   connecting it changes, then ask which to register (question tool, multi-select). Their pick is their yes; no
   pick: say "Nothing changed." and stop.
4. Each picked target: `compute connect --target <name> --yes --accept-env`. Never pass `--yes` before their
   pick. On a failure, show its error (connect rolls back by itself).
5. Check: `compute clusters --json` and `compute nodes --json`.

Never stop or change a running job, never fix a machine by hand, and never print an enrollment token. Never
uninstall on your own: an owner or admin removes Probe on the dashboard's Compute page. Only if the person asks to
undo a connect made from this computer: `compute disconnect <name>`, then `compute disconnect <name> --yes` after
their yes.
