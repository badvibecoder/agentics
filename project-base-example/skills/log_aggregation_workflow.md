---
name: log-aggregation-workflow
description: USE WHEN asked to parse system logs, application logs, journalctl output, or hardware performance metrics (CPU, memory, disk, network, temperature) and summarize, extract, or report them. Also use when writing or modifying log/metrics parsing code. Run this workflow before producing any log-derived output.
---

# Log & Hardware Metrics Aggregation Workflow

Defines the exact steps, canonical regex patterns, and output formatting for parsing system logs and hardware performance metrics in this repository. These rules exist because log content may be transmitted to an external API backend.

## 1. When to apply

Apply whenever a task involves: journalctl/syslog/dmesg/application logs, `/proc` or `/sys` telemetry, `sensors`, `df`/`iostat`/`mpstat`/`sar`/`ss` output, or any report derived from them.

## 2. Step 0 — Data-hygiene gate (always first)

1. Confirm the source is not excluded by `.claudeignore`.
2. Never paste raw log lines into the transcript. Summaries only.
3. If a log line could contain a secret (tokens, passwords, keys), redact before quoting.
4. Never write parsed log content to files outside the project, and never send it to endpoints not explicitly authorized.

## 3. Step 1 — Identify the source

- **systemd journal** → read via `journalctl --no-pager -o short-iso` (structured beats scraping).
- **syslog files** → standard syslog lines.
- **Python app logs** → Python `logging` format lines.
- **CPU** → `/proc/stat`, `mpstat -P ALL`, `sar -u`.
- **Memory** → `/proc/meminfo`.
- **Disk** → `/proc/diskstats`, `df -P`, `iostat -x`.
- **Network** → `/proc/net/dev`, `sar -n DEV`, `ss -s`.
- **Thermal** → `sensors`, `/sys/class/hwmon/hwmon*/temp*_input`.

## 4. Step 2 — Parse with canonical regexes

Use these patterns verbatim; do not improvise new ones. Named groups are required.

| Source | Pattern |
|---|---|
| systemd journal (`-o short-iso`) | `^(?P<ts>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d+)?[+-]\d{4})\s+(?P<host>[\w.-]+)\s+(?P<daemon>[\w.-]+?)(?:\[(?P<pid>\d+)\])?:\s+(?P<level>DEBUG|INFO|NOTICE|WARNING|ERROR|CRITICAL)?\s*(?P<message>.*)$` |
| syslog | `^(?P<month>[A-Z][a-z]{2})\s+(?P<day>\s?\d{1,2})\s+(?P<time>\d{2}:\d{2}:\d{2})\s+(?P<host>[\w.-]+)\s+(?P<daemon>[\w.-]+?)(?:\[(?P<pid>\d+)\])?:\s+(?P<message>.*)$` |
| python logging | `^(?P<ts>\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2},\d{3})\s+(?P<level>DEBUG|INFO|WARNING|ERROR|CRITICAL)\s+(?P<logger>[\w.]+)\s*:\s*(?P<message>.*)$` |

Numeric metrics:

- **CPU** utilization % = `100 * (1 - (idle2 - idle1) / (total2 - total1))` over a sample delta from `/proc/stat` (total = sum columns 0-7; idle = column 3 + iowait).
- **Memory** used % = `100 * (MemTotal - MemAvailable) / MemTotal` from `/proc/meminfo` (KiB).
- **Disk** utilization = `iostat -x` `%util`, or `(used / size) * 100` from `df -P`.
- **Network** = cumulative bytes/packets from `/proc/net/dev`; convert to a rate over the sample window.

## 5. Step 3 — Normalize

- Timestamps → ISO-8601 UTC `YYYY-MM-DDTHH:MM:SSZ`; convert systemd `+0000`/local offsets, and syslog month/day/time → full timestamp using the log's year (default to current year; flag ambiguity).
- Severity → canonical set: `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`. Map `notice→INFO`, `err→ERROR`, `alert→CRITICAL`, `emerg→CRITICAL`.
- Daemon names lowercase; keep the message as a single string; strip ANSI escapes and trailing whitespace.

## 6. Step 4 — Aggregate

- Group by (host, daemon, severity), then by time window (1m / 5m / 1h).
- Count occurrences; flag anomalies: severity escalations, rate spikes > 3× baseline, or metrics crossing thresholds:

| Metric | Alert threshold |
|---|---|
| CPU utilization | > 85% sustained 5m |
| Memory used | > 90% |
| Disk used | > 80% (or < 10% free) |
| Disk `%util` | > 90% |
| NIC rx/tx errors | > 0.01% of packets |
| Thermal | > 80°C |

## 7. Step 5 — Format output (mandatory)

- **Never** echo raw log lines. Summaries only.
- Prefer a markdown table: `time window | host | daemon | severity | count | notable examples`.
- Show at most 3 example messages per group, each trimmed to ~120 chars.
- End with: total lines parsed, window, alert summary, and the parse method used.
- If the user needs machine-readable output, emit JSON with the schema: `{window, source, total, groups:[{host, daemon, severity, count, examples[]}], alerts:[{metric, value, threshold}]}`.

## 8. Prohibited

- No `grep -v` chains when a structured reader exists.
- No invented regexes; use the canonical table and extend it deliberately (with tests).
- No raw-log pastes, no unredacted secret-like content, no external transmission without explicit approval.
