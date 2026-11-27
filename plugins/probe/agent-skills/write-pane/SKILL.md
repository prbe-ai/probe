---
name: write-pane
description: Write a pane - a live visual on a project, experiment or run page. Use it unasked whenever a run, experiment or project has important data the built-in charts and tables can't show; never for what they can. When you report such results, put them in a pane too, without asking, since the researcher wants them on the page.
---
# Write a pane

A pane is your own HTML/JS plus a recipe: the data it reads, never the data itself. Probe resolves
the recipe each time the pane opens, for the runs in view (an experiment's, a project's newest 50,
or the runs someone picked to compare), so new runs show up without a rewrite. Panes sit at the top
of the project's or experiment's page; summary rewrites never touch them. A pane about one run goes
on that run (`id:<run-id>` in place of the project or experiment) and reads that run alone.

First ask whether a built-in view shows it: a curve over steps is in the metric section; a score
per run, per config or per group (a leaderboard, by a config, by a group such as document length)
is a table or chart in the Metrics-center visuals of the experiment whose runs recorded it
(`overview-visuals`); the project page shows each experiment's charts too. A pane is for the
important data left, chart or not: each frame's score under two models, before/after images, the
PDFs a run wrote, why tasks did not finish.

A score per group that no run has logged yet is still a built-in chart:

1. The run logs the group's mean once per group, under its own key, with a dimension of a few
   values; never one call per task, never a task id as the dimension
   (`probe.log({"eval/score_by_length": mean}, dimensions={"length": "1-2 pages"})`). For a run that
   already ended, `probe log` the same value with that dimension, marked derived with what
   computed it, at the run's last step.
2. Chart it as a `bar` with `x: {dimension: "length"}` (`probe visuals set`; where the daemon
   records, ask its writer with `probe ask --writer`).

A pane shows rows, files and the server's `totals` as they are; it never averages `samples` rows
into a per-run or per-group score.

## When to write one

- A project's canvas preview (the pane its box on the canvas shows) is `write-preview`'s job.
- When results land, before you report them: if something important can't be shown by a built-in
  view, write a pane, unasked. Only while Probe records this chat (the daemon records it, or you
  took over inline): while Probe is `read` or `off`, or after it refuses a pane command, skip it
  and say in one line that a pane would help. Never add `--directed` to a pane nobody asked for.
- Run `probe pane list` first. Replace only a pane whose slug you pushed in this chat; every other
  pane is someone else's (when unsure, it is) and stays, and a new pane shows only what it doesn't.
  A new slug must not be in the list: a push to a taken slug replaces that pane for good. One new
  pane per question.
- Push only when `probe pane data` shows the rows the pane reads, each file by its name on its run.
  A writer's reply, or an ask still looking, is not proof. Then tell the researcher in one line,
  with the `url` the push printed. If the attach or the push fails, report the results in text and
  say the pane wasn't made; never give a link for it.
- A project or experiment holds 8. When it is full, ask the researcher which one to replace; with
  nobody to ask (a script or a sweep), skip the pane and say so.

## 1. Put the data on the runs

A pane reads only what its runs hold: metrics, spans and attached files. A run that writes results
to files attaches them (`probe.log_artifact("eval/results.jsonl", path=...)`). Results still only on
disk from a run that ended: where the daemon records, ask its writer to attach them and wait for
its answer (`probe ask --writer --wait`); otherwise `probe artifact add`. Attach only result files the run wrote, each by name:
never env, config or key files, logs, or results holding secrets or data the researcher hasn't
shared with the team.

Per-task results work best as one rollout span per task, scores in `attributes`, output files
linked to it:

```python
with probe.span("rollout", name=task.id, external_key=task.id) as span:
    out = solve(task)
    span.attributes.update(score=out.score, unfinished_reason=out.reason)  # None when it finished
    probe.log_artifact(f"eval/{task.id}/after.png", path=out.image, span_id=span)
```

