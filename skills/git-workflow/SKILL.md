---
name: git-workflow
description: >
  Apply conventional commits, semver tagging, and branching strategy.
  Triggered by phrases like "how should I commit this", "what branch should I use",
  "help me write a commit message", "how do I release a new version",
  "what's the commit format".
---

## What It Does

Guides commit message format, branch naming, versioning, and safe merge/push practices.

---

## When to Use It

- Writing or reviewing commit messages
- Creating a new branch for a feature, fix, or release
- Tagging a release
- Merging or pushing to shared branches

---

## Steps

### Commit Messages — Conventional Commits

Format: `<type>(<scope>): <short description>`

```
feat(auth): add JWT refresh token support
fix(cart): correct total price calculation on quantity change
chore(deps): update vite to 5.2.0
docs(readme): add local setup instructions
refactor(user): extract email validation to separate function
test(orders): add unit tests for order status transitions
```

Types:
- `feat` — new feature
- `fix` — bug fix
- `chore` — maintenance, deps, tooling
- `refactor` — code change with no behavior change
- `docs` — documentation only
- `test` — adding or updating tests
- `perf` — performance improvement

Rules:
- Subject line in lowercase, imperative mood, no period at the end
- Keep the subject under 72 characters
- Add a body if the why is not obvious from the title

### Branch Naming

```
feat/<short-description>
fix/<short-description>
chore/<short-description>
release/<version>
```

Examples:
```
feat/user-avatar-upload
fix/login-redirect-loop
chore/upgrade-go-1.22
release/1.4.0
```

### Semver Tagging

- `MAJOR.MINOR.PATCH`
- Bump `PATCH` for bug fixes
- Bump `MINOR` for new features (backwards compatible)
- Bump `MAJOR` for breaking changes

```bash
git tag -a v1.4.0 -m "release: v1.4.0"
git push origin v1.4.0
```

### Safe Push Rules

- Never force push to `main` or `master`.
- Merge feature branches via PR/MR — do not push directly to `main`.
- Rebase local branches before merging to keep history clean.

```bash
git fetch origin
git rebase origin/main
```

---

## What to Avoid

- Vague messages: `fix bug`, `wip`, `changes`, `update stuff`
- Mixing unrelated changes in a single commit
- Force pushing to `main` or `master` under any circumstances
- Committing secrets, `.env` files, or credentials
