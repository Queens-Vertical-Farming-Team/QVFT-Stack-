# Intelligence

This directory contains QVFT decision-support, analytics, rule engines, anomaly detection, and future ML components.

## Start here

If you are contributing to Intelligence & Decision Support, read:

- [Intelligence & Full-Stack Development Guide](../docs/workstreams/intelligence-fullstack-guide.md)
- [Workstreams Overview](../docs/workstreams/README.md)
- [Git Workflow](../docs/git-workflow.md)
- [Contributing](../CONTRIBUTING.md)

## Recommended build order

```text
Rule-based decisions
        ↓
SQL / Supabase data access
        ↓
Python decision-support engine
        ↓
FastAPI endpoints
        ↓
React + TypeScript dashboard
        ↓
Anomaly detection
        ↓
Predictive / ML experiments
```

Do not jump directly to ML.

The first goal is a reliable and explainable system that converts validated farm data into useful decisions and recommendations.

## Planned structure

As implementation begins, this directory may grow toward:

```text
intelligence/
  README.md
  rules/
  models/
  service.py
  tests/
```

Keep Intelligence logic separate from API routes and frontend components where practical.

## Important boundary

Intelligence & Decision Support should not directly control physical hardware.

It may produce:

- decisions
- alerts
- recommendations
- predictions
- proposed actions

The Automation & Control workstream is responsible for safe execution, manual override, interlocks, fail-safe behavior, and physical actuator commands.

## AI use

AI-assisted development is allowed and encouraged, but contributors are expected to understand, test, explain, and modify the code they submit.
