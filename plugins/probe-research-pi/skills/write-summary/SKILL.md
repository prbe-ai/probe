---
name: write-summary
description: Keep each project's and experiment's page - what people read to catch up.
---
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