A run that writes Word files also saves a PDF of each, so a pane can show the exact look
(`soffice --headless --convert-to pdf --outdir out/ out/after.docx`), and logs both to the span.

A sample is `failed` only when an exception escapes the `with` block; `unfinished_reason` alone
leaves it `completed`, so count why tasks did not finish by that attribute, not by status. From a
shell, `probe span add` prints the span id; `probe artifact add` can attach a file to that span.

## 2. Write the recipe and check it

```json
{"runs": "scope",
 "data": [
  {"id": "variants", "source": "runs", "config": ["lr", "model"]},
  {"id": "tasks", "source": "samples", "fields": ["attributes.score"], "counts": ["attributes.unfinished_reason"]},
  {"id": "after", "source": "files", "prefix": "eval/", "name": "*.png", "span": "rollout"},
  {"id": "results", "source": "files", "name": "*.jsonl"}]}
```

`runs` is `"scope"` (the runs in view) or a list of run ids. `files` filters by `prefix`, `name`
glob, `span` type and `kind` (an artifact kind such as `evaluation`; there is no `image` kind, so
match images by `name`, `*.png`). Each source's rows:

- `runs`: `id`, `name`, `status`, `experiment_id`, `created_at`, `locked`, `config{}` (null for a
  locked run, or one past the size cap, named in `cut`), `scalars{"<key>": {last, min, max, step}}`
  (a key the run never logged is absent).
- `series`: `run_id`, `key`, `points` as `[step, value]` pairs.
- `samples`: `id` (the span id), `run_id`, `name`, `external_key`, `status`, `step`, `started_at`,
  `ended_at`, `fields{"attributes.score": 0.8}`. A null or missing attribute is left out of
  `fields`.
- `files`: `id`, `run_id`, `name`, `kind`, `content_type`, `size_bytes`, `span_id`, `step`,
  `created_at`; a file belongs to the sample whose `id` is its `span_id`.

Before writing code, see what an unsaved recipe resolves to with `probe pane data`. Its preview
gives each entry's `state` (`ok`, `partial`, `too_large`), `rows_total` and first rows, and a
line's first and last points (`points_total`), long values cut. A samples entry adds `totals`: per
run, `total` and `counts{"status" or field: {values: [{value, count}], other}}` over ALL its
rollout spans, exact even when rows are cut. Spans without the field count under `value: null`:
drop that bucket when counting reasons.

`unmatched` names the keys no readable run has, and a samples entry's `unseen_fields` the fields
not in the rows read: a typo, or not logged yet (`state` stays `ok`). Take real names from a run:
`probe run series` lists its metric keys; `probe span list` on a few rollout spans shows their
`attributes` (not the RL samples command).

## 3. Write the code

- A pane is an HTML fragment: no `<!doctype>`, `<html>`, `<head>`, `<body>` or `<!-- comments -->`.
  Start with your markup, `<style>` and `<script>` tags.
- `PROBE.runs('variants')`, `PROBE.samples('tasks')`, `PROBE.files('after')` (and `PROBE.series(id)`
  for a `series` entry) return that entry. Name each id literally and read every entry the recipe has.
- `PROBE.image(row)` resolves to a `data:` URL, `PROBE.text(row)` to text (CSV and JSONL too),
  `PROBE.json(row)` to parsed JSON. They read only rows of the pane's own `files` entries. Each can
  reject (`budget`, `not_in_files`, `too_large`, `unavailable`): catch it and draw a placeholder, or
  the whole pane shows as failed. Text and JSON stop at 1 MiB: read a bigger result file with
  `PROBE.bytes(row)` and `new TextDecoder().decode(bytes)`.
- `PROBE.bytes(row)` resolves to an `ArrayBuffer` of the file's original bytes (a PDF, a Word
  file, any binary up to 25 MiB; bigger rejects `too_large`) for a library to draw.
