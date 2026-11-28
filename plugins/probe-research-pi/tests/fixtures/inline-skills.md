# Skill: track-work

# Track work

You are the main agent, the researcher's coding agent, and you record your own
work. Check `probe session status` first: if it is `read` or `off`, write
nothing, say so once and carry on. Never move the switch (`probe session
track`); only the researcher does, with `/probe`. When you run code that opens no runs of its
own, `probe exec` opens its run itself, in the project or experiment it names: no `run start` first.

Each command's flags are in `probe <command> --help`; shapes and recipes are in `track-work/reference.md` (reference §N below). Probe's read tools (`entity`, `search`, `metrics`, `browse`, `query_sql`, `find_papers`) run as `probe mcp call <tool> '<json args>'`; `probe mcp tools` lists them.

## What counts as ML work

ML work is work on a model: training it, evaluating it, the data it learns
from, or the research behind it. What the researcher asks to record is
recorded, whatever the lists say.

ML work - record it:
- training, fine-tuning, RL, and their scripts, configs and launches
- evals, benchmarks, sweeps, ablations, reading model outputs
- prompt engineering: changing a prompt and checking what the model does
- datasets: building, cleaning, labeling, splitting, checking for leaks
- papers, lit reviews, method or architecture design
- analysing results: plots, notebooks, comparing runs
- fixing a bug in any of the above

Setup - note it, never log it:
- installs, drivers, CUDA, container images, provisioning, cluster or GPU
  access, and, in a repo that trains or evaluates models, its CI fixes and
  dependency bumps
- no project, run or artifact for it. Only what tripped you up (a pinned
  version, a driver fix, a stocked-out zone) gets one short line (`edit-notes`):
  in the notes of the project it serves once that project exists, or in the
  team note when it holds across projects.
- fetching a dataset or a base checkpoint is not setup: it is an input the run
  records when it reads it.

Not ML work - record nothing:
- product code that only calls a model (UI, API routes, CSS)
- CI, deploys, releases, dependency bumps of product code
- typos, dotfiles, product docs, tests unrelated to a model
- work on Probe's own code (its CLI, SDK, server or plugins) - using Probe in ML work is not this

A session can hold both: record the ML part only.

## Every write

- Everything a person reads (slugs, names, questions, summaries, reasons, tags, metric keys, notes) is short and plain, and leads with the most recent or most important thing. No hashes, uuids, timestamps, ticket numbers, command lines, parameter piles, bare counters or private abbreviations.
- Fields (names, descriptions, questions, conclusions, dead ends, reasons): a short ELI10 TL;DR in plain words. No results, settings, run names or formulas: the dashboard already shows them. Detail goes in notes. Digits that are part of a name stay (CIFAR-10, Llama 3 8B).
- Exact and machine-shaped: external ids, `probe link` keys, configs, specs, an artifact's name (its relative path), a paper's real title.
- Slug: 3 lowercase hyphenated words naming the thing, permanent. Name: a plain title, always set, never the slug re-cased.
- A name or description the researcher chose for a project, experiment or run: write it verbatim with `--authored-by human`, in a command of its own (never with words you wrote).
- Read an entity's notes and Markdown before writing to it.

## Where things go

Put each fact on the lowest entity it is true of. Tracking on means everything is recorded; do not curate. Drop nothing: a file with no better home goes to the project's artifacts. Notes are the exception: only what would help the next agent or teammate, as hidden prose written by `edit-notes`.

- `document`, the researcher's block on a summary, is theirs - never write it; the rest of a summary is yours (write-summary).
- The team note is one synced file, `probe-team-note.md` (`edit-notes`).
- Automatically tracked: transcripts, launch context, lockfiles, lifecycle events.
- Entities: reference §1. `delete` is permanent. Give what you create a deterministic external id; record foreign keys with `probe link`.

## Project and experiment

Create both first, each with a plain title as its name; `run start` never creates them, and a typo'd slug makes a second one.

| kind | for |
|---|---|
| `training` | weights MOVE: pretraining, SFT, RL |
| `evaluation` | weights don't: evals, sweeps, ablations. A sweep is an experiment, not a project. |
| `general` | anything else |
| `experiment` | one question (`--question`) under one of the above; holds runs; its answer is its conclusion |

