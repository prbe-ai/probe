---
name: artifact-diagramming
description: Draw inline SVG diagrams that show a mechanism (what flows where, what differs between options) as the engineer who has to live with the decision. Load before drawing any diagram.
source: Claude Code built-in skill, captured 2026-09-07; condensed
---

Draw as the engineer who has to live with the decision, not as a decorator. A diagram earns its place when it lets a cold reader see a mechanism they would otherwise assemble from prose: where data flows, which components talk, what changes between two options, what states a request moves through. If a sentence says it faster, write the sentence.

## What to draw

**Depict the mechanism, not its name.** A box labeled "cache" says less than the prose; the path a request takes through it, the two stores it sits between, and the arrow that disappears without it say what words can't. Show the parts the argument hinges on (the boundary crossed, the hop added, the data that moves) and leave out the rest.

**Comparing options? Draw the difference.** Side by side, before and after, the one edge each option adds or removes: the reader should be able to point at what they are choosing between. A separate box per option, connected to nothing, is a restated option list, not a comparison.

**Match complexity to the stakes.** A one-hop question is three boxes; a migration that reroutes writes through a queue needs the queue, the writer, the reader and the ordering arrow. Draw what the decision turns on: no forced minimalism, no inventory of the whole system.

**Label the arrows.** An unlabeled arrow means "related somehow"; `writes`, `invalidates`, `polls every 30s` is information. Add a legend only when an encoding (dashed, colored, doubled) repeats; otherwise put the meaning on the mark.

## Inline SVG mechanics

Hand-author inline `<svg>` with native shapes (`rect`, `circle`, `line`, `polyline`, `path`) and `<text>`: no libraries, no runtime, no external images.

- **Size by `viewBox`.** Set `viewBox="0 0 W H"` and let CSS scale it (`max-width: 100%; height: auto`); choose W and H for the content. Wide flows read left-to-right; layered stacks top-to-bottom.
- **Theme with `currentColor`.** Strokes, text and arrowheads in `currentColor` inherit the page's foreground; keep a literal hue for the one element that carries meaning (the option leaned toward, the hop under discussion), and make sure it reads on the light ground.
- **Arrowheads are markers or polygons.** A `<defs><marker>` referenced by `marker-end="url(#arrow)"` (fragment-internal id) or a small `<polygon>` at the line's end, never an image.
- **Keep text legible.** About 11-13px at the drawn scale, `text-anchor` for alignment, labels of a word or three; explanations go in the caption, not the drawing.
- **Align to a grid.** Shared baselines and even gaps make a hand diagram read as deliberate; eyeballed offsets read as noise.
- **One figure, one claim.** Wrap the `<svg>` in `<figure>` with a `<figcaption>` stating what it shows, and give the `<svg>` `role="img"` plus an `aria-label` with the same claim.
- **Stay self-contained.** No `<script>`, `<style>` or `<foreignObject>` inside the SVG; gradients, patterns and `<use>` reference ids in the same fragment (`href="#id"`). Long decorative path data means the drawing wants simplifying.