- Images: create the `<img>` in script and set its `src` from `PROBE.image(row)`; an `<img>` in
  the markup is refused. A `<script>` is not a module, so wrap `await` in a function:

  ```js
  (async function () {
    var img = document.createElement('img');
    img.alt = row.name;
    document.getElementById('grid').appendChild(img);
    try { img.src = await PROBE.image(row); } catch (e) { img.alt = row.name + ': ' + e.message; }
  })();
  ```
- A `partial` entry still draws what it has; say it was cut.
- Color each run with its `color` from `PROBE_DATA.runs` (match by `id`) so the pane matches the
  explorer. The dashboard adds it (`probe pane data` does not show it) and it can be null: keep a
  fallback palette.
- The frame reaches no network: no `fetch`, XHR, WebSocket or `import()`, no `eval` or
  `new Function`, no `postMessage`, no cookies. Load a library as one exact-version cdnjs build,
  `<script src="https://cdnjs.cloudflare.com/ajax/libs/d3/7.9.0/d3.min.js"></script>` (not Plotly
  or Vega: they need `eval`). A module-only library loads by its src,
  `<script type="module" src=".../pdf.min.mjs"></script>`; its global exists from
  `DOMContentLoaded` on.
- Buttons, `<select>`, and checkboxes and sliders written as markup (`<input type="checkbox">`,
  `<input type="range">`) are fine; never `createElement('input')`. No text fields,
  `contenteditable` or key listeners. A `<script>` takes only `src` (plus `type="module"` beside a
  library's `src`); attach handlers with `addEventListener`, never `on*` attributes.
- Look: start with the content; the dashboard prints the title above it. No outer border, card or
  box, and no inner scrollbar: the pane grows to its content. Use the host's colors (`--ov-fg`,
  `--ov-muted`, `--ov-line`, `--ov-accent`, `--ov-good`, `--ov-bad`, and the chart hues
  `--ov-c1`...`--ov-c8` for fills and marks, never text); for design, `artifact-design`.

## 4. Live runs, documents, click-through

- **Live.** While a run in view is going, the dashboard reads the pane's data again every 30 s.
  Pass your draw function to `PROBE.onData`; the reads above answer the new data, and your own
  controls keep their state. Without it the pane keeps its first picture until someone refreshes.

  ```js
  function draw() { renderTable(PROBE.samples('tasks').rows, currentFilter); }
  draw();
  PROBE.onData(draw);
  ```
- **PDF.** pdf.js from its module build; draw only the pages on screen (this draws page 1):

  ```html
  <canvas id="page"></canvas>
  <script type="module" src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/6.4.299/pdf.min.mjs"></script>
  <script type="module" src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/6.4.299/pdf.worker.min.mjs"></script>
  <script>
  document.addEventListener('DOMContentLoaded', async function () {
    try {
      var bytes = await PROBE.bytes(PROBE.files('docs').rows[0]);
      var doc = await window.pdfjsLib.getDocument({ data: bytes }).promise;
      var page = await doc.getPage(1), viewport = page.getViewport({ scale: 1.2 });
      var canvas = document.getElementById('page');
      canvas.width = viewport.width; canvas.height = viewport.height;
      await page.render({ canvasContext: canvas.getContext('2d'), viewport: viewport, canvas: canvas }).promise;
    } catch (e) { document.getElementById('page').replaceWith(e.message); }
  });
  </script>
  ```
- **Word.** `mammoth.convertToHtml({arrayBuffer: bytes})` (cdnjs `mammoth/1.13.0/mammoth.browser.min.js`)
  gives a plain view: text, bold, italics and tables survive; fonts, colours and alignment do not.
  For the exact look, draw the PDF the run saved beside it.
