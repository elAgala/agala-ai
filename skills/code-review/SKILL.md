---
name: code-review
description: >
  Run a PR review checklist covering types, Vue template logic, error handling,
  secrets, and test coverage. Triggered by phrases like "review this PR",
  "can you review my changes", "checklist before merging", "review this diff",
  "is this ready to merge".
---

## What It Does

Applies a structured checklist to catch common issues before merging: type safety,
Vue template hygiene, error handling, hardcoded secrets, and test coverage.

---

## When to Use It

- Before opening a PR
- When reviewing someone else's code
- After finishing a feature and doing a final self-review

---

## Checklist

### Types

- [ ] No `any` unless strictly necessary and documented with a comment explaining why
- [ ] All function parameters and return types are explicit
- [ ] API response shapes are typed — no untyped `fetch` results
- [ ] No type assertions (`as SomeType`) that could hide runtime mismatches

### Vue Templates

- [ ] No business logic in templates — computed properties or composables instead
- [ ] No inline complex expressions (`v-if="items.filter(...).length > 0"`)
- [ ] All `v-for` have a `:key` with a stable, unique value (not array index unless list is static)
- [ ] Props are typed with `defineProps<{...}>()`
- [ ] Emits are typed with `defineEmits<{...}>()`

### Error Handling

- [ ] Async operations have error handling — no naked `await` in components or services
- [ ] Errors shown to the user are human-readable — no raw error objects in the UI
- [ ] Backend errors return consistent shapes (e.g., `{ error: string }`)
- [ ] Go: all returned errors are handled — no `_` discarding errors from fallible calls

### Security

- [ ] No hardcoded secrets, tokens, passwords, or API keys
- [ ] No `.env` files committed
- [ ] User input is validated before use (Zod on the frontend, binding validation in Go)
- [ ] No `console.log` with sensitive data left in the code

### Tests

- [ ] If a test file exists for the modified file, it has been updated
- [ ] New logic paths have test coverage
- [ ] No mocks introduced for things that do not need mocking

### General

- [ ] No unused imports or variables
- [ ] No dead code left behind
- [ ] Functions have a single clear responsibility
- [ ] No `continue` or `break` in loops
- [ ] Conditional nesting does not exceed 2 levels

---

## What to Avoid

- Approving PRs with `any` types without a documented reason
- Leaving `TODO` or `FIXME` comments in code going to `main` without a linked issue
- Skipping the test update because "it's a small change"
- Merging without checking for hardcoded credentials
