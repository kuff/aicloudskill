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
#SBATCH --mem=60G
#SBATCH --time=04:00:00

export SINGULARITY_TMPDIR=$HOME/.singularity/tmp
export SINGULARITY_CACHEDIR=$HOME/.singularity/cache
mkdir -p $SINGULARITY_TMPDIR $SINGULARITY_CACHEDIR

/home/container/cotainr build myenv.sif \
    --base-image=docker://ubuntu:24.04 \
    --conda-env=environment.yml \
    --accept-licenses
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

## 7. Interactive Session

```bash
# Get an interactive shell with 1 GPU
srun --gres=gpu:1 --mem=32G --cpus-per-task=8 --time=02:00:00 --pty bash

# Interactive shell inside a container
srun --gres=gpu:1 --mem=32G --cpus-per-task=8 --time=02:00:00 \
    --pty singularity shell --nv pytorch_24.09-py3.sif
```

## 8. Hugging Face Token Setup

Required for gated models (Llama, etc.). Run once on the front-end:
```bash
echo 'export HF_TOKEN="YOUR_TOKEN_HERE"' >> ~/.bashrc
source ~/.bashrc
```
Verify: `echo $HF_TOKEN`
