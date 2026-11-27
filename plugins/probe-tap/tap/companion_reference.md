# Reference

Lookup only; judgment is in the skill. Each command's flags are in `probe <command> --help`.

## INDEX

| section | holds |
|---|---|
| §1 ENTITIES | what each entity is for |
| §2 QUESTION AND TAGS | question, conclusion and tag shapes |
| §3 RUN | snapshot inputs, capture calls, delivery (the outbox) |
| §4 FILES AND NUMBERS | artifact commands, big files, names and versions, metric shape, derived metrics |
| §5 PAPERS | what goes in each paper field |
| §6 NOTES, §7 READING BACK, §8 PUBLISHING | notes, reads (RL samples too), experiment versions |
| §9 RECIPES | data processing, provisioning, trials |

## 1. ENTITIES

Each has a `probe <entity>` command group.

| entity | for |
|---|---|
| project | one effort; flat, linked to others by reference; a repo attached with `project code attach` |
| experiment | one question under a project; holds runs; can be forked, frozen and versioned (§8) |
| run | one execution; a relaunch is its child |
| group | a sweep's runs inside an experiment |
| trial | one rollout of a run (§9) |
| workspace | a set of projects |
| Shared folder | older team files: read and delete only (`probe shared`) |
| dataset | a named, versioned team dataset: `probe dataset add` makes it or its next version, and stores the bytes in Probe only when the researcher asks. Deleted in the dashboard only, refused while runs use it |
| source | where outside data came from (Hub id, bucket path): `source add`; deletion refused while runs use it |
| edge | a lineage link |
| decision | a recorded choice |

`probe experiment set` is the only way to change a question: opening a run
with `question=` on an existing experiment never rewrites it. It also sets the
conclusion; an empty conclusion clears it.

## 2. QUESTION AND TAGS

| field | shape |
|---|---|
| question | one testable question, ≤30 words: the change and what it might move. Never the outcome you expect |
| conclusion | the answer in plain words: "weight decay improves dev F1"; "dead end: runs out of GPU memory". Never a plan or a guess |

Explain acronyms. Use product or milestone names only when given - never
invent or expand a codename. Checkpoints, paths, commands and parameters go in
config or notes.

**Tags** at creation, 1-3 lowercase-kebab from a short vocabulary: `baseline`,
`ablation`, `sweep`, `debug`, `smoke-test`, `prod-candidate`, `infra`. Retag
when the meaning changes.

## 3. RUN

### Snapshot inputs - the decision record

`probe snapshot` captures a run's code and the inputs you include (paths or
globs); `probe snapshot-show` lists what was captured; `probe snapshot-restore
--verify-only` (verify only: it writes nothing) says whether it can be rebuilt: want "0 unavailable".

- A glob matching nothing is an error; a path outside the snapshot root is
  refused; a file already in the manifest is not added twice.
- Files over the snapshot's threshold (100 MB by default; not the 5 GB
  artifact limit) are recorded, not stored.
- A non-git directory is captured whole, skipping lockfile-rebuilt trees and
  credential-shaped names.
- Files git cannot supply are stored one artifact row per file
  (`snapshot-show` labels those `captured`), or as the run's `code-bytes`
  archive with `PROBE_CODE_STORAGE=archive`. Restore reads both.

The decision record is an artifact on the run, `inputs-decision.json` of kind
`inputs_decision`:

```json
{
  "included": [
    {"path": "data/train.jsonl", "why": "training set; regenerating is not deterministic"},
    {"path": "checkpoints/base.pt", "why": "base weights; referenced, 4.2GB on gpu-node-7"}
  ],
  "excluded": [
    {"path": "data/cache/", "why": "regenerated from train.jsonl on first epoch"},
    {"path": ".env", "why": "credentials; run needs HF_TOKEN, value not recorded"}
  ],
  "env_vars_that_matter": ["HF_TOKEN", "CUDA_VISIBLE_DEVICES"]
}
```

### Capture calls

Use the surface the run was opened with.

| | in the script (SDK) | from a shell (CLI) |
|---|---|---|
| metrics | `run.log({"loss": l}, step=i)` | `probe log` |
| per-device / per-actor | `run.log_hw({"gpu_temp": 88}, device=3)` | `probe log`, with the dimension |
| nested structure | `with run.span("trial", ...) as s:`, `run.step(i)` | `probe span add` |
| outputs | `run.log_artifact("ckpt", path=...)` | `probe artifact add` |
| external ids | `run.link(wandb_run_id="abc")` | `probe link` |
| a run FROM a run | `run.child(relation="retry")` | `probe exec` with the parent and relation |
| computed metrics | `run.log_derived_series("eval/auc", pts, producer="…")` | `probe log`, marked derived, with its producer |
| expression views | `run.create_view("loss_ratio", spec)` | `probe views create` |

