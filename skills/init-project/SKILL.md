---
name: init-project
description: >
  Creates the AGENTS.md file at the root of the current project. For existing
  projects, inspects the codebase to detect the stack, domain terms, and deviations
  from standard conventions, then asks what to keep before writing. For new projects,
  asks the right questions and generates a starter file. Triggered by phrases like
  "init this project", "set up AGENTS.md", "create the project config", "onboard this
  project", "add AGENTS.md to this project".
---

## What It Does

Generates a project-level `AGENTS.md` that tells global agents everything they cannot
infer from the code: which stack this is, domain vocabulary, path deviations, and
project-specific constraints.

---

## When to Use It

- First time setting up a project to work with the global OpenCode config
- The project has no `AGENTS.md` yet
- Refreshing an outdated `AGENTS.md` after major structural changes

---

## Steps

### Mode detection

Check whether the project has existing code:
- If source files exist beyond config files → **existing project mode**
- If the directory is empty or only has scaffolding → **new project mode**

---

### Existing project mode

**1. Detect the stack**

Check the project root:
- `nuxt.config.ts` → Nuxt stack → load `stacks/nuxt/STACK.md` from the global config
- `/backend` + `/frontend` dirs → Go + Vue SPA stack → load `stacks/go-vue-spa/STACK.md`
- Neither → unknown stack, ask the user

**2. Scan for deviations**

Compare the actual project structure against the standard paths in `STACK.md`.
For each layer, check whether the real path matches the standard. Examples:

| Layer | Standard path | Actual path found |
|---|---|---|
| Repositories | `backend/internal/repository/` | `backend/internal/store/` |
| Views | `frontend/src/views/` | `frontend/src/pages/` |
| API routes | `server/api/` | `server/routes/` |

Also grep for:
- Repeated domain nouns in type names, file names, and route paths → candidate domain terms
- Any non-standard packages or frameworks beyond what `STACK.md` defines

**3. Ask the user — one batch of questions**

Present your findings and ask before writing anything:

```
I found the following. Tell me what to keep, correct, or add:

Stack detected: go-vue-spa

Path deviations from standard:
- Repositories are in backend/internal/store/ (standard: backend/internal/repository/)
- Views are in frontend/src/pages/ (standard: frontend/src/views/)

Domain terms I found (confirm or rename):
- Listing (property available for rent?)
- Booking (confirmed reservation?)
- Host (user who owns listings?)

Anything off-limits or special constraints I should add?
```

**4. Write `AGENTS.md`**

Only after the user confirms:

```markdown
# Project: <name>

## Stack
This project uses the **<stack-name>** stack.
Agents: read `stacks/<stack-name>/STACK.md` from the global config.

## Domain terminology
- "<Term>" = <definition confirmed by user>

## Deviations from the standard stack
- <layer>: uses `<actual path>` instead of `<standard path>`

## Constraints
- <anything the user added>
```

---

### New project mode

Ask one batch of questions:

```
Let's set up your project config. Answer what you know — skip anything not decided yet.

1. Project name?
2. Which stack? (nuxt / go-vue-spa / other)
3. Any domain terms agents should know? (e.g. "Listing = a property for rent")
4. Any paths that will differ from the standard stack structure?
5. Any files or directories that are off-limits?
6. Any other constraints?
```

Then write the `AGENTS.md` with what was answered. Leave a `<!-- TODO -->` comment for anything skipped so it is easy to fill in later.

---

## What to Avoid

- Never write `AGENTS.md` before asking the user to confirm the findings
- Never invent domain definitions — only include terms the user confirms
- Never duplicate content already in the global `AGENTS.md` or `STACK.md`
- Do not add implementation details or code examples — those live in `STACK.md`
- If `AGENTS.md` already exists, read it first and offer to update rather than overwrite
