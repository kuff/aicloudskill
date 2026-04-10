# AAU AI Cloud Skill for Claude Code

*A Claude Code skill that turns your AI assistant into a competent AAU AI Cloud user — correct Slurm flags, ready-to-submit batch scripts, container workflows, and policy-aware behavior.*

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code Skill](https://img.shields.io/badge/Claude_Code-Skill-8A2BE2)](https://code.claude.com/docs/en/skills)

## What is this?

This is a [Claude Code](https://claude.com/claude-code) [skill](https://code.claude.com/docs/en/skills) for the [AAU AI Cloud](https://hpc.aau.dk/ai-cloud/) GPU compute cluster (also known as **CLAAUDIA** or **AAU HPC**, run by Aalborg University). It teaches Claude both *what* the cluster is — 27 nodes, seven GPU types from T4 to A100, Slurm scheduling, Singularity containers, tier-based fair-use quotas — and *how* to drive it correctly: SSH multiplexing through MFA, the right `sbatch` flags for each GPU type, container builds, and the cluster's fair-use policies (including the ones it will refuse to violate).

It's AAU-specific. If you don't have an AAU AI Cloud account, this skill won't be useful to you.

## What does this look like?

```
> /ai-cloud train a resnet50 on 2 L40s GPUs for 6 hours

I'll write you a batch script for that. Here's resnet50_train.sh:

#!/bin/bash
#SBATCH --job-name=resnet50_train
#SBATCH --output=resnet50_%j.out
#SBATCH --error=resnet50_%j.err
#SBATCH --gres=gpu:l40s:2
#SBATCH --cpus-per-task=16
#SBATCH --mem=64G
#SBATCH --time=06:00:00

singularity exec --nv pytorch_24.09-py3.sif python3 train_resnet50.py

Submit with:  ssh aicloud "cd ~/your-project && sbatch resnet50_train.sh"
Check status: ssh aicloud "squeue --me"
```

The skill knows the cluster's GPU inventory, so it picks `--gres=gpu:l40s:2` rather than guessing a generic `--gres=gpu:2`. It also won't let you accidentally do things the cluster prohibits — ask it for an interactive Jupyter session and it'll refuse and point you at the short-batch-job alternative instead.

## Features

- Generates correct `srun` / `sbatch` commands with proper resource flags
- Knows the full hardware inventory (27 nodes, 7 GPU types from T4 to A100)
- Handles SSH MFA via ControlMaster so you authenticate once per session
- 10 ready-to-submit batch script templates (single-GPU, multi-GPU DDP, NGC pulls, container builds, vLLM inference, job arrays, off-peak scheduling, …)
- Understands Singularity workflows: NGC pulls, cotainr builds, `.def` builds with `--fakeroot`, pre-built `/home/container/` images
- Knows resource quotas, partitions, fair-usage policies, time limits, and the ~4×/year maintenance windows
- Refuses workflows that violate AAU fair-use policy (no `srun --pty`, no Jupyter / VS Code Remote dev sessions)
- Built-in troubleshooting reference for the failure modes you'll actually hit (stale SSH sockets, NCCL hangs, missing CUDA libs in containers, jobs blocked by maintenance windows, …)
- Can SSH into the cluster to check job status, transfer files, and read logs

## Installation

Copy or symlink the `skills/ai-cloud` directory into your Claude Code skills folder:

```bash
# Personal install (available in all your projects)
mkdir -p ~/.claude/skills
cp -r skills/ai-cloud ~/.claude/skills/

# Project install (available only in a specific project)
mkdir -p .claude/skills
cp -r skills/ai-cloud .claude/skills/

# Or symlink for development (no need to re-copy after edits)
ln -s "$(pwd)/skills/ai-cloud" ~/.claude/skills/ai-cloud
```

## Usage

The skill auto-activates when you mention AI Cloud, CLAAUDIA, AAU HPC, Slurm, GPU jobs, batch scripts, Singularity, `sbatch`, `srun`, or any related cluster topic.

You can also invoke it explicitly:

```
/ai-cloud how do I run a 4-GPU PyTorch training job?
/ai-cloud write a batch script to pull the latest PyTorch NGC container
/ai-cloud check my job status
```

## What's in the skill

| File | What's in it |
|------|--------------|
| `skills/ai-cloud/SKILL.md` | Main entry point: connection setup, core Slurm commands, container patterns, storage, quotas, and the rules Claude must follow |
| `skills/ai-cloud/reference.md` | Hardware tables (nodes, GPU summary), partition table, SSH config (on-campus + off-campus), Service Portal links, troubleshooting quick reference, fair-use guidelines |
| `skills/ai-cloud/templates.md` | 10 ready-to-submit batch script templates: NGC pull, single-GPU train, multi-GPU DDP (with Python boilerplate), specific GPU type, cotainr + `.def` builds, vLLM single + tensor-parallel, short iteration job, HF token setup, job array, off-peak scheduling |

## Requirements

- [Claude Code](https://claude.com/claude-code) CLI
- AAU AI Cloud access (researchers — request via the [Service Portal](https://hpc.aau.dk/ai-cloud/how-to-access/))
- AAU network connectivity (campus, AAU VPN, or `sshgw.aau.dk` for off-campus users)

## Scope and non-goals

- **AAU-specific.** Hardware inventory, partition names, hostnames, and fair-use policies are hardcoded to AAU AI Cloud. Forking this for another HPC site would mean rewriting all three skill files.
- **Enforces fair-use policy.** This skill will refuse to help you set up interactive development sessions (`srun --pty`, Jupyter, VS Code Remote SSH) because cluster policy explicitly prohibits them. If you need to iterate on a script, use the short-batch-job pattern in `templates.md` §7 instead.

## Contributing

Issues and PRs welcome — see [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT
