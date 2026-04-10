# AI Cloud — System Reference

## Compute Nodes (27 total)

| Node Name | Count | CPU Cores | CPU Type | RAM | GPUs | GPU Type | GPU VRAM | Notes |
|-----------|-------|-----------|----------|-----|------|----------|----------|-------|
| a256-t4-[01-03] | 3 | 32 | AMD EPYC | 256 GB | 6 | NVIDIA T4 | 16 GB | — |
| a256-a40-[04-07] | 4 | 32 | AMD EPYC | 256 GB | 3 | NVIDIA A40 | 48 GB | — |
| i256-a10-[06-10] | 5 | 32 | Intel Xeon | 256 GB | 4 | NVIDIA A10 | 24 GB | — |
| i256-a40-[01-02] | 2 | 24 | Intel Xeon | 256 GB | 4 | NVIDIA A40 | 48 GB | 6.4 TB local disk |
| a512-l4-06 | 1 | 64 | AMD EPYC | 512 GB | 8 | NVIDIA L4 | 24 GB | — |
| a768-l40s-[01-06] | 6 | 64 | AMD EPYC | 768 GB | 8 | NVIDIA L40s | 48 GB | — |
| nv-ai-[01-03] | 3 | 48 | Intel Xeon | 1,470 GB | 16 | NVIDIA V100 | 32 GB | 30 TB local, NVLINK |
| nv-ai-04 | 1 | 128 | AMD EPYC | 980 GB | 8 | NVIDIA A100 | 40 GB | 14 TB local |

### GPU Summary
| GPU Type | VRAM | Total Count | Best For |
|----------|------|-------------|----------|
| T4 | 16 GB | 18 (3x6) | Inference, small models |
| A10 | 24 GB | 20 (5x4) | Training medium models |
| L4 | 24 GB | 8 (1x8) | Inference, medium models |
| A40 | 48 GB | 20 (4x3 + 2x4) | Training large models |
| L40s | 48 GB | 48 (6x8) | Training large models, highest availability |
| V100 | 32 GB | 48 (3x16) | Training, NVLINK for multi-GPU |
| A100 | 40 GB | 8 (1x8) | Largest models, highest performance |

### Access-Restricted Nodes
- `i256-a40-01`, `i256-a40-02`, `nv-ai-04` — research-group owned, priority access for owners
- Other users can access via `batch` partition (preemptible)

## Partitions
| Partition | Access | Preemption | Notes |
|-----------|--------|------------|-------|
| prioritized | Default | No | Up to 6-day time limit |
| batch | All users | Yes — interrupted when owners request | Access to restricted nodes |
| aicentre | Pioneer Centre | No | AI Centre nodes |
| aicentre-a100 | Pioneer Centre | No | A100 nodes |

## Network & Storage

- **Filesystem**: Ceph-based network storage, shared across front-end and all compute nodes
- **Home directory** (`~`): default **1 TB** quota, accessible everywhere — no bind mounts needed for containers. Request expansion via the Service Portal (see below).
- **Shared project storage** (`/home/project/<name>`): for collaborative use within a research group. Not for long-term archival; clean up when the project ends.
- **Network**: AAU internal — accessible from campus, AAU VPN, or via SSH gateway (`sshgw.aau.dk`)

### Local scratch (per-node, fast, ephemeral)

Only certain nodes have physically-attached scratch storage. It's much faster than the network FS but **purged after 90 days untouched** — never store anything you can't regenerate.

| Node | Local scratch |
|------|---------------|
| `i256-a40-[01-02]` | ~6.4 TB |
| `nv-ai-[02-03]` | ~30 TB |
| `nv-ai-04` | ~14 TB |

## Software Stack
- **OS**: Ubuntu Linux
- **Scheduler**: Slurm
- **Containers**: Singularity
- **Container builder**: cotainr (`/home/container/cotainr`, v2024.10.0)
- **Pre-built containers**: `/home/container/` directory (e.g. `vllm-openai_latest.sif`)
- **NVIDIA NGC**: Pull with `singularity pull docker://nvcr.io/nvidia/<framework>:<tag>`
- **Docker Hub**: Pull with `singularity pull docker://<image>:<tag>`

## File Transfer
```bash
# Upload to cluster
scp local-file.py aicloud:~/target-dir/

# Download from cluster
scp aicloud:~/result.txt ./

# Sync a directory
rsync -avz ./data/ aicloud:~/data/
```

