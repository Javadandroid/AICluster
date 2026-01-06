
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

 ## ${\color{#fee440}\textsf{Phase 1 — Make the cluster reliable (extended)}}$ 
### ${\color{#f15bb5}\textsf{Install Ubuntu Server LTS on 3 Nodes (and Desktop on the Controller)}}$ 

### ${\color{#f15bb5}\textsf{Goal}}$  
Install the same Ubuntu Server LTS on three workstations (node0/node1/node2) and optionally add a Desktop environment only on the controller (node0). [page:4][page:5]

Ubuntu Server uses a text-based installer rather than a graphical Desktop installer, which is typically a good fit for cluster nodes. [page:4]

### ${\color{#f15bb5}\textsf{Step-by-step: Ubuntu Server installation (all nodes)}}$  
#### ${\color{#90be6d}\textsf{1) Prepare installation media}}$ 
- Download the Ubuntu Server ISO (amd64 “Server install image”). [page:5]
- Create a bootable USB (this is the most common approach). [page:5]
- Boot the machine from the USB using the boot menu/BIOS key (examples include Esc/Enter/F2/F10/F12 depending on vendor). [page:5]

#### ${\color{#90be6d}\textsf{2) Run the installer}}$  

The Subiquity documentation suggests you can accept sensible defaults for a first installation: [page:5]
- Choose language. [page:5]
- Update the installer (if offered). [page:5]
- Select keyboard layout. [page:5]
- Networking: the installer tries DHCP on wired interfaces; you can continue even if networking fails and fix it later. [page:5]
- Do not set proxy or custom mirror unless required. [page:5]
- Storage: keep “Use an entire disk”, select the target disk, press Done, and confirm. [page:5]
- Enter username, hostname, and password. [page:5]
- On “SSH Setup” and “Featured Server Snaps”, select Done. [page:5]
- Reboot and log in. [page:5]

### ${\color{#f15bb5}\textsf{Add a Desktop only on node0 (controller)}}$ 
Ubuntu Server does not include a GUI installer by default. [page:4]

#### ${\color{#90be6d}\textsf{Option A: GNOME (recommended for lowest friction)}}$  
A GNOME guide for Ubuntu 24.04 shows:
- `sudo apt update`
- `sudo apt upgrade`
- Minimal GNOME install: `sudo apt install ubuntu-desktop-minimal` [page:6]
It also mentions `sudo dpkg-reconfigure gdm3` if you need to select GNOME/GDM as the default display manager/session. [page:6]

#### ${\color{#90be6d}\textsf{Option B: Cinnamon (Mint-like desktop)}}$  
A Cinnamon guide for Ubuntu 24.04 shows:
- `sudo apt update` [page:7]
- `sudo add-apt-repository universe` [page:7]
- `sudo apt install cinnamon-desktop-environment` [page:7]
Then log out/reboot and select the Cinnamon session from the login screen. [page:7]


### ${\color{#f15bb5}\textsf{Stable Hostnames, /etc/hosts, and SSH Connectivity}}$ 

### ${\color{#f15bb5}\textsf{Goal}}$ 

Make node naming stable and SSH access reliable across three Ubuntu Server nodes:
- `node0` (controller)
- `node1`, `node2` (compute)

This uses:
- Persistent hostnames via `hostnamectl`
- Name resolution via `/etc/hosts` (no DNS required)
- SSH key-based authentication for passwordless admin access

### ${\color{#f15bb5}\textsf{Prerequisites}}$  

- All three nodes have Ubuntu Server 24.04 LTS installed
- OpenSSH server is installed and running on all nodes
- You have a regular user account (not root) on each node
- Static or reserved DHCP IPs for all nodes (important: IPs must not change)

### ${\color{#f15bb5}\textsf{Step 1: Set a persistent hostname on each node}}$  

Run on each node:

```bash
# On node0
sudo hostnamectl set-hostname node0

# On node1
sudo hostnamectl set-hostname node1

# On node2
sudo hostnamectl set-hostname node2
```

Verify:

```bash
hostnamectl
# or simply:
hostname
```

**Why:** Slurm and cluster tools rely on stable hostnames for identification and communication. A hostname is also used in log files, certificates, and system configuration.

### ${\color{#f15bb5}\textsf{Step 2: Populate /etc/hosts on all nodes}}$  

Edit `/etc/hosts` on **every node**:

```bash
sudo nano /etc/hosts
```

Add or modify entries so that all three nodes can resolve each other by name. Replace the IP addresses with your actual node IPs:

```text
# Cluster nodes
192.168.1.10  node0
192.168.1.11  node1
192.168.1.12  node2

# Standard localhost mappings (should already exist)
127.0.0.1     localhost
::1           localhost
```

**Why:** Without these mappings, when Slurm, PyTorch, or SSH tries to reach a node by hostname (e.g., `node1`), it will fail because there is no DNS server to resolve it. A local `/etc/hosts` file is the simplest way to ensure name resolution works everywhere in the cluster.

