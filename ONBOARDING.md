# QVFT Stack — Team Onboarding

Welcome to the Queen's Vertical Farming Team Stack repository.

This guide is the starting point for every new Stack member.

## 1. What this repository contains

The Stack connects the farm's sensors, controllers, software, data, vision systems, intelligence, and dashboard.

High-level architecture:

```text
Sensors / Cameras / Actuators
            ↓
      ESP32 / Arduino
            ↓
       HTTP / MQTT
            ↓
      Python / FastAPI
            ↓
 PostgreSQL / Supabase
       ↙           ↘
Intelligence     Dashboard
       ↓
Automation / Control
```

## 2. Before you start

Required for everyone:

- Git
- GitHub account with access to the QVFT organization
- Visual Studio Code

Install additional tools only for the workstream you are contributing to. Workstream-specific setup is documented in each folder.

Recommended VS Code extensions are included in `.vscode/extensions.json`.

## 3. Clone the repository

```bash
git clone https://github.com/Queens-Vertical-Farming-Team/QVFT-Stack-.git
cd QVFT-Stack-
code .
```

## 4. Never begin feature work directly on main

Start new work with:

```bash
git switch main
git pull
git switch -c feature/short-description
```

Examples:

```text
feature/esp32-temperature
feature/database-schema
feature/dashboard-temperature-card
feature/vision-capture
fix/device-timeout
docs/sensor-calibration
```

## 5. Claim an Issue before starting

Go to the repository's **Issues** tab.

1. Choose an unassigned Issue in your workstream.
2. Assign yourself in the **Assignees** section.
3. Comment if you need to coordinate with another member.
4. Create a branch for that Issue.
5. Keep the Pull Request focused on that Issue.

If GitHub does not allow you to self-assign, comment **"I'd like to take this"** and a maintainer will assign you.

## 6. Save and push your work

```bash
git status
git add .
git commit -m "feat: short description of change"
git push -u origin your-branch-name
```

A commit is a saved checkpoint. A push sends those checkpoints to GitHub.

## 7. Open a Pull Request

On GitHub, open a Pull Request from your branch into `main`.

Complete the repository Pull Request template and request review.

Do not merge significant work until another Stack member has reviewed it.

## 8. Keep your branch current

If `main` changed while you were working:

```bash
git switch main
git pull
git switch your-branch-name
git merge main
```

Resolve any conflicts, test again, commit if necessary, and push.

## 9. Security

This repository is public.

Never commit:

- passwords
- API keys
- Supabase service-role keys
- private keys
- real `.env` files
- personal information
- sensitive farm information

Use environment variables. Only `.env.example` belongs in the repository.

## 10. Where should my work go?

See [docs/workstreams/README.md](docs/workstreams/README.md).

For the complete Git workflow, see [docs/git-workflow.md](docs/git-workflow.md).
