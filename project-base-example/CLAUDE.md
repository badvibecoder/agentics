# CLAUDE.md

> **Read this file first.** It defines how to work inside this repository.

**Project:** `agentics/deepseekv4-flash` — Python automation & infrastructure tooling for a small Linux fleet.

**Data-hygiene notice (read before anything else):** this repository is used with an **external API backend**. Assume every byte of context is transmitted to a third-party service and may be persisted. Treat `.claudeignore` as the hard boundary of what may ever be read. Never read, print, or summarize `.env`, secrets, private keys, or database files. If a task requires a secret, ask the user to supply a redacted value or to run the relevant command themselves via `!`.

## 1. High-Level Architecture

This repository automates fleet provisioning and runs an on-host observability agent. Three layers:

**A. Fleet provisioning (`ansible/`)**
Ansible playbooks and roles that bootstrap, harden, and update hosts, install the collector agent, and deploy configuration. All secrets are vault-encrypted; no plaintext credentials anywhere.

**B. Collector agent (`src/agent/`)**
A daemon, deployed per host, that samples hardware performance metrics and reads system logs, normalizes them into structured events, and publishes them to an external metrics API. Contains the multi-threaded dispatcher (see `skills/multithreading_rules.md`).

**C. Local CLI (`src/cli/`)**
Operator-facing tooling to inspect recent metrics, re-publish the spool, and sanity-check the agent.

### Data flow

```
[Host telemetry: /proc, /sys, journalctl, dmesg, sensors]
        │  raw samples & log lines
        ▼
[collectors/*.py]          per-source samplers (cpu, memory, disk, network, thermal)
        │
        ▼
[pipeline/parse.py]        canonical regex extraction
        ▼
[pipeline/normalize.py]    ISO-8601 UTC timestamps, canonical severities, SI units
        ▼
[dispatcher.py]            ThreadPoolExecutor fan-out, bounded queue, backpressure
        │
        ▼
[pipeline/publish.py]      batched POST → external metrics API (retry w/ backoff)
        │
        ▼
[spool.sqlite3]            local write-behind buffer (never shipped to the API)
```

### Source layout

```
ansible/                  playbooks + roles (common, agent, hardening)
src/agent/                collector daemon
src/agent/collectors/     cpu.py, memory.py, disk.py, network.py, thermal.py
src/agent/pipeline/       parse.py, normalize.py, publish.py
src/agent/dispatcher.py   thread-pool dispatch loop
src/cli/metricz.py        operator CLI
src/lib/                  logging.py, config.py
tests/                    pytest suite (parse, normalize, dispatcher)
pyproject.toml            project metadata + tool config
Makefile                  standard task runner
.env.example              env template (never a real .env)
```

## 2. Coding Conventions

### Python

- **Version:** Python 3.11+; add `from __future__ import annotations` to every module.
- **Style:** PEP 8, line length 100, enforced by `ruff`. No wildcard imports, ever.
- **Typing:** full type hints on all public functions; `mypy --strict` must pass.
- **Docstrings:** module docstring plus one per public function (Google style).
- **Errors:** never `except Exception: pass`. Catch narrowly, log with context, or re-raise.
- **Logging:** always through `src/lib/logging.py` — never `print`. Use the `RedactingFilter` for anything that could carry a secret.
- **Config:** pydantic-settings from environment / `.env` at runtime; never hardcode.
- **Secrets:** must never appear in code, logs, or commit messages.
- **Threading:** any threading/process code must comply with `skills/multithreading_rules.md`.
- **Log/metrics parsing:** any parsing code must comply with `skills/log_aggregation_workflow.md`.

### Ansible

- Ansible 2.16+; two-space indentation, no tabs, YAML 1.1.
- Every play/role task is **idempotent** — safe to run twice.
- Every task has a `name:`; every role has `meta/main.yml`.
- Secrets only via `ansible-vault`; never `vars_prompt` for plaintext passwords.
- `become: true` only where required and never with a password on the CLI.
- Tags on all tasks: `setup`, `configure`, `deploy`, `teardown`, `update`.
- Validate with `ansible-lint` before any change is accepted.

## 3. Standard Commands

| Action | Command |
|---|---|
| Create venv + install deps | `make setup` |
| Lint (ruff + mypy) | `make lint` |
| Run tests | `make test` (pytest -q) |
| Run one test file | `python -m pytest tests/test_parse.py -x -q` |
| Run agent locally | `make run-agent` |
| Dry-run playbooks | `ansible-playbook ansible/playbooks/site.yml --check --diff` |
| Lint playbooks | `ansible-lint ansible/` |
| Read CLI help | `python -m src.cli.metricz --help` |

Always run `make lint && make test` before declaring work complete.

## 4. Interaction Rules (how to work here)

1. **Scope:** never modify anything outside this project folder. No global files.
2. **Data hygiene:** before reading any file, mentally check `.claudeignore`. If in doubt, treat the file as off-limits and ask the user.
3. **Never read or echo:** `.env`, `*.pem`, `*.key`, `.ssh/`, `secrets.yml`, `*.db`, `*.sqlite*`, `*.log`. If a task needs one, ask the user to run the command themselves (`! command`) or provide a redacted excerpt.
4. **Logs & metrics:** always follow `skills/log_aggregation_workflow.md`. Never paste raw log lines into a response — summarize with the canonical patterns.
5. **Threading:** any change to threaded code follows `skills/multithreading_rules.md` and preserves the dispatcher's shutdown contract.
6. **Commands:** prefer the read-only tools (`Read`, `Grep`, `Glob`, `ls`), which are pre-approved in `.claude/settings.json`. Anything state-changing (Write/Edit/Bash) will prompt — never bypass with `--dangerously-skip-permissions`.
7. **Confirm before:** destructive shell commands, anything touching production hosts, or any command that reads outside the project root.
8. **Commits:** follow the repo's message style; never include secrets or log dumps. Only commit when asked.
9. **Done means:** lint clean, tests green, no sensitive data in the transcript.
10. **Ask when ambiguous:** if a request could involve reading a sensitive path, say so and get explicit approval first.