**Only the SDK runs inside the training loop**, so `step=` is a real curve
there and scattered points anywhere else. Omitting `step=` auto-increments per
metric kind; `step=None` puts the point on the wall-clock axis. `run.log()`
takes any type: numbers plot; strings, dicts, lists and None are stored on the
step and read back under `view="trajectory"`.

Use the `with` form of `run.span`: both timestamps off one clock, it nests
what opens inside it, and it closes `failed` if the body raises.
`instrument-code` covers metric-vs-span and the leak rule.

### Delivery - what queues

`probe log`, `probe span add` and a run's `probe artifact add` queue into a
durable local outbox and return at once; a project or experiment upload blocks
unless it is over 64 MB or you ask for async.

- `probe outbox status` exits 0 when everything is delivered; MCP reads lag
  the same way.
- **`probe run end` is the barrier**: it waits for the queued data, or queues
  the close behind it.
- **Failures surface** as a stderr `outbox:` banner and in `probe outbox
  status` / `probe doctor`. `probe outbox retry` requeues dead-lettered items.

Relaunch, rewind and fork: `instrument-code`, RELAUNCHING.

## 4. FILES AND NUMBERS

### Artifact commands

An artifact is a name with immutable versions, each pinned from an upload.

| what | command |
|---|---|
| upload an output, to a run, experiment or project | `probe artifact add` |
| list a run's outputs | `probe artifact list` |
| browse one folder at a time | `probe artifact tree` |
| download a pinned version | `probe artifact download` |
| read the version chain | `probe artifact versions` |
| pin a new version | `probe artifact version-add` |
| who depends on this | `probe artifact pin-impact` |
| which run wrote it, which runs read it | `entity(refs=["artifact:ID"], view="lineage")`; the SDK records both for the runs it watches |

### Big files

Up to 5 GB, upload the file (over 64 MB it goes up in parts). Over 5 GB,
record a pointer instead:

- bytes on this box or a shared volume: a reference, a `file://` uri plus path
  and host. Say where it lives in the run's notes. Fingerprinting it reads it
  whole (minutes on tens of GB); a path this host cannot see needs an explicit
  allow.
- bytes already in a bucket: the object uri, resolvable by anyone with the
  bucket.

