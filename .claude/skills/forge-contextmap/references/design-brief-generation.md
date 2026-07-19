## DESIGN BRIEF GENERATION *(shared by New / Existing / Migration modes)*

Goal: turn one vibe answer into a COMPLETE, CONCRETE design system — so every future UI task uses
the same values. No randomness: real hex codes, a real font name, a fixed scale.

From the user's vibe/brand answer, generate ALL of:

- **Color tokens** — at minimum Primary, Surface, Text, Muted, Border, Success, Error — each a
  concrete hex value that fits the vibe. If the user gave brand colors, build around them. Ensure
  Text-on-Surface contrast is readable (aim WCAG AA).
- **Typography** — one font family (prefer widely available: Inter, system-ui stack, Roboto, SF
  Pro, or the user's brand font), a numeric scale (e.g. 32/24/18/16/14), and weights.
- **Spacing & radius** — a base unit (e.g. 4px) with an allowed set, and radius values.
- **Reusable components** — the starter inventory appropriate to the app type (typically: Button
  primary/secondary/ghost, Card, Input, Modal, Toast; add List Item, Avatar, Badge, etc. as the
  features demand).
- **Screen style guidance** — one line each for list screens, detail screens, forms, empty states.

Present the whole set in a compact block and ask for approval:

```
Proposed design system (from "[their vibe answer]"):

Colors:   Primary #.. · Surface #.. · Text #.. · Muted #.. · Border #.. · Success #.. · Error #..
Type:     [font] — 32/24/18/16/14 · weights 400/500/700
Spacing:  4px base (4/8/12/16/24/32) · Radius 8px
Components: Button (1°/2°/ghost), Card, Input, Modal, Toast[, ...]

Approve, or tell me what to change (any value is adjustable).
```

Apply corrections, then write the approved values into `doc/design-brief.md`. If the codebase
already defines a theme/token file, EXTRACT from it instead of generating — the code is the truth;
present what you found for confirmation.