**Tip:** Ubuntu Server typically includes a line like `127.0.1.1 <hostname>` at install time. Keep it (or update it to match your node's hostname) for local hostname resolution to work correctly.


### ${\color{#f15bb5}\textsf{Test name resolution}}$  

From any node, verify that all nodes can resolve each other:

```bash
ping -c 1 node0
ping -c 1 node1
ping -c 1 node2

# Also test via DNS lookup
nslookup node0
nslookup node1
nslookup node2
```

All three should respond. If any ping fails or returns "Name or service not known", check:
- Your `/etc/hosts` entries on that node
- Spelling of hostnames
- Network connectivity (can you ping by IP? `ping -c 1 192.168.1.10`)

### ${\color{#f15bb5}\textsf{Step 3: Test SSH connectivity (password-based)}}$  

From `node0`, test that you can SSH to the other nodes:

```bash
ssh youruser@node1
# You should be prompted for a password
# Type the password and press Enter
# If successful, you will see a shell prompt on node1

exit
# Return to node0

ssh youruser@node2
# Same as above
exit
```

**Why:** This confirms that SSH daemon is running, network is working, and basic authentication is possible.

**What to check if SSH fails:**
- Can you ping the node by IP? (`ping 192.168.1.11`)
- Is SSH daemon running? (On the remote node: `sudo systemctl status ssh`)
- Are there firewall rules blocking port 22? (Less common on a local network, but possible)

### ${\color{#f15bb5}\textsf{Step 4: Configure SSH key-based authentication}}$  

Key-based authentication is more secure and avoids typing passwords repeatedly during cluster administration.

#### ${\color{#90be6d}\textsf{4a) Generate an SSH key on node0}}$  

On `node0`, generate an Ed25519 key (modern, secure, and compact):

```bash
ssh-keygen -t ed25519
```

Press Enter at each prompt to accept defaults (or customize the path/passphrase if needed).

Output will look like:

```
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/youruser/.ssh/id_ed25519): [press Enter]
Enter passphrase (empty for no passphrase): [press Enter or add a passphrase]
Enter same passphrase again: [press Enter or repeat passphrase]
Your public key has been saved in /home/youruser/.ssh/id_ed25519.pub
Your private key has been saved in /home/youruser/.ssh/id_ed25519
```

**Passphrase:** You can leave it empty for passwordless SSH (common in automated clusters), or set one for extra security. For a lab cluster, empty is fine.



#### ${\color{#90be6d}\textsf{4b) Copy the public key to node1 and node2}}$  

Use `ssh-copy-id` to copy your public key to the remote nodes:

```bash
ssh-copy-id youruser@node1
# Enter your password when prompted
# The tool will append your public key to ~/.ssh/authorized_keys on node1

ssh-copy-id youruser@node2
# Enter your password when prompted
```

**What `ssh-copy-id` does:**
1. Reads your local public key (`~/.ssh/id_ed25519.pub`)
2. Connects to the remote node (uses password auth)
3. Appends the key to the remote `~/.ssh/authorized_keys` file

#### ${\color{#90be6d}\textsf{4c) Test key-based SSH login}}$  

From `node0`:

```bash
ssh youruser@node1
# Should NOT ask for a password
# If it does, check the error message (usually a key permissions issue)

exit

ssh youruser@node2
# Should NOT ask for a password

exit
```

**If it still asks for a password:**
- Check file permissions: `ls -la ~/.ssh/` (should be `700` on the directory)
- Check on the remote: `ls -la ~/.ssh/authorized_keys` (should be `600`)
- Fix with: `chmod 700 ~/.ssh` and `chmod 600 ~/.ssh/authorized_keys` on the remote

### ${\color{#f15bb5}\textsf{Step 5: (Optional but recommended) Disable password authentication in SSH}}$  

Once you confirm key-based login works **from all admin machines**, you can disable password-based SSH login for better security.

#### ${\color{#90be6d}\textsf{5a) Edit SSH daemon config on all nodes}}$   

On **each node**, edit the SSH daemon configuration:

```bash
sudo nano /etc/ssh/sshd_config
```

Find the line (it may be commented out):

```text
#PasswordAuthentication yes
```

Change it to:

```text
PasswordAuthentication no
```

Save the file (Ctrl+X, then Y, then Enter in nano).

#### ${\color{#90be6d}\textsf{5b) Restart SSH daemon}}$  

On each node:

```bash
sudo systemctl restart ssh
```

#### ${\color{#90be6d}\textsf{5c) Keep at least one SSH session open}}$ 

**Before restarting SSH, open a second terminal with an active SSH session to one of the nodes.** If you misconfigure SSH, you can still use the open session to fix it.

**After restarting, test that key-based login still works:**

```bash
ssh youruser@node1
# Should work without a password
exit
```

**If locked out:** Use the open session you left running to revert the change, then investigate.



### ${\color{#f15bb5}\textsf{Verification checklist}}$ 

Before moving to the next step (NVIDIA drivers + CUDA), verify:

- [ ] Each node has a stable hostname (`hostnamectl`)
- [ ] Each node can ping every other node by name (`ping node0`, `ping node1`, `ping node2`)
- [ ] SSH login from node0 to node1 and node2 works without a password
- [ ] (If you disabled it) Password-based SSH is disabled and key-based works

### ${\color{#f15bb5}\textsf{Summary}}$ 

You now have:
1. **Stable hostnames** (`node0`, `node1`, `node2`) that persist across reboots
2. **Local name resolution** via `/etc/hosts` (DNS-independent)
3. **Passwordless SSH** from node0 to node1 and node2 (key-based authentication)

This is the foundation for Slurm and distributed training: every tool will use hostnames, and cluster scripts can run without interactive prompts.


# 1. System update and NVIDIA prerequisites
sudo apt update && sudo apt upgrade -y
sudo apt install linux-headers-$(uname -r) build-essential dkms -y

# 2. NVIDIA driver repository and installation
sudo add-apt-repository ppa:graphics-drivers/ppa -y
sudo apt update
sudo apt install nvidia-driver-560 -y

# 3. REBOOT ALL NODES (do this after driver install)
sudo reboot

nvidia-smi

# 4. CUDA Toolkit 12.6 installation
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update
sudo apt install cuda-toolkit-12-6 -y

# 5. MUNGE authentication (Slurm prerequisite)
sudo apt install munge libmunge-dev -y

# 6. Slurm user and directories
sudo groupadd -g 1002 slurm
sudo useradd -u 1002 -g slurm -d /var/lib/slurm -s /bin/bash slurm
sudo mkdir -p /etc/slurm /var/spool/slurm/d /var/log/slurm
sudo chown slurm:slurm /var/spool/slurm/d /var/log/slurm

# 7. Install Slurm packages
sudo apt install slurm-wlm slurm-client -y

# 8. Generate MUNGE authentication key
sudo /usr/sbin/create-munge-key
sudo cp /etc/munge/munge.key /root/munge.key

# 9. Create Slurm configuration files
sudo tee /etc/slurm/slurm.conf > /dev/null << 'EOF'
ControlMachine=node0
AuthType=auth/munge
SlurmctldPort=6817
SlurmdPort=6818
StateSaveLocation=/var/spool/slurmctld
SlurmdSpoolDir=/var/spool/slurmd
SwitchType=switch/none
MpiDefault=none
SlurmctldPidFile=/var/run/slurmctld.pid
SlurmdPidFile=/var/run/slurmd.pid
ProctrackType=proctrack/cgroup
TaskPlugin=task/none
NodeName=node[0-2] Procs=64 State=UNKNOWN
PartitionName=main Nodes=node[0-2] Default=YES MaxTime=INFINITE State=UP
EOF

sudo tee /etc/slurm/gres.conf > /dev/null << 'EOF'
Name=gpu Type=rtx5090 Count=1 File=/dev/nvidia0
EOF

 ## ${\color{#fee440}\textsf{References}}$ 
- Slurm Overview (official): https://slurm.schedmd.com/overview.html 
- Slurm Quick Start (official): https://slurm.schedmd.com/quickstart.html 
- PyTorch Multinode Training (official tutorial): https://docs.pytorch.org/tutorials/intermediate/ddp_series_multinode.html  
- Slurm REST API Quick Start (official): https://slurm.schedmd.com/rest_quickstart.html 
- Slurm REST API Details / JWT headers (official): https://slurm.schedmd.com/rest.html 
- Slurm daemons overview (slurmctld/slurmd), background: https://www.wwt.com/blog/workload-management-and-orchestration-series-slurm-workload-manager 
- Install Ubuntu Server (official tutorial): https://ubuntu.com/tutorials/install-ubuntu-server [page:4]
- Basic server installation (Subiquity / Canonical): https://github.com/canonical/subiquity/blob/main/doc/howto/basic-server-installation.rst [page:5]
- Install GNOME on Ubuntu 24.04 (example guide): https://greenwebpage.com/community/how-to-install-gnome-desktop-on-ubuntu-24-04/ [page:6]
- Install Cinnamon on Ubuntu 24.04 (example guide): https://www.tecmint.com/install-cinnamon-desktop-on-ubuntu/ [page:7]

- Setting the Hostname on Ubuntu 24.04: https://linuxconfig.org/setting-the-hostname-on-ubuntu-24-04
- How to Change Hostname on Ubuntu 22.04: https://linuxize.com/post/how-to-change-hostname-on-ubuntu-22-04/

- How to Change Hostname on Ubuntu 24.04: https://docs.vultr.com/how-to-change-a-hostname-on-ubuntu-24-04
- Understanding /etc/hosts: https://linuxize.com/post/how-to-change-hostname-on-ubuntu-22-04/
- SSH Copy ID for Copying SSH Keys to Servers: https://www.ssh.com/academy/ssh/copy-id
- How to Set Up SSH Keys on Ubuntu 22.04: https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-22-04
- How to allow or disallow SSH password authentication: https://www.simplified.guide/ssh/disable-password-authentication
- Disable SSH Password Authentication For Specific User Or Group: https://ostechnix.com/disable-ssh-password-authentication-for-specific-user-or-group/

 

s