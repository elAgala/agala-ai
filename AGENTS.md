# Global Coding Principles

These principles apply to every project, regardless of stack or language.

---

## Communication

- Always respond in English, regardless of the language the user writes in.

---

## Style and Readability

- Write clean, readable code. If something needs a comment to be understood, first try renaming or restructuring it.
- Do not use `continue` or `break` in loops — rewrite the logic to avoid them.
- Keep functions short with a single responsibility. If a function does more than one thing, split it.
- Use descriptive names. Never use `data`, `info`, `temp`, `aux`, or `res` as final variable names.
- Do not nest more than 2 levels of conditional logic — use early returns instead.

---

## Philosophy

- Do not over-engineer. The simplest solution that solves the problem is the correct one.
- Do not add abstractions "just in case" — only when there is a concrete case that justifies it.
- Do not add dependencies without thinking. If it can be done in 10 lines, do not add a package.
- Before creating something new, check if it already exists in the project.

---

## Security

- Never hardcode secrets, credentials, or API keys.
- Never force push to `main` or `master`.

---

## Tests

- If a test file exists for the file being modified, update it.
- Do not mock what does not need to be mocked.
