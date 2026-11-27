---
name: instrument-code
description: "Wires the Probe SDK (`probe.init`, `probe.log`) into a script that trains, evaluates, sweeps or serves, so that it records run-level metrics, and an RL trainer (Miles, ART, SkyRL, verl, TRL) or a Harbor job, so that it records every rollout as a sample. Use when writing or changing any such script, launching a training script on another machine, when a trainer already has its own integration (Miles, ART, SkyRL, verl, TRL, Harbor, Ray), when an agent or training or eval code in any other language (TypeScript, Rust, Go, ...) should record its runs or agent calls, and when a run finished green but recorded nothing, less than expected, or onto the wrong run. HARD TRIGGER - ENSURE YOU READ BEFORE ANY PAID GPU HOURS. Also when a job goes through the team's own job tool (a submit command, a queue, a cluster): wire Probe into that tool once so every job records, whoever starts it."
---
# Instrument code

Put the SDK in scripts you write. Use `probe exec` only for code you can't edit,
or a launcher that only submits the job. Both open the run. Each command's
options are in `probe <command> --help`.

No Probe SDK for this language (Python has one): read `other-languages.md`, build a small client, run `probe client check`.

```python
import probe
probe.init(project="folding", description="DockQ sweep over sampling temperature", tags=["dockq"],
           config=args, intent="does a lower sampling temperature improve DockQ?", run_type="sft")
# Every run needs a project, named by its slug or id (a name never finds one); a new slug,
# lowercase words joined by "-", creates it. A run naming none raises. In the `daemon` state,
# on this machine pass project=probe.FROM_ENV (`probe exec` has the same switch): the run goes
# to this session's project, which the daemon sets (`probe session home show` prints it); before it
# is set, probe.init() waits for it (up to 90 s). FROM_ENV also reads PROBE_PROJECT and this
# computer's default project (an AttributeError means that environment's probe-research is
# older: upgrade it). On another machine pass a slug; the daemon may move the run.
# `on` state: name the experiment you created too:
# probe.init(project="folding", experiment="dockq-sweep", question="does a lower sampling temperature improve DockQ?")
probe.log({"loss": 0.42}, step=42)   # ambient: no handle to thread through call frames
probe.finish()
```

Always pass `run_type`: `sft` for supervised fine-tuning, `rl` for RL training or an agent eval, `general` for anything else. A run without it reads as Other: Probe never guesses.

Pass the run's whole config object as is (`args`, a Hydra `DictConfig` never resolved, a dataclass, a dict), not
picked keys, and no credentials. A trainer integration that records config needs none.

## PIN THE RUN'S KEY METRICS:

Pinned metrics are the charts a run's page shows first, one list per project per run type. Pass the names your
script logs for these, in this order, with `run_type` (`[kind:]key` each, at most 12; a key holding a colon is
`model:<key>`):

- `sft`: loss, grad norm, learning rate, eval loss
- `rl`: train reward, eval reward, entropy, error rate
- `general`: loss, eval loss, reward, kl

```python
probe.init(project="folding", run_type="sft",
           key_metrics=["train/loss", "train/grad_norm", "train/learning_rate", "eval/loss"])
```

It fills the project's list for that type unless a person saved one (theirs wins). Other languages:
`PUT /v1/runs/{run_ref}/key-metrics` (`other-languages.md`).

## FILES AND LOGS ARE CAPTURED FOR YOU:

When a run ends (`finish()`, a crash, exit, or a hard death: segfault, OOM killer, SIGTERM) the SDK saves, on any
machine:

- every file the run CREATED OR CHANGED in its working folder, as `outputs/<path>`; `.git`, `.venv`,
  `node_modules`, caches, credential-shaped names (`.env`, `*.pem`) and, in a home folder, every dot-entry are skipped
- everything it PRINTED from `import probe` to exit, C-level output too, as the run's Logs (first 2 MB + last
  8 MB). Under `probe exec` the child's output is teed the same way, and `probe exec torchrun` interleaves its
  local ranks in one log; other ranks keep their own. A notebook's output is not kept.

Read what a run printed with `probe run logs`; read it again for newer output.

A close uploads files up to 64 MB and spends at most 120 s inspecting them (`PROBE_CAPTURE_BUDGET_SEC`). With
big-file capture on for the team, files up to 5 GB go up in parts after the run ends; only their first 16 MB is
checked for credentials, so keep secrets out of the files a run writes. Any other file past a
limit becomes a pointer (path and size, no bytes), which lasts only on lasting storage (a network drive, a
Modal Volume, your machine). On a throwaway disk (container, pod, CI) it is lost, so write checkpoints there
to a bucket or mounted volume.