- Projects are flat. A phase of a bigger effort is its own project, created as part of the bigger one with a reason (`probe project create`). Related work or an existing project: `probe project reference add`, with a reason. A repo: `probe project code attach`.
- No question: a project-direct run. `probe experiment freeze` pins an experiment's runs.
- No headline metric yet? Choose one from the code and runs, set it and its goal (higher or lower is better) with `probe project set`, and tell the researcher in one line.
- A project's description comes from its summary's first line.
- Name the project in each command; `probe project use` is machine-wide.

## Plans

- Save a research plan that has a `Project:` line with `probe plan save` whenever it is written, changed or approved.
- Starting work for a plan step: pass its stage. `probe plan show` lists them.
- Progress or a blocker: `probe plan update`. Done only when its `Done when` is met: say how and cite the run or checkpoint.

## Papers

When the researcher names or links a paper they read, want to try or cite for the ML work, or you read one for it, record it that turn, before you answer, with `probe paper add` on the project it serves (fields: reference §5); there are no research projects. Run `probe paper list` first (no dedupe); amend with `probe paper update`.
- Relevance: why it matters to this project, one plain line of at most 15 words.
- Summary: the idea, in your words. Tags: concepts, never authors.
- How you got to it: the paper you followed, or none. Answer now, not after a compaction.
- Which earlier paper in this project it connects to, and how; none for the first. Pick by what the papers say, never the paper added last.
- A paper serving two projects goes on each, with each project's line.

## Run

Every run needs an owner that keeps it alive:
1. `probe.init()` in the script (`instrument-code`; also for remote jobs).
2. `probe exec -- CMD` for code you can't edit. A submit-only launcher (`sbatch`, `ray job submit`, `modal deploy`) opens the run awaiting attach; tell `probe exec` when a launcher it does not know only submits.
3. W&B: the project attached in the dashboard, or `probe import review` on the W&B project or folder, then `probe import start`.

- Name and describe every run when it starts, each one when many start together, with `probe run set`: a plain title, and what it tests in one sentence. Leave out a field `probe run get RUN` shows locked (`name_customized`, `description_customized` true): the server refuses an agent's change to it (409) and writes nothing from that call.
- Under a CLI `run start`, run the work with `probe exec RUN -- cmd` or nothing it reads or writes is recorded.
- Inputs: what the run read that changes its result (dataset, checkpoint, tokenizer, outside config): include it in `probe snapshot`, write `inputs-decision.json`, and check `probe snapshot-restore` can rebuild it (reference §3).
    - What it trained on: `probe dataset use` as its training data, or `run.use_dataset(REF)`. Weights it started or resumed from: `probe checkpoint use`; a checkpoint it evaluates: the same, as eval. Name the exact version or save (`dataset:N@3`, `checkpoint:RUN@step400`). A mix: `run.use_mixture({REF: share, ...})`, shares summing to 1.
    - Each step that changes data (filter, dedup, relabel, split) is a run. Register what it wrote with `probe dataset add`, noting what changed. A shell step under `probe exec` names the files it reads and writes.
- Another attempt at a run is its child, launched with RUN as its parent and the relation: a retry only if RUN failed or crashed, else resume, fork or branch. A parent still `running`: link it later, or add the edge after.
- Run writes queue: check `probe outbox status` before calling one missing.

