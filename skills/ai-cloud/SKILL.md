---
name: ai-cloud
description: >
  AAU AI Cloud compute cluster interface. Use when the user mentions HPC, Slurm,
  GPU jobs, batch scripts, sbatch, srun, compute cluster, AI Cloud, CLAAUDIA,
  AAU HPC, Singularity containers on the cluster, training on GPUs, or anything
  related to running workloads on the AAU AI Cloud.
argument-hint: "[command or question]"
allowed-tools: Bash(ssh *ai-fe02.srv.aau.dk*), Bash(ssh *sshgw.aau.dk*), Bash(scp *ai-fe02.srv.aau.dk*), Bash(scp *sshgw.aau.dk*), Bash(rsync *ai-fe02.srv.aau.dk*), Bash(rsync *sshgw.aau.dk*), Read, Write, Edit, Grep, Glob
---

# AAU AI Cloud — Cluster Interface

You are helping the user work with the AAU AI Cloud GPU cluster. This cluster uses
**Slurm** for job scheduling and **Singularity** containers for software environments.
All compute must go through Slurm — never run workloads directly on the front-end node.

## Name Aliases
The user may refer to this cluster as **AI Cloud**, **AAU HPC**, or **CLAAUDIA**.
These all mean the same thing: the AAU AI Cloud GPU cluster documented here.
Always treat these names as synonymous.

## Quick Reference

### Connection
- **Front-end node**: `ai-fe02.srv.aau.dk`
- **SSH login**: `ssh -l <user>@domain.aau.dk ai-fe02.srv.aau.dk`
- **Via SSH gateway** (off-campus): `ssh -J <user>@domain.aau.dk@sshgw.aau.dk -l <user>@domain.aau.dk ai-fe02.srv.aau.dk`
- **Requires**: AAU network (campus or VPN) or SSH gateway
- **MFA**: SSH login requires two-factor authentication via a push notification
  on the user's phone. **Before running any SSH command, warn the user** that they
  will receive a notification on their phone and must approve it to complete login.
- The user may have an SSH config alias (e.g. `ssh aicloud`) — ask if unsure.

### Core Slurm Commands
| Command | Purpose |
|---------|---------|
| `srun [flags] <command>` | Run interactive/single-shot job |
| `sbatch <script.sh>` | Submit batch script to queue |
| `squeue --me` | Show user's queued/running jobs |
| `scancel <jobid>` | Cancel a job |
| `scancel --user=$USER` | Cancel all user's jobs |

### Resource Allocation Flags
| Flag | Example | Notes |
|------|---------|-------|
| `--gres=gpu:<N>` | `--gres=gpu:1` | Request N GPUs |
| `--gres=gpu:<type>:<N>` | `--gres=gpu:l40s:4` | Request N GPUs of specific type |
| `--cpus-per-task=<N>` | `--cpus-per-task=8` | CPU cores per task |
| `--mem=<size>` | `--mem=60G` | Memory allocation |
| `--time=<limit>` | `--time=08:00:00` | Max wall time (HH:MM:SS or D-HH:MM:SS) |
| `--ntasks=<N>` | `--ntasks=4` | Number of parallel tasks |
| `--job-name=<name>` | `--job-name=train` | Job name |
| `--output=<file>` | `--output=result_%j.out` | Stdout file (%j = job ID) |
| `--error=<file>` | `--error=error_%j.err` | Stderr file (%j = job ID) |

### Running Containers
All software runs inside Singularity containers. Key patterns:
```bash
# Interactive: run a command in a container with GPU support
srun --gres=gpu:1 singularity exec --nv <image.sif> python3 script.py

# Interactive shell inside a container
srun --gres=gpu:1 --pty singularity shell --nv <image.sif>

# Batch: submit a script that runs inside a container
sbatch job.sh   # where job.sh calls singularity exec --nv ...
```
The `--nv` flag is **required** to expose NVIDIA GPUs inside the container.

### Pre-built Containers
- **vLLM**: `/home/container/vllm-openai_latest.sif`
- **PyTorch/TensorFlow**: Pull from NVIDIA NGC (see templates)
- **Custom**: Build with cotainr at `/home/container/cotainr` (version 2024.10.0)

### Storage
- Ceph-based network filesystem shared across all nodes
- Home directory accessible from front-end and all compute nodes
- **Not for long-term research data storage** — only Level 1 (non-confidential) data

### Resource Quotas
| Tier | Max Simultaneous Jobs | Max GPUs | Notes |
|------|----------------------|----------|-------|
| Default | 12 | 12 | No application needed |
| Deadline | 24 | 24 | 14-day approval, reapply after 14-day wait |
| Unprivileged | Unlimited | Unlimited | Jobs are **preemptible** |
| AI Centre | Unlimited | Unlimited | Pioneer Centre affiliation required |

### Time Limits
- `prioritized` partition: up to **6 consecutive days** by default
- Always set `--time` explicitly so jobs can start before maintenance windows
- Jobs without `--time` may be delayed if a service window is approaching

## Important Rules

1. **Always use `--nv`** with `singularity exec`/`shell` when GPUs are needed.
2. **Set Singularity temp/cache dirs** when pulling containers:
   ```bash
   export SINGULARITY_TMPDIR=$HOME/.singularity/tmp
   export SINGULARITY_CACHEDIR=$HOME/.singularity/cache
   mkdir -p $SINGULARITY_TMPDIR $SINGULARITY_CACHEDIR
   ```
3. **Request only the GPUs you need.** Over-requesting causes longer queue times and wastes resources.
4. **Use `%j` in output filenames** (e.g. `result_%j.out`) so logs don't overwrite across runs.
5. **Container pulls are memory-intensive.** Allocate `--mem=60G` and `--cpus-per-task=32` when pulling.
6. **Only Level 1 (non-confidential) data** is permitted on the cluster.
7. **Not for CPU-only tasks** — this cluster is GPU-focused.

## How to Help the User

When the user asks about AI Cloud:

1. **Warn about phone authentication** — before running any `ssh`, `scp`, or `rsync` command to the cluster, tell the user: "You'll receive a push notification on your phone to approve the login." Wait for them to be ready.
2. **Generate correct commands** — use the syntax and flags documented above. Never guess Slurm flags.
3. **Write batch scripts** — produce complete, ready-to-submit `.sh` files with proper `#SBATCH` directives.
4. **Check job status** — use SSH to run `squeue --me` on the front-end if the user wants to know their job status.
5. **Help with containers** — guide pulling from NGC, building with cotainr, or running existing `.sif` files.
6. **Debug failures** — read error logs, suggest fixes for common issues (OOM, time limits, container problems).
7. **Transfer files** — use `scp` or `rsync` with the front-end hostname.

For detailed hardware specs, see [reference.md](reference.md).
For ready-to-use batch script templates, see [templates.md](templates.md).

$ARGUMENTS