- **Before/after diff.** For a document edit, unzip both files with JSZip
  (cdnjs `jszip/3.10.1/jszip.min.js`), read `word/document.xml`, and compare with jsdiff
  (cdnjs `jsdiff/8.0.2/diff.min.js`, global `Diff`): `Diff.diffArrays` over paragraph texts finds
  the changed paragraphs, `Diff.diffWordsWithSpace` the changed words, and comparing each run's
  `<w:b/>` and `<w:i/>` and each paragraph's `<w:jc>` shows formatting changes. Show it git-style:
  removed lines red, added green, unchanged paragraphs folded. The raw XML diff (`Diff.diffArrays`
  over its lines) is what the model wrote.
- **Click-through.** `PROBE.open(ref, {newTab})` opens a run (`{run: id}`: a `runs` row's `id`, any
  other row's `run_id`), a sample's trial page (`{sample: {run: row.run_id, id: row.id}}`) or a
  file's page (`{file: row.id}`). It opens only right after a click in the pane, once per click,
  and only what the pane's data holds; otherwise it rejects (`no_gesture`, `not_in_data`). Catch
  it, or the whole pane shows as failed: `PROBE.open(ref).catch(function () {})`.

## 5. Push it

Check first, then save:

```
probe pane push <scope> <slug> --html pane.html --recipe recipe.json --check
probe pane push <scope> <slug> --html pane.html --recipe recipe.json
```

The scope is the project or experiment slug, or a run as `id:<uuid>`. `--check` saves nothing and
lists every problem with its line: fix them all, check again, then push without it. A refused push names the same problems,
including a browser check's failing width, state and error; a warning means that check could not
run and the pane was saved unchecked. A new pane goes last unless you give it a place. The answer's
`url` is the page the pane is on.

Pushing the same slug again replaces the pane (the answer says `replaced`), so a retry is safe. To
change a pane later, `probe pane get` it into a folder (its HTML and recipe), edit both and push
them back; a re-push keeps the title unless you give a new one.

`probe pane rm` deletes a pane for good, with no trash: `probe pane get` it into a folder first.
Run it only when the researcher asks; then pass `--yes`, since an agent cannot answer its prompt.
Its runs and their data are untouched.

Pin a view only when the researcher asks, and once: pins sit at the top of the project (a run's
view: the run's page) for the whole team, 24 at most, so pinning and unpinning take `--directed`
(`probe pane push` can pin the pane it saves; `probe pin add` pins a saved pane, a metric's chart,
the key metrics or the parallel plot; `probe pin list`, `probe pin rm`). Never pin from a script or
a sweep that runs again: a pin someone removed would come back. When Pinned is full, ask the
researcher which view to unpin.

From a script or notebook, the SDK does the same:

```python
client = probe.Client()
client.save_pane("vision-evals", "before-after", html, recipe, title="Before / after")
client.read_pane_data("vision-evals", recipe)  # what `probe pane data` prints
```

## 6. Limits

Past one of these a push is refused, a data read comes back `partial` or `too_large`, or a `PROBE`
file read rejects `too_large`:

| Limit | Value |
|---|---|
| Panes in one project or experiment | 8 |
| A pane's HTML | 300,000 characters |
| A pane's title | 200 characters |
| Entries in one recipe | 16 |
| Runs a recipe reads (a scope's newest) | 50 |
| Keys in one `runs` or `series` entry | 32 |
| Fields in one `samples` entry | 32 |
| `counts` fields in one `samples` entry, beside status | 3 |
| Sample rows in one entry | 5,000 |
| File rows in one entry | 2,000 |
| Points in one series line | 300 |
| One pane's resolved data | 2 MiB |
| Time to resolve one pane's data | 5 s |
| One image `PROBE.image` reads | 10 MiB |
| One file `PROBE.text` or `PROBE.json` reads | 1 MiB |
| One file `PROBE.bytes` reads | 25 MiB |
| Slug | `^[a-z0-9][a-z0-9-]{0,62}$`: lowercase words joined by hyphens |
| Recipe entry `id` | `^[A-Za-z][A-Za-z0-9_-]{0,63}$` |