Kind, step, span and meta apply to a run's artifacts only. Code is always
stored, never pointed at (kind `code`, or the snapshot's `code_bytes`).
Uploads are content-addressed: bytes already held are never re-sent. From the
SDK, `run.log_artifact(name, path=…, reference=True)` records where the bytes
are without moving them.

### Artifact names and versions

**An artifact's name is the file's relative path; its extension is kept** (name
`ckpt` on `ckpt-4000.pt` stores `ckpt.pt`) so the dashboard can preview it. Never put a
description in a name. Files carry no notes: say what a file is, and where it
lives, in its run's notes (a project's file: the project's).

**Before creating an artifact, check `entity(refs=["artifact:<name>"],
view="versions")`:**

| resolved to | do |
|---|---|
| an exact compatible version | download it; record consumption, do not copy into a new identity |
| same purpose, content must change | produce it, upload, then `version-add` to the SAME artifact |
| nothing compatible | `artifact add` - a new artifact and its first version. Say why in the experiment's notes |

For datasets, pin provenance in the version meta: input asset versions, the
transform script version, parameters, schema, output content hash.

### Metric shape - decide before you log

A series is a key plus a dimension combination. Every distinct combination is
a separate series, and a one-point series renders as a scalar tile, not a
chart.

| shape | how to log it | renders as |
|---|---|---|
| curve (loss, lr, reward over time) | one series, many points, differing in `step=` | a line chart |
| headline scalar (final accuracy) | one series, one point, 0-2 low-cardinality dims | a stat tile |
| breakdown (accuracy by category) | one series per category value, cardinality under ~20 | N tiles, or a grouped read |

**Dimensions are the LOW-CARDINALITY axes you group by** — split, seed, rank,
category, never an identifier. Series ≈ the product of your dimension
cardinalities; past ~50 you have a wall of tiles.

**A labeled point is never plotted.** Charts read only unlabeled points;
labels make a point addressable in per-sample views. So the curve and the
per-sample record are two writes under two keys:

```python
for i, trial in enumerate(trials):
    run.log({"reward": trial.reward}, step=i)                    # THE CURVE: no labels
    run.log({"reward_sample": trial.reward}, step=i,             # the per-sample record
            labels={"instance_id": trial.id, "repo": trial.repo})
```

One key logged both ways draws a silent subset. Keep the headline ONE series
(`accuracy`, with `accuracy_per_example` under its own `kind=`) and declare
`agg="mean"` at the write.

**After the FIRST run of any new logging code, assert the shape** — a bad
shape is silent, since every call succeeds:

```python
series = client.run_series(run.id)      # one row per (key, kind, dimensions)
assert len(series) < 50, f"{len(series)} series — a wall of tiles, not graphs"
plottable = [s for s in series if s["point_count"] > 1 and not s["has_labeled_points"]]
assert plottable, "every point is labeled or single — nothing will draw"
```

Run BOTH: moving an id from `dimensions` to `labels` fixes the first and trips
the second. Do it on a 2-instance run, not after 300. For a breakdown,
`metrics(mode="grouped")` with a large `step_bucket` should return rows with
`n > 1`.

### Derived metrics and expression views

Two doors for a metric you thought of after the run finished:

- **Derived metrics** - computed in Python, pushed as points:
    `run.log_derived_series(key, points, producer=…)` for a curve,
    `run.log_derived({...}, step=…, producer=…)` for one step. `producer` and
    `step` are required; a backfill lands on existing steps.
- **Expression views** - a formula over existing series, evaluated at read
    time so it stays right as a live run advances. `probe.expr` or a spec file
    (`probe views create`). Preview before you create: a spec naming an
    unlogged series returns `missing_inputs` instead of an empty panel.

## 5. PAPERS

`probe paper add` on the project it serves; then `list`, `update`, `tag`,
`remove`, `edges`. What goes in each field:

- source (required): a url or local path.
- relevance: why it matters to THIS project, one line of at most 15 words;
  left out, Probe writes one.
- authors: free text, "Vaswani et al."; repo: the repository read alongside
  the paper.
- summary: the main idea in YOUR words. discrepancies: what the repo does that
  the paper does not say (censored code, a missing ablation, a contradicted
  hyperparameter). Both can be read from a file.
- tags: one concept each - the axis a forty-paper review gets grouped on.
- via: the paper you followed to this one, or none; leave it out only if you
  truly can't say. With a paper: one sentence on why it led here, and how you
  know: a tool call handed it to you (`find_papers` similar, references or
  citers), you took it off a reference list, someone told you, or you worked
  it out afterwards.
- relates to: an earlier paper in this project, or none. With a paper: a
  reason (required), and whether it is your judgment (the default) or the
  researcher said so.

Amend, never re-add:

- `paper update` leaves omitted fields alone; an empty value clears any field
  but title and source (a cleared relevance is rewritten by Probe).
- `paper tag` adds, removes or replaces concepts.
- `paper list` reads newest first; filtering by several tags requires ALL of
  them. `paper edges` reads the via chain and the relates-to links.
- Adding counts as project activity; editing and removing do not.

## 6. NOTES

Projects, experiments and runs, plus ONE team note; files, sweep groups and
trials have none. `probe notes checkout` -> edit the file -> `probe notes push`. The
`edit-notes` skill is the manual: caps, merging, sub-notes, compaction. The
CLI is the only writer; a script shells out to it.

## 7. READING BACK

**`metrics` (`probe mcp call metrics`) has three modes**; each refuses the others' arguments, so a
refusal means re-issue in the right mode.

| mode | reads |
|---|---|
| `grouped` | reduces a key |
| `coordinates` | enumerates the axes a run logged on — call it first when guessing at `by` |
| `points` | raw points, a page at a time |

`entity`'s own description (`probe mcp tools`) lists every view and `card` returns
`available_views` - ask the tool, never a memorised table. Take the narrowest
view that answers the question; `view_options` narrows server-side. A
`partial` read is not the whole record: name what you did not see next to the
finding it qualifies.

To trace a path, URI, artifact id or content hash: `search` with it
as the query (exact match), then `entity(view="lineage")` on the run that owns
the hit. No hit means unknown provenance, not none - say so.

### RL samples

An RL run's attempts (rollouts, trajectories) are SAMPLES, recorded by the
trainer's adapter (`instrument-code`). Each has a task and its scenario (the
outside values it was posed under: `scenario.set`, `scenario.as_of`), a reward
and its parts, the weights that made it, the step that trained on it, and a
result: `passed` (reward at or above the full reward), `failed` (graded below
it), `error` (never graded: an engine abort, a sandbox, a verifier, a timeout).
Means leave `error` samples out and count them beside. By default a read counts
what the model trained on, at the step that trained on it; it can add dropped
groups, read eval passes, or group by the step that generated them.

