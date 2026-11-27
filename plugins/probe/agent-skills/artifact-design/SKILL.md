---
name: artifact-design
description: Design a self-contained HTML page as the design lead at a small studio would - palette, type, layout and copy pitched to the subject, never templated. Load before writing any page.
source: Claude Code built-in skill, captured 2026-09-05 and checked against the 2026-10-28 one (same rules); condensed, without the passages about publishing on claude.ai, dark mode, editorial pages, apps and tools, which a Summary page never is
---

In Probe a page is the Summary pane's content: light only (choose every color for the light ground, no dark-mode blocks), no title (the dashboard prints the name), and a utilitarian treatment, a research memo made polished.

Work as the design lead at a small studio known for versatility: deliberate palette, typography and layout specific to this subject, never a template.

## The treatment

A Summary page is a memo: polished, with real typographic hierarchy, considered spacing and a proper palette, but never over-designed. No gigantic hero; few, tasteful flourishes. A well-composed page is never the wrong answer; an over-designed identity sometimes is.

## Fundamentals

**Honor what's already there.** The host's tokens (`--ov-*`, `overview-contract`) are the existing system: build on them. Precedence: the user's own words (the page skills), then that system, then your choices.

**Ground it in the subject.** Pin one concrete subject, its audience and the page's single job. Distinctive choices come from the subject's own world: its materials, instruments, vernacular. Carry at least one detail only this subject would have (its real units and scales, document conventions, terms of art) as content, not ornament. Real content throughout, never lorem.

**Pair typefaces.** Typography carries the page. Google Fonts is the one font host the Artifact CSP admits - link it directly (`<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=...&display=swap">`); a face from anywhere else must be inlined as a @font-face data URI or it falls back silently. Declare a real fallback stack. Keep running text near 65 characters wide; set a type scale and stay on it; give headings `text-wrap: balance`, body text room to breathe, and uppercase labels a touch of letter-spacing.

**Load libraries, don't paste them.** When a chart genuinely needs one, load a pinned build `overview-contract` lists, with one `<script src>` before the inline script that uses its global; never paste its source or hand-write a stand-in. Most visuals need no library.

**Choose neutrals, don't default to them.** A pure mid-grey reads as unconsidered; a grey with a slight hue bias toward the accent reads as chosen. Pure white and near-black are fine grounds when they suit the subject, as long as the neutral was picked.

**Let layout do the spacing.** Lay out sibling groups with flex or grid and `gap`, not per-element margins that collapse or double. Wide content (tables, code, diagrams) gets `overflow-x: auto` on its own container so the body never scrolls sideways. Use `font-variant-numeric: tabular-nums` wherever digits line up in columns.

**Compose repeated things as one object.** Cards in a row, label/value pairs, badges on siblings: same edges, baselines and inner padding, and a recurring element in the same place on each. Let content set a container's height and pick a column count the items fill, so nothing stretches over dead space or sits alone in a row. Text that can outgrow its track wraps or scrolls in its own container; clipped text is a bug.

**Not everything is a card.** Border, fill, radius and shadow each say "separate object": spend them by role on the one thing that needs lifting, not one radius and shadow stamped on every block. Lead with big-number tiles only when those figures are the point of the page.

**Draw charts to the scale.** One scale places marks, ticks and labels, and every label names a value the chart reaches; chart text takes its color from the page's tokens; marks, labels and edges stay clear of one another and inside the bounds - in SVG, leave room in the viewBox for the outermost labels and give every shape an explicit fill.

**Show the page at rest.** Everything meant to be read is visible once the page has loaded, without scrolling to trigger it: that first frame is what a thumbnail, a shared link and a skimming reader get. A section may animate in from a visible resting state, never parked at `opacity: 0` waiting on an observer. Size a hero to what it holds; a `100vh` opener pushes the page out of that first frame.

**Avoid AI-generated design.** It clusters around a few looks: warm cream (#F4F1EA) with a serif display and terracotta accent; near-black with a lone acid-green or vermilion pop; broadsheet hairline rules with dense columns; a purple-to-blue gradient hero on white; Inter or Space Grotesk as the "safe" face; emoji as section markers; everything centered; `rounded-lg` everywhere; accent bar/rail on rounded cards. When the user pins a visual direction, follow it exactly, even one of these looks; when nothing is specified, don't spend that freedom on one of them.

**Build cleanly.** Watch for overlapping elements, cascade collisions and silent font fallbacks. Close every non-void element, double-quote attributes, give keyboard focus a visible state, respect `prefers-reduced-motion`. For generative or decorative graphics, use Canvas or WebGL rather than long hand-written SVG paths.

**CSS rules.** Watch selector specificity: classes easily cancel each other (a `.section` rule fighting a `.cta` rule over padding and margins). Structure the cascade so it doesn't silently undo your spacing.

**Writing the copy.** Words are design material. Write from the user's side of the screen: name things by what people recognize, not how the system is built (a person manages *notifications*, not *webhook config*). Active voice. Specific beats clever.

**Structure is information.** Numbering, eyebrows, dividers and labels encode something true about the content, not decoration. Numbered markers (01 / 02 / 03) fit only a real sequence, such as a process or a timeline whose order the reader needs.

## Process

Before writing code, settle a short design plan and write it into the page itself, as tokens at the top of its `<style>` (scoped under `.ov-page`), not into your reply:
- **Color**: the palette as 4-6 named tokens.
- **Type**: typefaces for 2+ roles - a characterful display face used with restraint, a complementary body face, and a utility face for captions or data if needed.
- **Layout**: a layout concept as a one-line comment above the tokens.

Then build, deriving every color and type decision from the plan.
