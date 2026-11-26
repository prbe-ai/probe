---
name: compute-setup
description: Connects a team's GPU machines or cluster to Probe with `probe compute connect`, so Probe sees each machine, its GPUs and its jobs. Use when someone asks to track, monitor or connect their GPUs, cluster, Kubernetes, SLURM, SSH hosts or cloud machines to Probe, or asks Probe for a job queue.
---
# Compute setup

`probe compute connect` finds where this team's GPU work runs and installs probe-node there (it
reports each machine's GPUs and jobs to Probe). It changes nothing until the person says yes.

1. See what it finds and the exact change for each target:
   ```bash
   probe compute connect --dry-run
   ```
   Each target shows how it would be connected:
   - **Kubernetes**: probe-node as a DaemonSet (Helm), one pod per node.
   - **SSH host**: probe-node installed on the host.
   - **Pod mode** (Modal, Daytona and other container sandboxes): a wrapper that starts the job
     with probe-node beside it. It ends with the job, so Probe cannot tell when that machine sits idle.
   - **`skypilot@<cluster>`**: offered only to a team with no job queue (no SLURM, no SkyPilot
     server). It installs a SkyPilot API server with Probe's policy into the team's Kubernetes or onto
     one VM; then `sky jobs launch --env PROBE_PROJECT=<project>` opens a Probe run for each job. A team
     with its own queue keeps it.
2. Tell the person, in plain words, which targets it found and what each change does. Ask which
   ones to connect, and wait for a clear yes to that plan.
3. Connect only what they said yes to:
   ```bash
   probe compute connect --target <name> --yes
   ```
   `--yes` stands for the person's yes, so never pass it before they gave one. If it asks for a
   browser approval, tell the person to approve it. A failed step is rolled back; show them the error.
   If the cluster access is read-only or refused, say what is missing: rights to install cluster-wide
   (a namespace, a Secret, a DaemonSet and cluster roles; in practice cluster-admin). Someone who has
   them runs connect.
4. Check: `probe compute clusters` lists the cluster, and `probe compute nodes` lists each machine
   once it reports.

If something looks wrong, stop and tell the person what you saw. Never fix it by hand: no commands
inside system pods, no relay or key of your own, no changes to the installed release.

Never uninstall. An admin removes Probe from a cluster on the dashboard: tell the person that, and
never run `probe compute disconnect`, `helm uninstall` or any delete yourself. Never print or paste
an enrollment token; connect gets its own. Options: `probe compute connect --help`.