| question | command |
|---|---|
| did every sample arrive (yes or no, with reasons) | `probe samples capture RUN` |
| mean reward per step | `probe samples stats RUN --by step --measure mean_reward,count` (also `min`, `max`, `pass_rate`) |
| the misses at a step on one scenario | `probe samples list RUN --step N --reward 0 --where scenario.set=P2` |
| one task at step 0 beside the best checkpoint | `probe samples compare --task T --at RUN@step:0 --at "RUN@checkpoint:best(eval/val)"` |
| pass rate per template over steps; groups with no reward variance | `probe samples stats RUN --by step,scenario.template --measure pass_rate`; `--measure zero_variance_share` |
| did async staleness hurt | `probe samples stats RUN --by staleness_max --measure mean_reward,count` |
| the best checkpoint, with its base model, data, config, commit and parent | `probe checkpoint best RUN --metric eval/val` (highest wins: `--mode min` for a loss; `--from-evals` when the score is on the eval runs) |
| a run's tasks; one task, its samples, its reward at every step | `probe samples tasks RUN`; `probe samples task RUN TASK` |
| the result mix per step; where errors come from | `probe samples stats RUN --by step,result`; `--by host --where result=error` |
| what broke; one sample whole, its log, its tokens | `probe samples list RUN --result error`; `probe samples show`, `logs`, `tokens` |
| text in responses, turns or logs | `probe samples list RUN --grep "Traceback"` |
| attempts still running but gone quiet | `probe samples list RUN --outcome in_flight --stale-after 10m` |
| weight versions and their checkpoints | `probe samples weights RUN` |
| runs side by side | `probe samples stats RUN --run B --run C --by step` |

`probe samples export` writes samples whole, turns included, as JSONL or as
ATIF trajectories (Harbor's shape), and can record the output as a dataset's
next version made by the run (an RFT-SFT set: only the passed training
samples, `--result passed --split train`; check the count before recording it). Text output is cut at 4 KiB
except `show`.

## 8. PUBLISHING

Only when the researcher explicitly asks to mark, publish or approve. What
gets published is an immutable **experiment version** (a manifest of its runs)
plus results pinned as artifact versions. There is no run-level "official"
flag; never invent one.

1. `entity(view="reproduce")` on each candidate run: check question and config, that `env_ref` resolves (`missing: ["execution_record"]` = no snapshot, not reproducible), and that `code.manifest.n_pending_upload` is zero.
2. `entity(view="versions")` on the experiment: if a version already covers this set, do not mint a second.
3. Present the exact experiment + asset versions and get explicit approval for that set. Metrics and exit status are not approval.
4. `probe artifact version-add` per approved asset, then mint it with `probe version create` and a label, and report the version.

Never decide on a `partial` view - §7.

## 9. RECIPES

### Data processing steps as runs

One project-direct run per script or stage VERSION, opened by `probe exec` on
the project with a deterministic external id (`clean-structures-v2`); then
`probe run inputs RUN` shows what it read and where those bytes are stored.

The run records the files it reads and the ones it writes in its folder. Probe
links nothing from them: link what it used (`probe dataset use`) and register
what it wrote (`probe dataset add`). A FAILED step resumes on retry with the same id: the same run,
reopened, so no `retried_from`. A COMPLETED one refuses it - bump the version. Thresholds chosen and rows deleted go in the run's notes:
deletions are provenance, not housekeeping.

### Provisioning

The machine a training run got goes on THAT run, as `probe link` keys
(`gcp_zone`, `gcp_machine_type`) in its `foreign_keys`.

**`foreign_keys` is ONLY for what the lineage vocabulary cannot say**
(`consumes` `produces` `evaluates_on` `forked_from` `resumed_from`
`retried_from` `branched_from` `promoted_to` `derived_from` `supersedes`
`informed_by`) - a relaunch after a crash is a retry child of the run, not a key.

### Trials

A TRIAL is one rollout, addressed by its rollout span id - the last segment of
a `/runs/<run>/trials/<id>` link. `probe trial list` is the authored
inventory, not `probe span list`: the span is what the producer emitted, the
trial is the row beside it.

**Its name resolves through a chain**, and only the first link is yours:
a name set with `probe trial set` (or a generated title) -> the rollout span's
generated description -> the span name the script passed -> `Unnamed trial`.
The generated title lands once the run goes terminal, from the span's
`attributes`, so at instrumentation time the lever is a meaningful `name=` and
populated `attributes`. A name you set is sticky: generation and the
producer's retries both leave it alone. Name and description are a trial's
only authored fields; it has no notes.