Still call `probe.log_artifact` for:

| case | why |
|---|---|
| a file OUTSIDE the run's folder | capture only sees that folder |
| a name or kind you choose | captured files are named by path and never filed `checkpoint`; `log_artifact` files `.ckpt`/`.safetensors`/`.pt`/`.pth` as `checkpoint` |
| a checkpoint written straight to a bucket | `log_artifact(name, uri="s3://...")` records where it is |

Runs that SHARE a folder at the same time (sweep workers, seeds) cannot tell whose files are whose: capture
skips them and warns. Give each its own: `probe.init(..., outputs=f"results/{seed}")` (`probe exec` too).

Opt out with `probe.init(capture_outputs=False)` or `PROBE_CAPTURE_OUTPUTS=0` (`probe exec` too). To keep some
paths out (datasets, checkpoints), list them in a `.probeignore` at the git toplevel (gitignore syntax) or
`PROBE_IGNORE`: it covers code, output and read capture. `PROBE_CAPTURE_LOG=0` drops the log and, with it,
capture after a hard death. Nothing survives a death that takes the whole container or job step (the program
as PID 1): write to lasting storage there.

On SIGTERM (preemption, `docker stop`, a deleted pod) the SDK spends up to `PROBE_SIGTERM_FLUSH_SECONDS` (20 s)
sending what is queued and closes the run `failed` (reason `preempted`); a `kill` from your own process on the
machine closes it `canceled` (reason `stopped`). A SIGTERM handler installed before `probe.init()` runs first.
`log()` queues to local disk; rows go out every `PROBE_SEND_INTERVAL_SEC` (5 s), so the page trails by that much.

## UPLOAD ONLY WHAT A RERUN NEEDS:

Before the first run in a folder, write a `.probeignore` there (gitignore syntax; `!` re-includes) so the code
capture holds what a fresh sandbox needs to rerun it: the code it executes, its configs, its build and
dependency files (lockfiles, `pyproject.toml`, `Cargo.toml`). Leave out what those rebuild (build output, venvs,
caches), data and checkpoints (link them instead), other projects and earlier outputs. Update it when the run
starts using something new.

## THE ONE RULE:

**A process that emits telemetry must itself be configured to emit it.** A parent's config never reaches a
child, and neither does `probe.init()`'s ambient binding.

```
launcher process          config lives here
  └── trainer subprocess         ...but not here
        └── Ray driver                 ...nor here
              └── Ray actor  <-- logs the metrics, knows nothing
```

A flag set on `args` in the Ray driver ARRIVES in the actors (`args` pickles) while the registry entry does
not: every check passes, nothing initialises.

### FIXES, IN ORDER:
1. Configure inside the emitting process: the actor's `__init__`, or Ray's `worker_process_setup_hook`
2. Pass config through what crosses the boundary: `runtime_env["env_vars"]`; Ray workers do NOT inherit the submitter's environment
3. Assert it landed where it is READ, not where you set it
4. `import probe` needs `probe-research` in THAT process's environment (`uv add probe-research`). The wizard's CLI install is isolated and NOT importable; `probe` and `probe-agent` on PyPI are unrelated packages

The RL launchers (below) do 1 and 2 for you. 3 stays yours: check the capture on a short run.

## SWITCHING OFF W&B:

A script that logs to W&B (`import wandb`) moving to Probe: follow `instrument-code/from-wandb.md` (beside this
file): the order (both for one run, check parity, then remove W&B) and the call map.

## RL TRAINERS AND HARBOR:

Each records the trainer's metrics and every attempt (rollout, trajectory) as a SAMPLE: task, scenario, reward,
the weights that made it, the step that trained on it, passed / failed / error. Before launching, read your
framework's section of `instrument-code/rl-frameworks.md` (beside this file).

| framework | the door |
|---|---|
| Miles (upstream and the July fork) | `python -m probe.connectors.miles.launch train_async.py ...` + `--custom-rollout-log-function-path probe.connectors.miles.log_samples` |
| ART | `probe.integrations.art.track(model)`, `WANDB_API_KEY` unset |
| SkyRL | `trainer.override_entrypoint=probe.integrations.skyrl.entrypoint` on `skyrl.train.entrypoints.main_base` |
| verl | `python -m probe.integrations.verl.launch -- python3 -m verl.trainer.main_ppo ...` |
| TRL | `report_to="probe"` + `probe.integrations.trl.track(trainer)` |
| Harbor | `harbor run ... --plugin probe --pk run=RUN` (`probe-research[cli,harbor]`) |

