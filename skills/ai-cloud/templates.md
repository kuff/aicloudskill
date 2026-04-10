# AI Cloud — Batch Script Templates

## 1. Pull a Container from NVIDIA NGC

```bash
#!/bin/bash
#SBATCH --job-name=pull_container
#SBATCH --output=pull_%j.out
#SBATCH --error=pull_%j.err
#SBATCH --cpus-per-task=32
#SBATCH --mem=60G
#SBATCH --time=01:00:00

export SINGULARITY_TMPDIR=$HOME/.singularity/tmp
export SINGULARITY_CACHEDIR=$HOME/.singularity/cache
mkdir -p $SINGULARITY_TMPDIR $SINGULARITY_CACHEDIR

# PyTorch example — change the tag as needed
singularity pull pytorch_24.09-py3.sif docker://nvcr.io/nvidia/pytorch:24.09-py3

# TensorFlow example:
# singularity pull tensorflow_24.11-tf2-py3.sif docker://nvcr.io/nvidia/tensorflow:24.11-tf2-py3
```
Submit: `sbatch pull_container.sh`

## 2. Simple Single-GPU Training Job

```bash
#!/bin/bash
#SBATCH --job-name=train
#SBATCH --output=train_%j.out
#SBATCH --error=train_%j.err
#SBATCH --gres=gpu:1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=12:00:00

singularity exec --nv pytorch_24.09-py3.sif python3 train.py
```

## 3. Multi-GPU DDP Training (4 GPUs)

```bash
#!/bin/bash
#SBATCH --job-name=ddp_train
#SBATCH --output=ddp_%j.out
#SBATCH --error=ddp_%j.err
#SBATCH --gres=gpu:4
#SBATCH --ntasks=4
#SBATCH --cpus-per-task=8
#SBATCH --mem=60G
#SBATCH --time=24:00:00

srun singularity exec --nv pytorch_24.09-py3.sif \
    python3 train_ddp.py --world_size=4
```

### Minimal DDP Python Boilerplate

```python
import os
import torch
import torch.distributed as dist
import torch.nn as nn
from torch.utils.data import DataLoader
from torch.utils.data.distributed import DistributedSampler

def setup(rank, world_size):
    os.environ['MASTER_ADDR'] = 'localhost'
    os.environ['MASTER_PORT'] = '12355'
    dist.init_process_group("nccl", rank=rank, world_size=world_size)
    torch.cuda.set_device(rank)

def cleanup():
    dist.destroy_process_group()

def train(rank, world_size):
    setup(rank, world_size)

    model = YourModel().to(rank)
    model = nn.parallel.DistributedDataParallel(model, device_ids=[rank])

    sampler = DistributedSampler(dataset, num_replicas=world_size, rank=rank)
    loader = DataLoader(dataset, batch_size=64, sampler=sampler, num_workers=4)

    for epoch in range(num_epochs):
        sampler.set_epoch(epoch)  # Important for proper shuffling
        for batch in loader:
            # ... training step ...
            pass

    cleanup()
```

## 4. Request Specific GPU Type (e.g. L40s)

```bash
#!/bin/bash
#SBATCH --job-name=l40s_job
#SBATCH --output=l40s_%j.out
#SBATCH --error=l40s_%j.err
#SBATCH --gres=gpu:l40s:2
#SBATCH --cpus-per-task=16
#SBATCH --mem=64G
#SBATCH --time=08:00:00

singularity exec --nv pytorch_24.09-py3.sif python3 train.py
```

## 5. Build a Custom Conda Container with Cotainr

### environment.yml
```yaml
name: myenv
channels:
  - conda-forge
  - pytorch
dependencies:
  - pytorch
  - torchvision
  - pip
  - pip:
    - transformers
    - datasets
```

### build.sh
```bash
#!/bin/bash
#SBATCH --job-name=build_container
#SBATCH --output=build_%j.out
#SBATCH --error=build_%j.err
#SBATCH --cpus-per-task=32
#SBATCH --mem=80G
#SBATCH --time=04:00:00

export SINGULARITY_TMPDIR=$HOME/.singularity/tmp
export SINGULARITY_CACHEDIR=$HOME/.singularity/cache
mkdir -p $SINGULARITY_TMPDIR $SINGULARITY_CACHEDIR

# cotainr handles --fakeroot internally; you don't need to pass it.
/home/container/cotainr build myenv.sif \
    --base-image=docker://ubuntu:24.04 \
    --conda-env=environment.yml \
    --accept-licenses
```

### Building from a `.def` file (non-cotainr)

For full Singularity definition files, use `singularity build` directly. **`--fakeroot` is required** for unprivileged users on this cluster:

```bash
#!/bin/bash
#SBATCH --job-name=build_def
#SBATCH --output=build_%j.out
#SBATCH --error=build_%j.err
#SBATCH --cpus-per-task=32
#SBATCH --mem=80G
#SBATCH --time=04:00:00

export SINGULARITY_TMPDIR=$HOME/.singularity/tmp
export SINGULARITY_CACHEDIR=$HOME/.singularity/cache
mkdir -p $SINGULARITY_TMPDIR $SINGULARITY_CACHEDIR

singularity build --fakeroot my_image.sif my_image.def
```

## 6. LLM Inference with vLLM

```bash
#!/bin/bash
#SBATCH --job-name=vllm_infer
#SBATCH --output=vllm_%j.out
#SBATCH --error=vllm_%j.err
#SBATCH --gres=gpu:1
#SBATCH --mem=40G
#SBATCH --time=04:00:00

singularity exec --nv /home/container/vllm-openai_latest.sif python3 inference.py
```

### Large Model with Tensor Parallelism (4 GPUs)
```bash
#!/bin/bash
#SBATCH --job-name=vllm_large
#SBATCH --output=vllm_large_%j.out
#SBATCH --error=vllm_large_%j.err
#SBATCH --gres=gpu:l40s:4
#SBATCH --mem=120G
#SBATCH --time=04:00:00

singularity exec --nv /home/container/vllm-openai_latest.sif python3 -c "
from vllm import LLM, SamplingParams
import os
os.environ['HUGGING_FACE_HUB_TOKEN'] = os.getenv('HF_TOKEN')
llm = LLM(model='meta-llama/Llama-3.3-70B-Instruct', tensor_parallel_size=4)
# ... your inference code ...
"
```

## 7. Short Iteration Batch Job

Use this **instead of an interactive session** when you want to iterate on a script.
AAU AI Cloud explicitly prohibits interactive development sessions (`srun --pty`,
Jupyter, VS Code Remote SSH) — see https://hpc.aau.dk/ai-cloud/fair-usage/. Submit
short batch jobs in a tight loop instead: edit, `sbatch`, read the log, repeat.

```bash
#!/bin/bash
#SBATCH --job-name=iter
#SBATCH --output=iter_%j.out
#SBATCH --error=iter_%j.err
#SBATCH --gres=gpu:1
#SBATCH --cpus-per-task=8
#SBATCH --mem=16G
#SBATCH --time=00:10:00

# A single experiment / smoke test. Keep it short so the queue stays responsive.
singularity exec --nv pytorch_24.09-py3.sif python3 run.py "$@"
```
Submit: `sbatch iter.sh` then `tail -f iter_<jobid>.out` after `squeue --me` shows it running.

## 8. Hugging Face Token Setup

Required for gated models (Llama, etc.). Run once on the front-end:
```bash
echo 'export HF_TOKEN="YOUR_TOKEN_HERE"' >> ~/.bashrc
source ~/.bashrc
```
Verify: `echo $HF_TOKEN`

Alternatively, `huggingface-cli login` inside a container also works and stores the token under `~/.cache/huggingface/token`.

## 9. Job Array (Hyperparameter Sweep)

Run the same script over multiple configs in parallel. Slurm picks how many run at once based on free GPUs.

```bash
#!/bin/bash
#SBATCH --job-name=sweep
#SBATCH --output=sweep_%A_%a.out
#SBATCH --error=sweep_%A_%a.err
#SBATCH --gres=gpu:1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=02:00:00
#SBATCH --array=0-9

CONFIGS=(config_0.yaml config_1.yaml config_2.yaml config_3.yaml config_4.yaml \
         config_5.yaml config_6.yaml config_7.yaml config_8.yaml config_9.yaml)

singularity exec --nv pytorch_24.09-py3.sif \
    python3 train.py --config "${CONFIGS[$SLURM_ARRAY_TASK_ID]}"
```
Submit: `sbatch sweep.sh`. Use `%A` (array job ID) and `%a` (task index) in `--output`/`--error` so each task gets its own log. Cancel a single task with `scancel <jobid>_<taskid>`.

## 10. Scheduled Off-Peak Job

For long-running jobs, ask Slurm to start them during low-demand hours. Useful for being a good cluster citizen — see https://hpc.aau.dk/ai-cloud/fair-usage/.

```bash
#!/bin/bash
#SBATCH --job-name=overnight
#SBATCH --output=overnight_%j.out
#SBATCH --error=overnight_%j.err
#SBATCH --gres=gpu:1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=12:00:00
#SBATCH --begin=now+4hours

singularity exec --nv pytorch_24.09-py3.sif python3 train.py
```
Or pass `--begin` to `sbatch` directly (no `#SBATCH` line needed):
```bash
sbatch --begin=2026-04-11T02:00:00 overnight.sh
```
