# QVFT Git & GitHub Workflow

## Core rule

`main` is the team's approved code. Feature work happens on branches.

## Standard workflow

```text
GitHub Issue
    ↓
Claim / assign yourself
    ↓
Update local main
    ↓
Create feature branch
    ↓
Make changes
    ↓
git add
    ↓
git commit
    ↓
git push
    ↓
Pull Request
    ↓
Review
    ↓
Merge into main
```

## Start a task

```bash
git switch main
git pull
git switch -c feature/short-description
```

## During development

Check what changed:

```bash
git status
```

Stage changes:

```bash
git add .
```

Create a checkpoint:

```bash
git commit -m "feat: describe the checkpoint"
```

Push the branch:

```bash
git push -u origin feature/short-description
```

After the first push, later pushes can normally use:

```bash
git push
```

## Commit prefixes

- `feat:` new functionality
- `fix:` bug fix
- `docs:` documentation
- `test:` tests
- `refactor:` restructuring without changing behaviour
- `chore:` setup or maintenance

## Branch naming

- `feature/...`
- `fix/...`
- `docs/...`
- `test/...`
- `chore/...`

## Pull Requests

Every Pull Request should answer:

- What changed?
- Why was it needed?
- How was it tested?
- Which Issue does it close?
- Are there known limitations?

Use `Closes #123` in the PR description when applicable.

## Claiming work

GitHub Issues are the unit of ownership.

A workstream is a broad area. An Issue is the task a member claims.

To claim a task:

1. Open the Issue.
2. Use **Assignees → assign yourself**.
3. If self-assignment is unavailable, comment that you want the Issue.
4. Create a branch named for the task.
5. Open a PR when ready.

Avoid two people independently working on the same Issue unless they have agreed to collaborate.
