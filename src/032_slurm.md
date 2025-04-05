# Slurm

Okay, finally, geez.

So this is about [Slurm](https://slurm.schedmd.com), an open-source, highly scalable, and fault-tolerant cluster management and job-scheduling system.

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

The next step is to set up Lua and Lmod for managing environments. This itself isn't terribly fun or interesting. Lmod is on sourceforge, Lua is in Apt, we install some things, build Lmod from source, create some symlinks, and when we shell in and type a command, we can list our modules.

```bash
$ module av

------------------------------------------------------------------ /mnt/nfs/slurm/apps/modulefiles -------------------------------------------------------------------
   StdEnv

Use "module spider" to find all possible modules and extensions.
Use "module keyword key1 key2 ..." to search for all possible modules matching any of the "keys".
```