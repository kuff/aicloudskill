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
- **Home directory**: `~` accessible everywhere — no bind mounts needed for containers
- **Network**: AAU internal — accessible from campus, AAU VPN, or via SSH gateway (`sshgw.aau.dk`)

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

## SSH Config
Add to `~/.ssh/config` (required for ControlMaster connection reuse):
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
Then connect with: `ssh aicloud` (authenticate once, reuse for 8 hours)

## Fair Usage Guidelines
- Cluster is for GPU workloads only, not CPU-only computation
- Only Level 1 (non-confidential) data permitted
- Not intended for long-term research data storage
- Request only resources your job can effectively use
- Monitor GPU utilization with `nvidia-smi` to verify usage
