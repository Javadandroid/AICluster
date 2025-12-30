
# ${\color{#00f5d4}\textsf{Ai Cluster on multiple high-end Computers}}$
## ${\color{#fee440}\textsf{3-Node RTX 5090 Slurm Cluster (RandD)}}$ 

### ${\color{#f15bb5}\textsf{What this project is}}$ 
This project turns three single-GPU workstations into a small GPU cluster for ML R&D. The cluster supports:
- Running many independent experiments concurrently (one job per GPU).
- Running one distributed training job across all three GPUs using PyTorch Distributed Data Parallel (DDP).

Slurm is used as the cluster manager and job scheduler. PyTorch DDP (launched via `torchrun`) performs the actual distributed training across nodes.

### ${\color{#f15bb5}\textsf{Why Slurm (and what it does)}}$ 
Slurm is an open-source, fault-tolerant, and scalable cluster management and job scheduling system for Linux clusters.
Slurm’s main responsibilities are:
- Allocating access to resources (nodes/GPUs/CPUs/memory) for a job.
- Launching and monitoring work on the allocated compute nodes.
- Managing queues and scheduling when multiple jobs compete for limited resources.

Slurm architecture is typically:
- `slurmctld` on the controller node (the “control plane”).
- `slurmd` on every compute node (the “execution agent”).

### ${\color{#f15bb5}\textsf{How distributed training fits in}}$ 
Slurm does not “magically parallelize” a single training script across machines.
Instead:
1) Slurm reserves the resources (3 nodes + 3 GPUs).
2) Slurm launches the processes on those nodes.
3) Your training framework (PyTorch DDP) coordinates communication and synchronization.

PyTorch supports multinode training by running `torchrun` on each machine with the same rendezvous configuration, and it also explicitly supports launching multinode jobs using a workload manager like Slurm.

Important note: PyTorch warns that multinode training can be bottlenecked by inter-node communication latency, so network speed matters.

 ## ${\color{#fee440}\textsf{Cluster design (for this hardware)}}$ 
### ${\color{#f15bb5}\textsf{Hardware (x3 identical nodes)}}$  
- CPU: Intel Core Ultra 9 285K
- RAM: 128 GB DDR5
- GPU: NVIDIA GeForce RTX 5090 (~31–32 GB VRAM class)
- Storage: 2 TB NVMe
- Network: 2.5GbE + Wi‑Fi 7 (use wired Ethernet for the cluster)

### ${\color{#f15bb5}\textsf{Recommended node roles}}$ 
- node0 (controller + login/dev):
  - Runs `slurmctld` (and optionally `slurmdbd` later).
  - Can have a Desktop environment for convenience.
- node1, node2 (compute):
  - Run `slurmd`.
  - Prefer headless/server setup for stability.

 ## ${\color{#fee440}\textsf{Operating model (what we will optimize for)}}$ 
Because the cluster network is currently 2.5GbE:
- Primary win: maximize throughput of *independent* experiments (3 concurrent GPU jobs).
- Secondary capability: enable DDP across 3 nodes for workloads that still benefit from it.

 ## ${\color{#fee440}\textsf{Roadmap}}$ 
### ${\color{#f15bb5}\textsf{Phase 1 — Make the cluster reliable}}$  
- Install the same Linux OS on all nodes (e.g., Ubuntu Server LTS).
- Ensure stable naming (hostnames + /etc/hosts or DNS) and SSH connectivity.
- Install NVIDIA driver + CUDA toolkit consistently across all nodes.
- Install Slurm with a minimal configuration:
  - node0: `slurmctld`
  - node0/node1/node2: `slurmd`
  - Configure GPU scheduling (1 GPU per node).

### ${\color{#f15bb5}\textsf{Phase 2 — Validate scheduling and GPU isolation}}$  
- Verify Slurm can allocate 1 GPU per node.
- Run:
  - Single-node GPU job.
  - Three simultaneous single-GPU jobs.
  - Multi-node “hello world” jobs for networking and launch verification.

### ${\color{#f15bb5}\textsf{Phase 3 — Validate distributed training}}$ 
- Run a small PyTorch DDP multinode training job:
  - `--nnodes=3`
  - `--nproc_per_node=1`
- Benchmark scaling vs. single-node runs and decide when multinode makes sense on 2.5GbE.

### ${\color{#f15bb5}\textsf{Phase 4 — Build a Web UI (optional)}}$  
Two sane options:
1) Use Slurm’s REST API (`slurmrestd`) as the backend API for your UI.
2) Use/extend an existing UI like Slurm-web.

Slurm provides a REST API quickstart, including JWT-based authentication/token usage, which is the recommended integration point for building custom tools.

 ## ${\color{#fee440}\textsf{References (high-level)}}$ 
- Slurm Overview (official): https://slurm.schedmd.com/overview.html 
- Slurm Quick Start (official): https://slurm.schedmd.com/quickstart.html 
- PyTorch Multinode Training (official tutorial): https://docs.pytorch.org/tutorials/intermediate/ddp_series_multinode.html  
- Slurm REST API Quick Start (official): https://slurm.schedmd.com/rest_quickstart.html 
- Slurm REST API Details / JWT headers (official): https://slurm.schedmd.com/rest.html 
- Slurm daemons overview (slurmctld/slurmd), background: https://www.wwt.com/blog/workload-management-and-orchestration-series-slurm-workload-manager 


 

s