---
name: design-prototype
description: >
  PM → UX → HTML mockup workflow. Takes a feature request, scopes it against the
  existing project, designs the UI using project conventions and design tokens,
  and produces pure HTML prototypes + MD specs in .designs/. Triggered by phrases
  like "design the UI for X", "create a mockup of X", "prototype this page",
  "how should X look", "redesign X page".
---

## What It Does

Three-stage pipeline:

```
Feature request → PM clarifies scope → UX reads project → HTML prototype + MD spec
```

Produces two files per feature in `.designs/<feature-name>/`:
- `index.html` — pure HTML/CSS prototype, no frameworks
- `spec.md` — developer-ready spec with component tree, states, DS mappings

### Folder structure

```
.designs/
├── tokens.css              ← shared token sheet (link from all prototypes)
├── dashboard/
│   ├── index.html
│   └── spec.md
├── pacientes-list/
│   ├── index.html
│   └── spec.md
├── paciente-detail/
│   ├── index.html
│   └── spec.md
├── turnos/
│   ├── index.html
│   └── spec.md
├── configuracion/
│   ├── index.html
│   └── spec.md
├── historia-clinica/
│   ├── index.html
│   └── spec.md
├── odontograma/
│   ├── index.html
│   └── spec.md
└── facturacion-obras-sociales/
    ├── index.html
    └── spec.md
```

### Versioning

- **No v1, v2 suffixes.** Git manages versions. Just edit the files in place.
- If the user says "don't touch the other one" or asks for a completely different flow, create a new folder (e.g., `facturacion-obras-sociales/` separate from `configuracion/`).
- If a feature replaces another, update the existing folder. Don't create duplicates.

### Core principle: design is source of truth for looks

- The design file dictates visual decisions. Code follows the design.
- But the code drives structure — models, stores, API endpoints constrain what's possible.
- Iterate both: read code → improve design → update code to match.
- When design and code differ, add a "Design → Code alignment" section in the spec to flag what code changes are needed.

---

## Stage 1: PM — Clarify the scope

Before any design work, resolve ambiguity. Ask all questions in one batch:

1. **Where does this live?** — New page? New tab? Replace existing component? Which route?
2. **What data is involved?** — What models? What API endpoints exist (or need to exist)?
3. **Who uses it?** — Dentist? Secretary? Patient? What device (desk, tablet, phone)?
4. **What's the primary action?** — What does the user do most often here?
5. **What already exists?** — Is there a current version? What's wrong with it?
6. **Edge cases** — Empty state? Errors? First-time use?

If the feature is vague ("build me a dashboard"), push back for specifics before designing.

---

## Stage 2: UX — Read the project before designing

**Mandatory steps before any design work:**

1. **Detect the stack**: check for `nuxt.config.ts` (Nuxt) or `/backend` + `/frontend` (Go + Vue SPA).
2. **Read existing design specs** in `.designs/` — many patterns are already solved.
3. **Read the current implementation** — the actual `.vue` files, stores, and models. Don't design in a vacuum.
4. **Check the design tokens** in `.designs/tokens.css` — all prototypes link to this file.
5. **Check @el-agala/ui components** — load the `agala-ui` skill to know what components exist.
6. **Search for similar UI patterns** already used in the project — reuse before inventing.

**Design principles:**
- Reuse existing components from @el-agala/ui. Do not invent new ones unless necessary.
- Use semantic HSL tokens (`hsl(var(--agala-primary))`), never hardcoded hex or px.
- Every interactive component must have all four states defined: **loading, empty, error, populated**.
- Medical/safety-critical info must be always visible (alerts above tabs, prominent banners).
- Simple yet sophisticated — creative layouts, not locked to columns.
- No assumptions about copy/tone — flag user-facing text that needs review.

---

## Stage 3: HTML Prototype + MD Spec

### HTML prototype (`index.html`)

Rules:
- Pure HTML + CSS. No JavaScript frameworks. No Vue. No Tailwind.
- Link to `<link rel="stylesheet" href="../tokens.css">`
- Import fonts: Inter (body) + Space Grotesk (headings) + JetBrains Mono (code/data)
- Include `<meta name="viewport" content="width=device-width, initial-scale=1.0">`
- Three responsive breakpoints:
  - **Tablet** `@media (max-width: 900px)` — sidebar drops, columns stack
  - **Mobile** `@media (max-width: 640px)` — single column, compact cards, touch targets ≥40px
  - **Small phone** `@media (max-width: 400px)` — further compaction, font floor at 0.42rem
