# AI Cloud Skill for Claude Code

A [Claude Code](https://claude.com/claude-code) skill that lets Claude interface with the [AAU AI Cloud](https://hpc.aau.dk/ai-cloud/) GPU compute cluster. Claude learns to generate correct Slurm commands, write batch scripts, manage Singularity containers, and help you run GPU workloads — all with knowledge of the cluster's specific hardware, partitions, and policies.

## What it does

- Generates correct `srun` / `sbatch` commands with proper resource flags
- Writes ready-to-submit batch scripts for training, inference, container builds
- Knows the full hardware inventory (27 nodes, 7 GPU types from T4 to A100)
- Understands Singularity container workflows (NGC pulls, cotainr, vLLM)
- Knows resource quotas, partitions, fair-usage policies, and time limits
- Can SSH into the cluster to check job status, transfer files, read logs
- Warns about phone-based MFA before initiating SSH connections
- Includes templates for multi-GPU DDP training, LLM inference, and more

## Installation

Copy the `skills/ai-cloud` directory into your Claude Code skills folder:

```bash
# Personal install (available in all your projects)
mkdir -p ~/.claude/skills
cp -r skills/ai-cloud ~/.claude/skills/

# Project install (available only in a specific project)
mkdir -p .claude/skills
cp -r skills/ai-cloud .claude/skills/
```

## Usage

The skill activates automatically when you mention AI Cloud, Slurm, GPU jobs, batch scripts, CLAAUDIA, AAU HPC, or related topics.

You can also invoke it directly:

```
/ai-cloud how do I run a 4-GPU PyTorch training job?
/ai-cloud write a batch script to pull the latest PyTorch NGC container
/ai-cloud check my job status
```

## File Structure

```
skills/ai-cloud/
├── SKILL.md        # Main skill — quick reference, rules, and instructions
├── reference.md    # Full hardware specs, node table, partitions, SSH config
└── templates.md    # 8 ready-to-use batch script templates
```

## Requirements

- [Claude Code](https://claude.com/claude-code) CLI
- AAU AI Cloud access (researchers — request via the [service portal](https://hpc.aau.dk/ai-cloud/how-to-access/))
- AAU network connectivity (campus, VPN, or SSH gateway)

## License

MIT
