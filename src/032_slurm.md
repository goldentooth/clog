# Slurm

Okay, finally, geez.

So this is about [Slurm](https://slurm.schedmd.com), an open-source, highly scalable, and fault-tolerant cluster management and job-scheduling system.

> Before we get started: I want to express tremendous gratitude to Hossein Ghorbanfekr, for [this Medium article](https://medium.com/@hghcomphys/building-slurm-hpc-cluster-with-raspberry-pis-step-by-step-guide-ae84a58692d5) and [this second Medium article](https://medium.com/@hghcomphys/building-an-hpc-software-stack-with-environment-modules-4055556dc454), which helped me set up Slurm and the modules and illustrated how to work with the system and verify its functionality. I'm a Slurm newbie and his articles were invaluable.

First, we're going to set up MUNGE, which is an authentication service designed for scalability within HPC environments. This is just a matter of installing the `munge` package, synchronizing the MUNGE key across the cluster (which isn't as ergonomic as I'd like, but oh well), and restarting the service.

Slurm itself isn't too complex to install, but we want to switch off `slurmctld` for the compute nodes and on for the controller nodes.

The next part is the configuration, which, uh, I'm not going to run through here. There are a ton of options and I'm figuring it out directive by directive by reading the [documentation](https://slurm.schedmd.com/slurm.conf.html). Suffice to say that it's detailed, I had to hack some things in, and everything _appears_ to work but I can't verify that just yet.

The control nodes write state to the NFS volume, the idea being that if one of them fails there'll be a short nonresponsive period and then another will take over. It recommends _not_ using NFS, and I think it wants something like Ceph or GlusterFS or something, but I'm not going to bother; this is just an educational cluster, and these distributed filesystems really introduce a lot of complexity that I don't want to deal with right now.

Ultimately, I end up with this:

```bash
$ sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
general*     up   infinite      9   idle bettley,cargyll,dalt,erenford,fenn,gardener,harlton,inchfield,jast
debug        up   infinite      9   idle bettley,cargyll,dalt,erenford,fenn,gardener,harlton,inchfield,jast
```

```bash
$ scontrol show nodes
NodeName=bettley Arch=aarch64 CoresPerSocket=4
   CPUAlloc=0 CPUEfctv=1 CPUTot=1 CPULoad=0.84
   AvailableFeatures=(null)
   ActiveFeatures=(null)
   Gres=(null)
   NodeAddr=10.4.0.11 NodeHostName=bettley Version=22.05.8
   OS=Linux 6.12.20+rpt-rpi-v8 #1 SMP PREEMPT Debian 1:6.12.20-1+rpt1~bpo12+1 (2025-03-19)
   RealMemory=4096 AllocMem=0 FreeMem=1086 Sockets=1 Boards=1
   State=IDLE ThreadsPerCore=1 TmpDisk=0 Weight=1 Owner=N/A MCS_label=N/A
   Partitions=general,debug
   BootTime=2025-04-02T20:28:31 SlurmdStartTime=2025-04-04T12:43:13
   LastBusyTime=2025-04-04T12:43:21
   CfgTRES=cpu=1,mem=4G,billing=1
   AllocTRES=
   CapWatts=n/a
   CurrentWatts=0 AveWatts=0
   ExtSensorsJoules=n/s ExtSensorsWatts=0 ExtSensorsTemp=n/s

... etc ...
```

The next step is to set up Lua and Lmod for managing environments. Lua of course is a scripting language, and the Lmod system allows users of a Slurm cluster to flexibly modify their environment, use different versions of libraries and tools, etc by loading and unloading modules.

Setting this up isn't terribly fun or interesting. Lmod is on sourceforge, Lua is in Apt, we install some things, build Lmod from source, create some symlinks to ensure that Lmod is available in users' shell environments, and when we shell in and type a command, we can list our modules.

```bash
$ module av

------------------------------------------------------------------ /mnt/nfs/slurm/apps/modulefiles -------------------------------------------------------------------
   StdEnv

Use "module spider" to find all possible modules and extensions.
Use "module keyword key1 key2 ..." to search for all possible modules matching any of the "keys".
```

After the StdEnv, we can set up OpenMPI. OpenMPI is an implementation of Message Passing Interface (MPI), used to coordinate communication between processes running across different nodes in a cluster. It's built for speed and flexibility in environments where you need to split computation across many CPUs or machines, and allows us to quickly and easily execute processes on multiple Slurm nodes.

OpenMPI is comparatively straightforward to set up, mostly just installing a few system packages for libraries and headers and creating a module file.

The next step is setting up Golang, which is unfortunately a bit more aggravating than it should be, involving "manual" work (in Ansible terms, so executing commands and operating via trial-and-error rather than using predefined modules) because the latest version of Go in the Apt repos appears to be 1.19 but the latest version is 1.24 and I apparently need 1.23 at least to build Singularity (see next section).

Singularity is a method for running containers without the full Docker daemon and its complications. It's written in Go, which is why we had to install 1.23.0 and couldn't rest on our laurels with 1.19.0 in the Apt repository (or, indeed, 1.21.0 as I originally thought).

Building Singularity requires additional packages, and it takes quite a while. But when done:

```bash
$ module av

------------------------------------------------------------------ /mnt/nfs/slurm/apps/modulefiles -------------------------------------------------------------------
   Golang/1.21.0    Golang/1.23.0 (D)    OpenMPI    Singularity/4.3.0    StdEnv

  Where:
   D:  Default Module

Use "module spider" to find all possible modules and extensions.
Use "module keyword key1 key2 ..." to search for all possible modules matching any of the "keys".
```

Then we can use it:

```bash
$ module load Singularity
$ singularity pull docker://arm64v8/hello-world
INFO:    Converting OCI blobs to SIF format
INFO:    Starting build...
INFO:    Fetching OCI image...
INFO:    Extracting OCI image...
INFO:    Inserting Singularity configuration...
INFO:    Creating SIF file...
$ srun singularity run hello-world_latest.sif
WARNING: passwd file doesn't exist in container, not updating
WARNING: group file doesn't exist in container, not updating

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (arm64v8)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

We can also build a Singularity definition file with

```bash
$ cat > ~/torch.def << EOF
Bootstrap: docker
From: ubuntu:20.04

%post
    apt-get -y update
    apt-get -y install python3-pip
    pip3 install numpy torch

%environment
    export LC_ALL=C
EOF
$ singularity build --fakeroot torch.sif torch.def
INFO:    Starting build...
INFO:    Fetching OCI image...
24.8MiB / 24.8MiB [===============================================================================================================================] 100 % 2.8 MiB/s 0s
INFO:    Extracting OCI image...
INFO:    Inserting Singularity configuration...
....
INFO:    Adding environment to container
INFO:    Creating SIF file...
INFO:    Build complete: torch.sif
```

and finally run it interactively:

```bash
$ salloc --tasks=1 --cpus-per-task=2 --mem=1gb
$ srun singularity run torch.sif \
    python3 -c "import torch; print(torch.tensor(range(5)))"
tensor([0, 1, 2, 3, 4])
$ exit
```

We can also submit it as a batch:

```bash
$ cat > ~/submit_torch.sh << EOF
#!/usr/bin/sh -l

#SBATCH --job-name=torch
#SBATCH --mem=1gb
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2
#SBATCH --time=00:05:00

module load Singularity

srun singularity run torch.sif \
    python3 -c "import torch; print(torch.tensor(range(5)))"
EOF
$ sbatch submit_torch.sh
Submitted batch job 398
$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
               398   general    torch   nathan  R       0:03      1 bettley
$ cat slurm-398.out
tensor([0, 1, 2, 3, 4])
```

The next part will be setting up Conda, which is similarly a bit more aggravating than it probably should.

Once that's done, though:

```bash
$ conda env list

# conda environments:
#
base                   /mnt/nfs/slurm/miniforge
default-env            /mnt/nfs/slurm/miniforge/envs/default-env
python3.10             /mnt/nfs/slurm/miniforge/user_envs/python3.10
python3.11             /mnt/nfs/slurm/miniforge/user_envs/python3.11
python3.12             /mnt/nfs/slurm/miniforge/user_envs/python3.12
python3.13             /mnt/nfs/slurm/miniforge/user_envs/python3.13
```

And we can easily activate an environment...

```bash
$ source activate python3.13
(python3.13) $
```

And we can schedule jobs to run across multiple nodes:

```bash
$ cat > ./submit_conda.sh << EOF
#!/usr/bin/env bash

#SBATCH --job-name=conda
#SBATCH --mem=1gb
#SBATCH --ntasks=6
#SBATCH --cpus-per-task=2
#SBATCH --time=00:05:00

# Load Conda and activate Python 3.13 environment.
module load Conda
source activate python3.13

srun python --version

sleep 5
EOF
$ sbatch submit_conda.sh
Submitted batch job 403
$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
               403   general    conda   nathan  R       0:01      3 bettley,cargyll,dalt
$ cat slurm-403.out
Python 3.13.2
Python 3.13.2
Python 3.13.2
Python 3.13.2
Python 3.13.2
Python 3.13.2
```

Super cool.