- Add a design badge: `<div class="design-badge">FeatureName — prototype</div>` (see tokens.css)
- For app shell: include a static sidebar + topbar placeholder so the prototype feels real
- If the feature is inside the patient detail or other parent context, show that context

### MD spec (`spec.md`)

Required sections:

```md
# FeatureName — Design Specification
> **Prototype**: `.designs/feature-name/index.html`
> **Target**: `app/pages/...` or `app/components/...`

## Design → Code alignment (if applicable)
| Design element | In current code? | Action |
|---|---|---|

## Component tree
## States table
| Component | Loading | Empty | Error | Populated |
## DS component mapping
| UI element | Existing component / token |
## Interaction flows
## Layout
## Accessibility notes
## Open UX questions
```

---

## Conventions

### File naming
- Each feature gets a folder: `.designs/<feature-name>/`
- Inside: `index.html` + `spec.md`
- No version suffixes — git handles history
- New flow that diverges → new folder

### Token usage
```css
/* ✅ Correct */
color: hsl(var(--agala-foreground));
background: hsl(var(--agala-primary) / 0.1);
border-radius: var(--agala-radius);
font-family: var(--agala-font-display);

/* ❌ Wrong */
color: #1a1a2e;
background: rgba(26, 168, 150, 0.1);
border-radius: 8px;
```

### tokens.css
- Lives at `.designs/tokens.css`
- Linked from all prototypes via `<link rel="stylesheet" href="../tokens.css">`
- **Ask the user periodically if tokens.css needs updating** — new tokens may be needed as designs evolve
- Mirror of the project's real design tokens (`--agala-*` HSL values)

### flows.json sync
- Lives at `.designs/flows.json`
- Documents architecture: nodes (pages, components, stores, externals) and flows (step-by-step data paths)
- **After any design change that affects architecture** — new component, moved responsibility, changed data flow — update `flows.json`:
  - Add/remove nodes as needed
  - Update flow steps to reflect new data paths
  - The `flows.html` visualizer reads this file — no HTML changes needed
- **Ask the user**: "¿Actualizo los flows con este cambio?"

### Component states
Every component that loads data must describe all four states:
| State | What to show |
|---|---|
| **Loading** | Skeleton/spinner. Use shimmer rectangles, not just a spinner. |
| **Empty** | Meaningful empty state with action. "No hay turnos para hoy. ¿Querés ver la semana?" |
| **Error** | Alert banner with retry action. Never just "Error." |
| **Populated** | The normal state with real-looking mock data. |

### Mock data
- Use realistic Argentine names (María Gómez, Carlos Ruiz, etc.)
- Use realistic dates (relative to "today" in the prototype)
- Use realistic dental terminology
- Pre-populate with enough data to show the design working

### Responsive
- Sidebar hides on mobile (`display: none` at ≤640px)
- Tables become cards on mobile (each row → bordered card)
- Hover actions become always-visible on touch devices
- Buttons: full-width, min-height 2.5rem on mobile
- Filter chips: horizontal scroll, hide scrollbar
- Calendar/charts: horizontal scroll container, hide scrollbar

---

## What NOT to do

- ❌ Don't write Vue/React/TypeScript code in prototypes
- ❌ Don't hardcode hex colors or pixel values
- ❌ Don't invent new design patterns if an existing component covers the case
- ❌ Don't design in a vacuum — always read the existing code first
- ❌ Don't skip the PM step for vague requests
- ❌ Don't create prototypes without the MD spec
- ❌ Don't forget responsive breakpoints
- ❌ Don't put dictation/voice features unless the user explicitly asks for them
- ❌ Don't assume data models — read the actual `/app/models/` files
- ❌ Don't create circular references between design files
- ❌ Don't add v1/v2 suffixes — git handles that
- ❌ Don't forget to update `flows.json` when architecture changes (new components, new data flows, changed responsibilities)

---

## Integration with other skills

- **Before designing**: load `agala-ui` skill to know available components and tokens
- **For Nuxt pages**: reference `nuxt-pinia-store` and `nuxt-server-route` patterns
- **For Vue SPA**: reference `vue-repository-method` for data fetching patterns
- **After approval**: invoke `feature-planner` with the UI spec + PRD to create implementation plan
