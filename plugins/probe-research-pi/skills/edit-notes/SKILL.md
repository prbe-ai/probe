---
name: edit-notes
description: How to write notes on a project, experiment or run, and the team note.
---

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
