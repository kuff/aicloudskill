---
name: ai-cloud
description: >
  AAU AI Cloud compute cluster interface. Use when the user mentions HPC, Slurm,
  GPU jobs, batch scripts, sbatch, srun, compute cluster, AI Cloud, CLAAUDIA,
  AAU HPC, Singularity containers on the cluster, training on GPUs, or anything
  related to running workloads on the AAU AI Cloud.
argument-hint: "[command or question]"
allowed-tools: Bash(ssh aicloud*), Bash(ssh -O *aicloud*), Bash(scp *aicloud:*), Bash(rsync *aicloud:*), Bash(mkdir -p */.ssh/sockets*), Bash(ssh *ai-fe02.srv.aau.dk*), Bash(ssh *sshgw.aau.dk*), Bash(scp *ai-fe02.srv.aau.dk*), Bash(scp *sshgw.aau.dk*), Bash(rsync *ai-fe02.srv.aau.dk*), Bash(rsync *sshgw.aau.dk*), Read, Write, Edit, Grep, Glob
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

### Connection & Authentication

SSH to the cluster requires MFA (phone push notification). To avoid repeated MFA
prompts, this skill uses **SSH ControlMaster** to multiplex all commands over a
single authenticated connection. The user authenticates **once per session**; all
subsequent SSH commands reuse that connection automatically.

- **Front-end node**: `ai-fe02.srv.aau.dk`
- **SSH alias**: `ssh aicloud` (configured via `~/.ssh/config`)
- **Requires**: AAU network (campus or VPN) or SSH gateway

#### SSH Config for ControlMaster

The user's `~/.ssh/config` must have an entry for the cluster with ControlMaster
settings. The entry must include `aicloud` as an alias (so agent commands and
`allowed-tools` patterns match consistently). Example:

```
Host aicloud
    HostName ai-fe02.srv.aau.dk
    User <user>@domain.aau.dk
    ControlMaster auto
    ControlPath ~/.ssh/sockets/aicloud
    ControlPersist 8h
    # Uncomment if off-campus:
    # ProxyJump <user>@domain.aau.dk@sshgw.aau.dk
```

Users may already have an entry under a different name (e.g. `Host AI-Cloud`).
That's fine — add `aicloud` as an additional alias: `Host aicloud AI-Cloud`.

The socket directory `~/.ssh/sockets/` must exist.

#### Before Any SSH Command — Connection Check Procedure

**You MUST follow this procedure before the first SSH command in a conversation:**

1. **Check for an active connection:**
   ```bash
   ssh -O check aicloud 2>&1
   ```
   - If this returns `Master running`, skip to step 5.
   - If this returns `No ControlPath specified` or `No such name`, the SSH config
     needs setup — go to step 2.
   - Other errors likely indicate a network issue.

2. **Discover existing SSH config:** Read `~/.ssh/config` and look for any `Host`
   entry with `HostName ai-fe02.srv.aau.dk`.

   **a) Entry exists under a different name** (e.g. `Host AI-Cloud`):
   - Add `aicloud` as an additional alias: change `Host AI-Cloud` to `Host aicloud AI-Cloud`
   - Add ControlMaster settings if missing:
     ```
     ControlMaster auto
     ControlPath ~/.ssh/sockets/aicloud
     ControlPersist 8h
     ```

   **b) No entry exists:**
   - Ask the user for their AAU login (format: `username@domain.aau.dk`)
   - Create a new `Host aicloud` entry with all settings shown above

3. **Ensure socket directory exists:**
   ```bash
   mkdir -p ~/.ssh/sockets
   ```

4. **Ask the user to authenticate:** Tell the user:
   > I need an SSH connection to AI Cloud. Please run this in a **separate terminal**:
   > ```
   > ssh aicloud
   > ```
   > You'll receive a push notification on your phone — approve it to complete login.
   > Leave that terminal open and let me know once you're connected.

   After the user confirms, verify with `ssh -O check aicloud 2>&1`.

5. **Proceed.** Use `ssh aicloud <command>` for all remote commands.

#### All Remote Commands Use the `aicloud` Alias

Always use `ssh aicloud` for commands — never the raw hostname. This ensures
commands go through the ControlMaster socket and match `allowed-tools` patterns.
```bash
ssh aicloud "squeue --me"
ssh aicloud "sinfo -N --long"
scp aicloud:~/results.txt ./
rsync -avz ./data/ aicloud:~/data/
```

### Core Slurm Commands
| Command | Purpose |
|---------|---------|
| `srun [flags] <command>` | Run interactive/single-shot job |
| `sbatch <script.sh>` | Submit batch script to queue |
| `squeue --me` | Show user's queued/running jobs |
| `scancel <jobid>` | Cancel a job |
| `scancel --user=$USER` | Cancel all user's jobs |

### Common Cluster Queries

Use these tested commands — do not construct your own `sinfo`/`squeue` variants.

**Important:** `sinfo -l`/`--long` and `--Format` are **mutually exclusive**. Never combine them.

```bash
# GPU availability per node (type, total, in-use)
ssh aicloud "sinfo -N --Format=NodeHost:20,StateLong:12,Gres:25,GresUsed:25"

# Overall node status
ssh aicloud "sinfo -N --long"

# Who is using GPUs right now
ssh aicloud "squeue --format='%.8i %.12j %.10u %.6D %.5C %.15R %.12b %.10T' --sort=-b"

# My jobs
ssh aicloud "squeue --me"
```

**Reading GPU availability output:** `Gres` shows total GPUs (e.g. `gpu:a40:3`),
`GresUsed` shows how many are allocated (e.g. `gpu:a40:2(IDX:0-1)`). The difference
is the number of free GPUs on that node.

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

1. **Ensure connection** — before the first SSH command, follow the **Connection Check Procedure** above. If the ControlMaster connection drops mid-session (command fails with connection error), re-run the procedure.
2. **Generate correct commands** — use the `aicloud` alias and the Slurm flags documented above. Never guess Slurm flags.
3. **Write batch scripts** — produce complete, ready-to-submit `.sh` files with proper `#SBATCH` directives.
4. **Check job status** — use SSH to run `squeue --me` on the front-end if the user wants to know their job status.
5. **Help with containers** — guide pulling from NGC, building with cotainr, or running existing `.sif` files.
6. **Debug failures** — read error logs, suggest fixes for common issues (OOM, time limits, container problems).
7. **Transfer files** — use `scp` or `rsync` with the front-end hostname.

For detailed hardware specs, see [reference.md](reference.md).
For ready-to-use batch script templates, see [templates.md](templates.md).

$ARGUMENTS
