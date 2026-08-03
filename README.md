# SRE Runbooks

A collection of real incidents I've diagnosed and resolved, documented as runbooks: symptom, diagnosis path, root cause, fix, and the lesson that came out of it.

Most of these come from hands-on lab environments (KodeKloud-style production simulations) built to mirror real on-call scenarios, not toy tutorials. Each one follows the same structure a production incident report would, so the diagnostic process is reusable, not just the fix.

## Index

| Runbook | System | Root Cause Category |
|---|---|---|
| [MariaDB Service Failure](./mariadb-service-failure.md) | MariaDB / systemd | Filesystem permissions |
| [EC2 Disk Exhaustion During npm install](./ec2-disk-exhaustion-vs-ram-pressure.md) | AWS EC2 / EBS | Storage capacity, IaC |
| [Tomcat Connector Bind Failure](./tomcat-connector-bind-failure.md) | Tomcat / systemd | Port conflict, misleading service status |

More incidents get added here as they're worked through. Each file is self-contained, pull up the one relevant to what you're debugging.

## Why this exists

Most portfolios show finished code. This shows the part that doesn't fit in a repo, the process of finding out *why* something broke before fixing it. That's most of the actual job.

## Format

Every runbook follows the same shape:

- **Symptom** — what was actually observed, in the words a user or monitoring system would report it
- **Diagnosis Path** — the commands run, in order, and what each one ruled in or out
- **Root Cause** — the actual underlying issue, not just the surface error
- **Fix Applied** — the exact remediation, with commands
- **Key Lesson** — the thing worth remembering next time, usually not the fix itself