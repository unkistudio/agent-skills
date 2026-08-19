---
name: design-md-writer
description: >
  Generates a structured, project-specific DESIGN.md design system document from
  user input. Use this skill whenever the user asks to create, write, or update a
  DESIGN.md, design system, visual language guide, style guide, or design tokens
  file for a project — even if they just say "write me a design doc" or "define
  the styles for this app". Also use it when the user describes a visual direction
  (e.g. "Apple-inspired", "minimal and dark", "bold and editorial") and wants that
  captured as a reusable design reference. Do not wait for the user to use the
  exact word "DESIGN.md".
---

# Design.md Writer

A skill for producing a thorough, opinionated `DESIGN.md` — a single source of truth for a project's visual and UI language. The output should be specific enough that a developer or AI agent can implement pixel-accurate styles from it alone, without needing to ask follow-up questions.

## What a good DESIGN.md contains

A complete `DESIGN.md` covers these sections (adapt headings to suit the project's tone):

1. **Overview** — one concise paragraph describing the design philosophy, audience, and the single strongest visual metaphor. State what the design is *not* (e.g. "no decorative gradients, no playful motion").
2. **Principles** — 4–6 named, opinionated principles. Each: short title + one-sentence rationale. They act as tie-breakers when implementation choices are ambiguous.
3. **Colours** — every token needed to build the UI, grouped by role (brand, surface, text, status, border/focus). For each: CSS variable name, hex value, plain-language description of where and why it is used.
4. **Typography** — font family stack (with fallbacks), a hierarchy table (token, size, weight, line-height, use), and concise rules covering tracking, line length, and weight ladder.
5. **Layout** — spacing scale (as a code block), max container width, padding rules per breakpoint, named breakpoints, numbered description of the page structure, and how each major component reflows at each breakpoint.
6. **Components** — key reusable pieces: buttons (all variants), cards/tiles, form inputs, links, tables, banners/alerts. For each: background, border, radius, padding, hover/focus/error states.
7. **Motion** — what transitions are allowed (property, duration, easing) and what is explicitly forbidden.
8. **Accessibility checklist** — concrete, checkable items: focus ring spec, contrast requirements, keyboard navigation, label requirements, language attribute.

Sections may be renamed or reordered. Skip a section only if genuinely not applicable.

## Gathering input

**Before writing a single line of the DESIGN.md, you must ask the user about accessibility intent.** This is the one input that must never be inferred — even if context seems to imply an answer. Getting it wrong silently produces a design system that fights the intended aesthetic. Ask explicitly every time.

For all other inputs, extract from context first. If something is still missing, ask before writing.

| Input | What to look for |
|---|---|
| **Project type** | What kind of product? (dashboard, marketing site, e-commerce, internal tool…) |
| **Audience** | Who uses it and in what context? |
| **Visual direction** | Any adjectives or reference sites the user mentioned |
| **Existing brand assets** | Logo, colour palette, font constraints |
| **Existing styles** | Any CSS already written — describe it, don't contradict it |
| **Accessibility intent** | **Always ask.** Is accessibility a legal/contractual requirement (→ enforce WCAG AA/AAA), a best-effort goal (→ aim for AA where it doesn't compromise aesthetics), or not a priority (→ design freely)? |
| **Language** | UI language(s) — capture relevant typographic conventions |
| **Responsive target** | Which breakpoints matter? Mobile-first or desktop-first? Any breakpoints to explicitly support? |

## Writing the document

### Colour tokens

- Name every token as a CSS custom property (`--primary`, `--surface-alt`, etc.) with hex and use description.
- Group under: Brand, Surface, Text, Status, Borders & Focus.
- Colour contrast requirements are only enforced to the extent the project's accessibility intent demands. Do not default to WCAG AA — derive the expectation from the stated intent (legal obligation → AA/AAA; broad public audience → AA best-effort; artistic/editorial).

### Typography

- Prefer system font stacks unless the project specifies a brand font.
- Present hierarchy as a table: Token | Size | Weight | Line-height | Use.
- Write rules as bullet points — they are scanned, not read linearly.
- Include language-specific conventions if the UI is not English.

### Responsive design

- Define named breakpoints explicitly (e.g. `--bp-sm: 480px`, `--bp-md: 768px`, `--bp-lg: 1024px`). Do not leave breakpoints implicit or scattered.
- State the approach: **mobile-first** (default — `min-width` media queries, style the smallest viewport first) or desktop-first (`max-width`). Be consistent throughout.
- For every layout component (grid, nav, card grid, sidebar), describe exactly how it reflows at each breakpoint. "Stacks on mobile" is not enough — specify the column count, gap, and padding at each step.
- Typography must scale too: define which tokens shrink at smaller viewports and by how much.
- Touch targets must be at minimum 44×44px on mobile regardless of accessibility intent — this is a usability baseline, not an accessibility constraint.
- Avoid fixed pixel widths on containers that will be viewed on mobile. Use `max-width` + `width: 100%` patterns.

### Component specs

- Describe every interactive state: default, hover, focus, active, disabled, error.
- Use concrete values (`border-radius: 4px`, `padding: 10px 20px`). Avoid vague language like "slightly rounded" or "subtle shadow".
- Derive component style (radius, weight, motion) from the project's stated visual direction.

### Tone matching

| Project type | Prose tone |
|---|---|
| Professional / utility | Direct, technical, no superlatives. |
| SaaS / startup | Confident, concise, slightly opinionated. |
| Agency / editorial | Descriptive, visual metaphors welcome. |
| Internal tooling | Terse, spec-first, minimal prose. |

### What to avoid

- Do not copy another product's design system verbatim — adapt to the project's actual constraints.
- Do not define tokens or components the project will never use.
- Do not use vague adjectives without a concrete backing value ("clean" means nothing; `border-radius: 4px` does).
- Do not define decorative gradients unless explicitly called for.

## Output format

Every `DESIGN.md` **must begin with a YAML front-matter block** (between `---` fences) before any Markdown headings. The block is a machine-readable snapshot of the full design system — always included, never optional.

### Required front-matter structure

```yaml
---
version: <semver or stage, e.g. 1.0 / alpha>
name: <kebab-case project identifier>
description: >
  One dense paragraph: visual direction, audience, canvas/ink colours, primary
  accent, type stack, layout approach, and what the design explicitly is not.

colors:
  <token-name>: "<hex>"   # one entry per colour token

typography:
  <token-name>:
    fontFamily: "..."
    fontSize: <px>
    fontWeight: <number>
    lineHeight: <ratio>
    letterSpacing: <px>   # omit if 0

spacing:
  <label>: <px>   # every step in the scale

components:
  <component-name>:
    backgroundColor: "{colors.<token>}"
    textColor: "{colors.<token>}"
    # ... other concrete properties (border, borderRadius, padding, height, etc.)
---
```

**Rules for the front-matter:**
- Use `{colors.<token>}`, `{typography.<token>}`, `{spacing.<token>}` references inside `components:` — never inline hex values.
- Keep token names consistent between `colors:` / `typography:` / `spacing:` and their use in `components:` and in the Markdown body.
- `description` must stay general enough to apply to any page in the product, not describe a single component.
- **Always use the block scalar `>` or quote the `description` value with double quotes.** A bare (unquoted) description containing a colon followed by a space (e.g. `design systems: high-contrast`) will cause a YAML parse error — YAML treats `key: value` anywhere in a scalar as a nested mapping. Use `description: >` (folded block) or `description: "..."` (double-quoted string) to avoid this.
- Strip backticks from all front-matter string values — backtick characters have no meaning in YAML and can trigger edge-case parser errors.
- After the closing `---`, continue with the standard Markdown sections (`# Title`, `## Overview`, etc.).

### Other output rules

- Clean GitHub-flavoured Markdown. `##` for sections, `###` for subsections.
- Colour tokens as a table; typography hierarchy as a table; spacing scale as a fenced code block.
- Keep under ~300 lines. If longer, split component detail into `COMPONENTS.md` and cross-reference.
- After writing, offer to update the project's CSS to match.

## Quick-reference checklist before finishing

- [ ] Every CSS variable defined is used somewhere in the UI spec.
- [ ] Colour contrast decisions reflect the project's stated accessibility intent (enforced ratio, best-effort, or design freely).
- [ ] Every interactive component has a focus state.
- [ ] Every layout component has explicit reflow rules for each named breakpoint.
- [ ] Touch targets are ≥ 44×44px on mobile.
- [ ] The spacing scale is complete with no gaps.
- [ ] Language-specific typographic conventions captured if UI is not English.
- [ ] The doc states what the design is *not*, not just what it is.
- [ ] The front-matter `description` is either a block scalar (`>`) or double-quoted — never a bare string containing `: ` (colon-space).