Note: only `scp` and WinSCP are mentioned in the [official docs](https://hpc.aau.dk/ai-cloud/getting-started/file-management/). `rsync` over SSH works in practice (the cluster runs standard OpenSSH) but is not officially endorsed — use it at your own discretion.

## SSH Config
Add to `~/.ssh/config` (required for ControlMaster connection reuse).

**On-campus or AAU VPN:**
```
Host aicloud
    HostName ai-fe02.srv.aau.dk
    User <user>@domain.aau.dk
    ControlMaster auto
    ControlPath ~/.ssh/sockets/aicloud
    ControlPersist 8h
```

**Off-campus (no VPN):** route through the AAU SSH gateway with `ProxyJump`:
```
Host aicloud
    HostName ai-fe02.srv.aau.dk
    User <user>@domain.aau.dk
    ProxyJump <user>@domain.aau.dk@sshgw.aau.dk
    ControlMaster auto
    ControlPath ~/.ssh/sockets/aicloud
    ControlPersist 8h
```

Then connect with: `ssh aicloud` (authenticate once, reuse for 8 hours). The socket directory `~/.ssh/sockets/` must exist — create it with `mkdir -p ~/.ssh/sockets`.

## Service Portal

The official channel for everything that isn't `ssh`-able:

- **Access requests** (researchers — students go to AI-LAB instead)
- **Account renewals / extensions** for long-running projects
- **Custom container image requests** — the ops team can build a `.sif` and place it in `/home/container/`
- **Storage quota expansion requests** (default home is 1 TB)
- **General support and policy questions**

URL: https://serviceportal.aau.dk/ — search for "AI Cloud" or follow the link from https://hpc.aau.dk/ai-cloud/how-to-access/.

## Troubleshooting Quick Reference

| Symptom | Diagnosis | Fix |
|---------|-----------|-----|
| `ssh -O check aicloud` returns `Master running` but commands hang | Stale ControlMaster socket | `ssh -O exit aicloud`, then re-run the Connection Check Procedure in SKILL.md |
| `singularity: command not found` | You're on the front-end node; Singularity only exists on compute nodes | Wrap the command in `srun` or `sbatch` |
| `singularity build` → permission denied / unshare errors | Unprivileged build needs fakeroot | Add `--fakeroot`: `singularity build --fakeroot my.sif my.def` |
| Container pull or build OOM | Default `--mem` too small | Use `--mem=80G --cpus-per-task=32` for builds, `--mem=60G --cpus-per-task=32` for pulls |
| `libcublasLt.so.12` / CUDA libs missing inside container | `--nv` only mounts the NVIDIA driver, not the CUDA toolkit | Use an NVIDIA CUDA base image (`docker://nvidia/cuda:...` or `nvcr.io/nvidia/...`), not a plain `python:slim` |
| NCCL hangs / timeouts in multi-GPU DDP | Process group misconfigured | Verify `MASTER_ADDR`/`MASTER_PORT` are set, use `srun` (not bare `python`), confirm `--ntasks` matches `--gres=gpu:N` |
| Job stuck `PD` with reason `ReqNodeNotAvail, Reserved for maintenance` | `--time` overlaps the next maintenance window | Shorten `--time`; check the login banner for the next window date |
| Job stuck `PD` with reason `QOSMaxJobsPerUserLimit` | Hit your tier's concurrent-job cap (Default: 12) | Cancel old jobs, wait, or apply for Deadline tier |
| `i256-a40-*` rejects your job | Those are `aicentre` partition only (Pioneer Centre affiliation required) | Use `a256-a40-*` nodes in `prioritized`, or omit `--nodelist` and let Slurm pick |
| `UnicodeEncodeError: surrogates not allowed` from PDF text extraction | pypdf produces unpaired UTF-16 surrogates on math-heavy docs | `raw.encode("utf-8", errors="surrogatepass").decode("utf-8", errors="replace")` |
| `onnxruntime-gpu` silently downgrades to CPU | A dependency pulled CPU `onnxruntime` over the GPU build | Install order matters: install the dep first, then `pip install --force-reinstall --no-deps onnxruntime-gpu` |
| Files visible on front-end but not inside container | `~` auto-mounts; arbitrary paths like `/tmp` do not | Bind-mount explicitly: `singularity exec --nv -B /path:/path ...` |
| Jobs get preempted in the `batch` partition | Expected — the owning research group reclaimed the node | Use `prioritized`, or checkpoint your training |

## Fair Usage Guidelines

- **No interactive development sessions.** `srun --pty`, Jupyter, and VS Code Remote SSH for development are explicitly **prohibited** by AI Cloud policy. See https://hpc.aau.dk/ai-cloud/fair-usage/. Use short batch jobs (`templates.md` §7) instead.
- **VS Code Remote SSH is officially discouraged** for any use — it puts heavy load on the front-end node.
- Cluster is for GPU workloads only, not CPU-only computation
- Only Level 1 (non-confidential) data permitted
- Not intended for long-term research data storage
- Request only resources your job can effectively use
- Monitor GPU utilization with `nvidia-smi` to verify usage