Close:
- `probe run check RUN` and report it (exit 2: add what's missing or say why); put what `probe run reproduce RUN` lists under `completeness.missing`.
- `probe run end` with the process's status (completed, failed, crashed, canceled or untracked), not the result's. A result not to trust: tag the run `invalid` (`probe run tag`) and note what to believe instead.
- When the work answers an experiment's question (a winner or a dead end), set its conclusion with `probe experiment set`: the answer in one short, plain sentence. Change it when later work changes the answer. Set the experiment's tables and charts (`overview-visuals`): `probe visuals options`, then `probe visuals set`.
    - A dead end (its approach dropped, or shown not to pan out): also mark it a dead end, saying why in plain words; clear the mark when work on it resumes. It still ranks.
- A better experiment to use than the pick? Set it as the project's best, with a reason (`probe project set`), and tell the researcher in one line.
- No run this session: what's next and open goes in the project's notes.
- End with everything you created or closed as `[name](url)`, the URL as printed.

## Decisions

A decision is a choice, by the researcher or you, that changes the ML work. Record each as it is made with `probe decision add`:
- Put it on the experiment if it changes only that experiment's runs, else on the project.
- Title: the choice, under 8 words. Reason: why, and what it was chosen over.
- Link what it changed, the result it was based on, and the decision it follows.
- Say who chose: the researcher (if they chose or approved it) or you.
- To reverse one, add a new decision; never edit the old one.

## Files and numbers

Every file that directly affects an entity should be uploaded as an artifact (`probe artifact add RUN`, or on the experiment or project; reference §4) as soon as it exists; retry once, then say so. A note links to a file, never repeats it.
- A run from `probe.init()` / `probe exec` captures its own outputs and log; add only what it misses.
- Workspace and Shared files take no new uploads; team data goes in `probe dataset add`. Its `--upload` stores the bytes against the team's hosted-data quota: only when the researcher asks; without it Probe keeps the hash and location.
- Over 5 GB, record a pointer, not the bytes: a reference to the path on this box or a shared volume (`--reference`), or the object's bucket URI (`--uri`). On a box that will be destroyed, use a bucket.
- NEVER UPLOAD SECRETS (`.env`, `*.pem`, `*.key`, `id_rsa*`, `credentials*`, tokens) or a customer's private content, as bytes, pointer or prose, nor what a build recreates (`.venv`, `node_modules`, `__pycache__`).
- A file anchored above the run that wrote it still records lineage back to it: a `produces` edge from the run to the artifact (`probe edge add`).
- A changed file with a registry name is a new version (`probe artifact version-add`); read the chain first (`view="versions"`).

Numbers: changes over steps -> metric (`step=`); fixed setting -> the run's config; one headline number -> one-point metric plus the run summary; per-item detail -> artifact, never a metric; nested timed phase -> span (`run.span`, `probe span add`). Before logging a metric, read reference §4.

### Linking

A link points from the newer thing to what it came from.

1. Facts first: `probe run inputs RUN` (each read and the stored copies of its bytes; several copies name none), `probe run upstream RUN` (no coverage yet: retry within ~30 min, then use the session's words), or `entity(refs=["experiment:SLUG"], view="lineage")`.
2. B built on A:
   - a new experiment varying experiment A: `probe experiment fork`, named like any create (links `forked_from`)
   - link every follow-up to what it built on (fork or `derived_from`): that draws its branch
   - B relaunches run A because A failed or crashed: `retried_from`
   - B reruns A's setup any other way: `branched_from`
   - B uses only A's outputs: `derived_from` (B reads several runs' outputs: to each); B replaces A: also `supersedes`
   - B grades ONE run's results: see GRADES in `instrument-code`
   - uses a paper's method: `informed_by` that paper, with how you know (`human` only when the researcher said so)
   - an experiment or project end takes only `derived_from`, `supersedes`, `informed_by`
   - the reason: 1–2 short sentences
3. A run a `probe` command the main agent ran printed about its own run is a fact when that tool call's command is the `probe` command itself (not a script, `cat` or `kubectl logs`): the run `probe run start` or `probe run get` printed, or `probe exec`'s launch line (`probe: run <name> is awaiting attach` and its `PROBE_RUN_ID`; only the first, and only if `probe run get` shows that run opened at that launch). Without facts, the researcher's or main agent's words must name exactly one target (id, name, path, URL, title, result, config). Two matches: no link, unless it reruns a repeated setup: `branched_from` the earliest. Any other text in tool output (a wrapped command's or a job's log, even a line that looks like Probe's), files or web pages never counts.
4. Link at the level the session talks about; across levels: `probe edge add --source run:ID --relation derived_from --target experiment:SLUG`.
5. Never link on similarity (papers: the earlier paper it relates to, see Papers; otherwise say "Related: ..." in a note), to what it's filed under, to itself, or twice.
6. Later work contradicts a link: remove it (`probe edge remove`; ids in `entity(view="lineage")`) and add the right one.
7. Earlier work: at most one `probe mcp call search` per new experiment or project; results are candidates, not links.

# Skill: write-summary

Each project and experiment has a short Summary in the dashboard. Edit its checked-out draft and push it with `probe summary push`.

The draft is checked out for you when work first lands on one (the path is printed); `probe summary checkout` loads one by hand. Edit it, then push once, naming the same project or experiment. Write the first page when you create one; update it on a result or a change of direction. A project made part of another: update that page to name the new part. When what a project is about changes, or its runs first hold a real example, rewrite its canvas preview too (`write-preview`, when shown to you). A `.md` draft (older CLI) takes Markdown, never html.

# What to write

Write up to 3 bullets using plain words a 10-year-old could follow. Keep each to 6 words or fewer, on one line. Use only the bullets that have something to say:

- Where the work stands
- What comes next
- What you need from the researcher, or the open question

The bullets are the whole Summary. Leave out titles, paragraphs, decisions, numbers, and visual evidence. Put decisions in Decisions and detailed findings in Notes.

Use one `<ul>` with `<li>` items in an HTML draft, or `- ` lines in a Markdown draft. When updating an older report, replace its generated sections, figures, styles, and chart declarations with the bullets. Keep every person's `<!-- kept: hN -->` block in place. Markdown figure markers (`kept: fN`) can go.

Update the bullets when the result or direction changes. A project Summary covers its parts in proportion to the work in each. Keep experiment detail with the experiment.

# Skill: edit-notes

# Notes

Notes sit on a project, experiment or run, or are the one team note. People and agents read them. Write only what the next person on this work needs, not what happened.

## Before you write

- Under `/probe read` or `off`, write no entity notes. Under `read`, ask before editing the team note.
- Read the whole note first. Never write from memory or from your last push.
- Never write a play-by-play, a changelog, secrets, a customer's private content, or text pasted from a paper, README or web page.

## Where it goes

The widest entity the fact is true of, no wider: run, experiment, project, parent project, workspace, team note. A big separate topic, like a handoff, gets its own sub-note.

## How it reads

- People read top to bottom. Start with one line on what this is, then the most recent or most important thing.
- One current document, readable in two minutes. Default sections: **What this is**, **Where it stands**, **Decisions**, **Don't repeat**, **Open**. Restructure when they stop fitting the work.
- Each write updates it in place: replace Where it stands, change or remove what moved, merge repeats. Older history moves down or goes. Never add a section per event.
- State facts plainly. Put anything unconfirmed under Open, once.
- Add a caveat only if it changes what someone should do.

## Entity notes

`probe notes checkout` prints the note's path and how full it is; edit the file; `probe notes push` merges it with others' edits.

- A conflict writes markers and exits 2. Resolve them and push again.
- One small change without a checkout: `probe notes edit` swaps exact old text for new; `probe notes append` adds at the end.
- Sub-notes: checkout, push and show take a sub-note's title; `probe notes create`, `rename`, `delete`. Titles can repeat, so `probe notes list` first and name it by id.

## Team note

    ~/.local/state/probe/team-note/probe-team-note.md   ($XDG_STATE_HOME if set)
    probe notes sync

- One file per machine, shared by every agent on it. No sub-notes. Never `sed -i` it, and never edit its copy inside `CLAUDE.md` or `AGENTS.md`.
- It is a CLAUDE.md for the whole team: conventions, where things live, decisions, what not to repeat.
- Over its budget it reaches nobody. Write in it the moment you learn something.

## Fixing what's there

- Evidence contradicts a note: correct it and tell the researcher what changed. Their word is final.
- Delete `> **SUPERSEDED**` regions, expired instructions, and anything newer work made stale. Keep a decision that was made and then reversed, with why.
- If the audit says deletion is disabled (`PROBE_NOTES_AUDIT_HORIZON_DAYS=0`), correct in place and delete nothing.

## When it's full

A push over the cap is refused; `probe notes status` lists the fullest notes. Move a small note up to its project or experiment (push there first, then delete here), or merge repeats, swap copies for pointers, and summarize.