A per-sample number of an RL attempt belongs to its sample, never a labeled metric point. A custom loop that
records rollouts with `run.samples` is marked `rl` on its own. Prove the capture on a two-step run:
`probe samples capture` says yes, or no with reasons, and failed samples carry why (`rl-frameworks.md`, CHECK
THE CAPTURE). While it trains, `probe samples live` shows running, slow, hung or failing samples; cancel or
retry one where its framework allows (`rl-frameworks.md`, LIVE; no adapter: LIVE IN YOUR OWN HARNESS).

## WHAT MUST REACH THE JOB:

Three things, none of which travel on their own: the PACKAGE, the run id + epoch, and a TOKEN. `probe exec
--token-ttl <the job's expected length plus a margin>` exports `PROBE_RUN_ID`, `PROBE_RUN_EPOCH` and a
`PROBE_TOKEN` that records that one run only into its child, but that export dies at the machine boundary:

| Launcher | id + epoch + token | the package |
|---|---|---|
| Modal | `secrets=[modal.Secret.from_local_environ(["PROBE_RUN_ID", "PROBE_RUN_EPOCH", "PROBE_TOKEN"])]` (never a literal: the snapshot uploads this script) | `modal.Image.debian_slim().pip_install("probe-research")` |
| Ray | `runtime_env={"env_vars": {...}}`, the same three (Ray's dashboard shows them: only ever a run's own token) | `runtime_env={"pip": ["probe-research"]}` |
| Slurm | `sbatch --export=ALL,PROBE_TOKEN,PROBE_RUN_ID=...,PROBE_RUN_EPOCH=...` | the venv the job activates |
| SkyPilot | `probe exec`, naming the project, around `sky jobs launch ...` (or `sky launch`, `sky exec`) adds all three; a plain `sky jobs launch --env PROBE_PROJECT=<slug>` gets them from the team's Probe policy | `setup: pip install probe-research` |
| Docker / k8s | `-e PROBE_TOKEN -e PROBE_RUN_ID=...` / `env:` on the pod | the image |

With them set (the token alone is enough), `probe.init(config=args)` joins the launcher's run. A `project` /
`experiment` naming that run's own home (or `probe.FROM_ENV`) is accepted; a different one raises, and `name` /
`question` are ignored with a warning. If the launcher only submitted the job, `probe exec` opened the run
AWAITING ATTACH and this call takes ownership.

THE TEAM'S OWN LAUNCHER: when jobs start through a team tool (a submit command, a queue, a shared entrypoint or
image), wire Probe into that shared code and commit it to the team's repo; your own job's files record only
your jobs. The launcher wraps each submit in `probe exec` as a detached launcher, naming the project (a
teammate's shell has no session project), and passes `PROBE_RUN_ID`, `PROBE_RUN_EPOCH` and `PROBE_TOKEN` into
the job. A job that tests a question also names its experiment, taken from that submit (never fixed in the
launcher) and already existing (`probe exec` never creates one); an experiment named only in the script never
applies. One run per job: ranks of one job share it; an array or several seeds need one each. The launcher
first runs `probe whoami`; where that fails it submits the plain way with `PROBE_MODE=disabled` and says so in
one line. If `probe exec` itself fails, show the error and never resubmit: the job may be queued. Then launch
one job and check its run appears.

The credential is `PROBE_TOKEN` (plus `PROBE_BASE_URL` off the default server); there is NO `PROBE_API_KEY`.
Give the job the run's own token from `probe exec --token-ttl`. Near expiry on a live run Probe alerts you; `probe run
token <run> --extend 7d` extends it (without `--extend` it prints a new token instead). Never copy your Probe login (`config.json`) to another machine or a secret: every job would write as
the researcher. Only a job that opens runs of its own (a sweep controller, an array, a queue nobody is signed in
to) needs a token that can read and write but not delete: `probe token create --scope read --scope write`, which
the researcher runs in their own terminal and keeps in the platform's secret store. Recording from the launcher after the job ends loses its curves, logs and
data reads: say so before offering it.

On Kubernetes the outbox and its worker die with the pod. Put `PROBE_OUTBOX_DIR` on a persistent volume, set
`PROBE_FINISH_TIMEOUT_SEC=600`, a `terminationGracePeriodSeconds` at least that long, and raise
`PROBE_SIGTERM_FLUSH_SECONDS` with it.

Without the EPOCH a stale process holding an old run id attaches to a row a newer attempt reopened. Several
ranks on one id is fine: the first reopens, the rest join.

## WHICH RUN ARE YOU WRITING TO:

PyTorch Lightning: `Trainer(logger=ProbeLogger(experiment=..., name=...))` from `probe.integrations.lightning`
(`probe-research[lightning]`). Hugging Face: `import probe.integrations.huggingface`, then
`TrainingArguments(report_to="probe")`, or `callbacks=[ProbeCallback(experiment=..., name=...)]` to choose the
run (`probe-research[huggingface]`; one or the other). Both adopt a run `probe.init()` opened, log from rank 0
only, open one run per Trainer, and never finish it mid-script (exit or `probe.finish()` does).

Both record each checkpoint's LOCATION; one on a throwaway disk warns. To store the files pass
`checkpoints="upload"` (`ProbeCallback` / `ProbeLogger`) or set `PROBE_CHECKPOINTS=upload` with
`report_to="probe"` (rank 0's view: shards on other nodes' disks are not gathered). Get one back with
`probe artifact download` (a run's best or latest checkpoint) or `client.checkpoint(run_id, "best")`,
`client.download_checkpoint(row, dir)`.

Ray Tune: one run per trial, opened inside the trainable. A function trainable's run closes itself `completed`.
A `tune.Trainable` class must call `probe.finish()` in `cleanup()`: Tune SIGKILLs 200 ms after its SIGTERM,
too soon for the exit hook, and the run stays `running` or reads `failed`.

A trainer integration (Miles, ART, SkyRL, verl, TRL, Harbor, Ray) may open its OWN run and ignore the id you
supply. Steer it with the knobs it reads (`PROBE_PROJECT`, experiment, run name, external id), then check where
it wrote:

- each RL adapter keys its run so a relaunch finds it (`rl-frameworks.md` names each key); every Ray worker's
  runtime env carries the `PROBE_RUN_ID` it joined
- a default project name in a run spec quietly collects runs (Miles: `miles`)
- an empty run proves where you looked, not what happened

## METRIC OR SPAN:

| Your work | Use |
|---|---|
| a flat loop repeating phases | metrics, one series per phase |
| nested work of varying structure (trial -> agent turns -> verifier) | spans |
| one value per step | `probe.log({...}, step=i)` |
| one final number (test accuracy) | `probe.log({"test/acc": acc})` once: a single number, not a chart |
| a value per sample | LABELS, never dimensions (per-sample ids shatter a series into one-point tiles). Labeled points NEVER plot, so the curve needs a second, unlabeled key |
| an RL attempt and its numbers | a SAMPLE: the trainer's adapter, or `run.samples.log` (RL TRAINERS AND HARBOR) |

## GRADES:

A grade of ONE run's results (a checkpoint's score, an eval job's outputs graded later) is a metric on THAT run,
not a new run. Log it as derived, so the grader stays on record; it works on a finished run. The grading script
logs it on the graded run's id (not `probe.init()`, which opens a run of its own):

    with probe.Client() as client:
        run = probe.Run(client, client.get_run(RUN_ID))
        run.log_derived({"grade/<rubric>/<key>": value}, step=N, producer="grade.py")

From a shell: `probe log RUN grade/<rubric>/<key>=<value> --derived --producer <grader> --step N`.

`N` is the graded checkpoint's step (0 for a job with no steps). Work that reads several runs' results (a
comparison, a leaderboard) is its own run, linked to each by a `derived_from` edge (`probe edge add`).

## SPANS:

`spans: 0` on a training run is usually correct.

Spans have no heartbeat. In the two-call form (`run.span(...)`, then a second call with `id=`), one opened
before the work leaks on every raise: record it AFTER, with real timestamps. `with probe.span(...)` closes on
every path (`failed` when an exception escapes).

`span_type="rollout"` is the only span that becomes an entity: a TRIAL keyed by the span's id. Pass a real
`name=` and rich `attributes`: the trial's title is generated FROM them once the run ends.

Per-task eval results are one rollout span per task, its scores in `attributes`, its output files linked to it.
From a shell, `probe span add` prints the span id; attach each output to it with `probe artifact add`.

```python
with probe.span("rollout", name=task.id, external_key=task.id) as span:
    out = solve(task)
    span.attributes.update(score=out.score, unfinished_reason=out.reason)
    probe.log_artifact(f"eval/{task.id}/output.png", path=out.image, span_id=span)
```

A run that writes its results to files attaches them too (`probe.log_artifact`, never secrets), so a pane
(`write-pane`) can show what built-in charts and tables can't.

## NAME WHAT A CRASH WAS PROCESSING:

Wrap each batch or sample so a crash email names what it died on. It sends nothing and changes no metric:

```python
for epoch in range(epochs):
    for i, batch in enumerate(loader):
        with probe.context(batch_id=i, epoch=epoch):   # or sample_id= / task_id=
            train_step(batch)
```

Keys: `batch_id`, `sample_id`, `task_id`, `prompt_id`, `epoch`, `split`, `phase` (ints or strings) and `step`
(non-negative int); others are dropped. Never `yield` inside the block. Never `run.unit(labels=/coords=)` for
this: it stamps every `log()` inside and the curve stops plotting.

## DECLARE WHERE A METRIC MUST STAY (OPTIONAL):

Only for a range you are SURE of, declared after `probe.init()` and before the loop; the run's creator is emailed
the first time a value leaves it. A wrong range emails someone about a healthy run; NaN and inf are reported
anyway.

```python
probe.expect({"val/acc": (0.5, 1.0), "train/loss": (None, 20)})  # None = open end
```

For a run already going: `probe run expect`.

## HARBOR TRIALS - THE FILE DOOR:

Harbor writes each trial to a directory (`result.json`, `agent/trajectory.json`, verifier logs). Probe captures
it into the run, keyed to the training step; no code in the agent. A whole Harbor job is samples, one per
attempt: `harbor run ... --plugin probe` as it runs, or `probe trial add` on the job folder after
(`rl-frameworks.md`, HARBOR). `probe trial add` on one trial folder records a rollout span, reward metric, files
and manifest at the step you give; a recognised trajectory (ATIF v1.x) expands into turn and tool_call spans,
an unknown one is stored raw and `probe trial expand` expands it later.

With Miles, the trainer writes one export descriptor per trial and nothing uploads until something consumes it
(key it by the attempt's `TrialResult.id`: `rl-frameworks.md`, MILES). `probe trial stage` copies and checksums
a trial to a durable volume (no network; name the files it must hold with `--expect`, so an unfinished trial is
refused); `probe trial export` consumes one descriptor; `probe trial drain`
retries every unfinished one under a root (`--run "$PROBE_RUN_ID"` binds descriptors written before the run existed); `probe trial watch` keeps exporting as trials land (run it somewhere
that outlives the job); `probe trial reconcile` republishes the completeness manifest. A retry updates the same
rollout.

## DO NOT DUPLICATE, DO NOT BLOCK:

Capture what the trainer's integration CANNOT know: your reward function, scoring sandbox, dataset transform.
Re-emitting its timings, throughput, loss or config adds latency and two sources of truth.

- Never call HTTP synchronously inside a coroutine: it serialises every sample and timeouts overshoot. Use
  `asyncio.to_thread` or a queue
- `probe.log` dates each point at call time; pass `wall_clock=` only for an earlier event
- Prefer the SDK's on-disk queue to per-event network calls
- Cache handles per process, but cache only SUCCESS: a cached miss disables capture for the worker's life

## WHERE THE QUEUE LIVES:

Every recording process queues in its machine's outbox (`~/.local/state/probe/outbox`, or `PROBE_OUTBOX_DIR`),
never in `cwd`: Ray DELETES a job's `cwd`. A detached worker sends it, so a killed actor's rows still go. A queue
of your own follows the same rules: not under `cwd`, on a disk that outlives the job, drained at the end.

## VERIFY AT THE DESTINATION:

The exit code says the process ended, not what was stored.

```python
run = client.get_run(run_id)
assert run["counts"]["metrics"] > 0
assert len(client.run_series(run_id)) < 50   # else: a wall of one-point tiles
# on a box you are about to destroy, a file:// pointer resolves nowhere afterwards:
assert all(not a["uri"].startswith("file://") for a in client.list_run_artifacts(run_id))
```

Series count near point count means something high-cardinality became a dimension: fix it before spending more
compute. Print which outcome occurred: drained, written but unpublished, nothing written. An RL run: `probe samples capture` says yes, and `probe samples stats` per step matches the trainer's own
mean reward (`rl-frameworks.md`, CHECK THE CAPTURE). A file recorded by reference on a box you will destroy
resolves nowhere; code is always uploaded.

## WHAT THE RUN READ:

`probe.init()` (and `probe exec` for a Python child) records every file the run opens for reading (path +
sha256); the server links each to the run that WROTE those bytes. Its edges:

- It sees `open`, `pathlib`, numpy, pandas, `torch.load`, PIL, pickle, `pyarrow.parquet`, `pyarrow.memory_map`,
  `h5py`, `safetensors`; NOT other C readers (`pyarrow.dataset`, feather, sqlite). For those:
  `run.use_dataset("dataset:NAME@V")` or `run.use_checkpoint("checkpoint:RUN@best")`; for an artifact, a
  `consumes` edge from the run (`probe edge add`)
- `Client.run(...)` handles record only with `capture_reads=True`
- Opt out: `probe.init(capture_reads=False)` or `PROBE_CAPTURE_READS=0`. Written files are recorded too; off
  alone with `capture_outputs=False` or `PROBE_CAPTURE_WRITES=0`
- A read is evidence, not a link. A wrong link: `probe edge remove`, then `run.use_dataset` / `run.use_checkpoint`
- Where the Probe daemon records, `probe edge`, `dataset use` and `checkpoint use` are refused: send the link or
  fix to its writer: `probe ask --writer`

## RELAUNCHING A RUN:

Reusing an `external_id` conflicts with the incumbent; `on_conflict` says what the relaunch MEANS:

| Intent | Use | What it does |
|---|---|---|
| continue past a crash | `"auto"` (default) | resumes a dead incumbent in place; steps at or below the resume point refused |
| restart from an EARLIER checkpoint, tail discarded | `on_conflict=probe.Rewind(step=400_000)` (`probe run start` can rewind too) | hides the record from that step on, so the relaunch REWRITES it; the only policy that reopens a `completed` run |
| keep BOTH curves | `client.fork_run(source, step=400_000)` or `probe run fork <run> --step 400000` | a NEW run from that checkpoint; the source is untouched |
| repeat from scratch | `"supersede"` | a fresh `external_id-rN` row; the incumbent is tagged `superseded` |

To restart from a checkpoint with new settings in one step: `probe run resume` (or print its exports for
sbatch). The script loads `run.resume_checkpoint` (HF `resume_from_checkpoint=`, Lightning `ckpt_path=`); the HF
and Lightning integrations apply the new values after the load, a plain script applies `run.resume_overrides`
itself and calls `run.confirm_resume_overrides()`.

`Rewind` needs a server that declares it; the SDK refuses loudly on an older one. Never work around that by
deleting and re-uploading the run: upgrade, or fork.

Logging a run that already happened keeps its real times (ISO 8601 with an offset): `probe run start` and
`probe run end` take them, or `client.run(..., started_at=...)` then `run.set_status("completed", ended_at=...)`.

## TAKE COMMANDS WHILE IT RUNS:

Register what the script can be asked; `probe run send` asks it, and the handler runs at the next `run.log()`:

```python
from enum import StrEnum
from probe import CommandKind

run.on(CommandKind.CHECKPOINT, lambda: save_checkpoint())        # return the artifact id
run.on(CommandKind.CONFIG, lambda values: set_lr(values["lr"]))  # then recorded on run.config
run.on(CommandKind.STOP, lambda: trainer.request_stop())         # graceful; a forced stop signals the process
run.on(CommandKind.PAUSE)  # no handler: run.log() holds while paused, until resume or stop (log once per step)

class TrainerCommand(StrEnum):
    EVAL_NOW = "eval_now"

run.on(TrainerCommand.EVAL_NOW, lambda payload: evaluate(**payload))  # payload: the JSON args sent with it
```

A kind is an enum member, never a string. An unregistered ask is refused at once; `probe run commands` lists what
it accepts and what was asked. The HF Trainer callback handles checkpoint, config (`lr`), stop and pause (verl,
SkyRL, ART and Harbor pause too); with Lightning add `ProbeCommands()` (`probe.integrations.lightning_commands`)
beside `ProbeLogger`. A plain script on several processes calls `probe.commands.sync()` on every rank at the
same point (every N steps), or its asks are refused.
