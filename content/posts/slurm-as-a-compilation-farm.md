---
title: "SLURM as a Compilation Farm"
date: 2025-11-06
draft: true
---

> 📢 NOTE: This document is based on my understanding of SLURM and is in no way a detailed guide covering every single topic, take this as a practical guide from a noob’s perspective diving into it.

## **Introduction:**

This guide is designed to help you effectively use the SLURM scheduler on Rocky Linux 9.5 server. The Server allows you to run computational jobs using both interactive and non-interactive modes. The goal here is to make a compilation farm, although this guide specifically focuses on compiling the Linux kernel, one should note that this may also me used to compile any other tool given that the perquisites and dependencies are known.

<!---**[STILL IN PROGRESS, WILL BE REMOVED IF NOT PRACTICAL]** Additionally, We’d also setup distributed compilation using **distcc** --->

## **Lab Infrastructure:**

The following are all on VMware ESXI

1.  **Master:**
    - CPUs 4
    - Memory 4 GB
    - Hard disk 20 GB
    - Hostname: master

1.  **Node 1:**
    - CPUs  4
    - Memory 4 GB
    - Hard disk 40 GB
    - Hostname: node1

1. **Node 2:**
    - CPUs 8
    - Memory 8 GB
    - Hard disk 40 GB
    - Hostname: node2

1. **Network File Storage**
    - Since compiling creates dozens of file, at least 30 Gigs is required for a successful compilation.
    - Used the existing testing server assigned to me.
    - NFS share path located in **/mnt/slrum_share**

Every instance has Rocky Linux 9.5 installed with ssh, root login and defined ip of all 4 nodes in the /etc/hosts file

The Architecture diagram for the looks like this:

<img src="images/slurm-as-a-compilation-farm/SLUM_arch.drawio.png" alt="SLUM_arch.drawio.png">

## **Chapter 1: The installation:**

1.  **Install and configure dependencies**

Installation of slurm requires EPEL repo to be installed across all instances, install and enable it via

```bash
dnf config-manager --set-enabled crb
dnf install epel-release
sudo dnf groupinstall "Development Tools"
sudo dnf install munge munge-devel rpm-build rpmdevtools python3 gcc make openssl-devel pam-devel
```

MUNGE is an authentication mechanism for secure communication between Slurm components. configure it on all instances using:

```bash
sudo useradd munge
sudo mkdir -p /etc/munge /var/log/munge /var/run/munge
sudo chown munge:munge /usr/local/var/run/munge
sudo chmod 0755 /usr/local/var/run/munge # Owner can rwx, group/others can rx (common for run dirs)
```

On Master:

```bash
sudo /usr/sbin/create-munge-key
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 0400 /etc/munge/munge.key
```

This creates a munge key with the necessary permissions, we need to copy the key to both our nodes by using scp.

```bash
scp /etc/munge/munge.key root@node1:/etc/munge/
scp /etc/munge/munge.key root@node2:/etc/munge/
```

Start and Enable the service using:

```bash
sudo systemctl enable --now munge

```

1.  **Installation of SLURM**

Slurm is avaliable in the EPEL repo install on all 3 instances using:

```bash
sudo dnf install slurm slurm-slurmd slurm-slurmctld slurm-perlapi
```

If by any chance packages are not avaliable (Highly unlikely) you can always download tar file from [Download Slurm - SchedMD](https://www.schedmd.com/download-slurm/) extract it and use the below command to compile and install it.

```bash
make -j$(nproc)
sudo make install
```

## **Chapter 2: The Configuration:**

1. **Slurm configuration**

On all 3 instances use the following command

```bash
sudo useradd slurm
sudo mkdir -p /etc/slurm /var/spool/slurmctld /var/spool/slurmd /var/log/slurm
sudo chown slurm:slurm /var/spool/slurmctld /var/spool/slurmd /var/log/slurm
```

This creates the user, log files and changes the ownership of the files to be slurm

Now open the slurm config file on the master using:

```bash

sudo nano /etc/slurm/slurm.conf
```

The main lines to be focused on are:

`ClusterName=debug`

`SlurmUser=slurm`

`ControlMachine=slurm-master` 

`SlurmctldPort=6817`

`SlurmdPort=6818`

`AuthType=auth/munge`

`StateSaveLocation=/var/spool/slurmctld`

`SlurmdSpoolDir=/var/spool/slurmd`

`SwitchType=switch/none`

`MpiDefault=none`

`SlurmctldPidFile=/var/run/slurmctld.pid`

`SlurmdPidFile=/var/run/slurmd.pid`

`ProctrackType=proctrack/pgid`

`ReturnToService=1SchedulerType=sched/backfill`

`SlurmctldTimeout=300`

`SlurmdTimeout=30`

`NodeName=node1 CPUs=4 RealMemory=3657 State=UNKNOWN`

`NodeName=node2 CPUs=8 RealMemory=7682 State=UNKNOWN`

`PartitionName=debug Nodes=node[1-2] Default=YES MaxTime=INFINITE State=UP`

Make sure to the values are the same as above without any spelling mistakes, also if any value is not available create them.

Copy the same config onto **node1** and **node2 using**

```bash
scp /etc/slurm/slurm.conf root@node1:/etc/slurm/slurm.conf
scp /etc/slurm/slurm.conf root@node2:/etc/slurm/slurm.conf
```

Start and enable the services on boot

On master:

```bash
sudo systemctl enable --now slurmctld
```

On compute nodes:

```bash
sudo systemctl enable --now slurmd
```

Slurm should be working and synced up to the nodes

1. **Firewall Configuration:**

Firewall prevents ports required by slurm to be accessed, while in my testing case I had disable the firewall service, I would not recommend doing this in production, hence the correct way to open ports in slurm, on all 3 instances:

```bash
sudo firewall-cmd --permanent --add-port=6817/tcp
sudo firewall-cmd --permanent --add-port=6818/tcp
sudo firewall-cmd --permanent --add-port=6819/tcp
sudo firewall-cmd --reload
```

## **Chapter 3: Testing and Introduction to the commands:**

**[While this is a short guide on the commands and its flags, you could always use man pages to understand it more deeply]**

1. `sinfo`:

This command is displays the information about the nodes, what the partition name is, on what state they’re in and how many nodes are included in the partition

```bash
[root@master ~]# sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
debug*       up   infinite      2   idle node[1-2]
```

Idle here means that the nodes are waiting for a job to be assigned. 

1. `srun`:

srun is used to run command on the compute nodes via master in an interactive manner. For example if I needed to know the cores available in node 1 and 2

```bash
[root@master ~]# srun -N2 -n2 nproc
 4
 8
```

Where the `-N`  is the number of nodes for the job (here nproc) to be allocated, and `-n` is the number of tasks to be given to the node.

1. `sbatch` :

Used to submit a job script which is provided as an argument to the file. For example

```bash
[root@master ~]# sbatch [testjob.sh](http://testjob.sh/)
Submitted batch job 1
```

We’ll be learning more about the syntax of these files later

1. `squeue`:

Used to display details of currently running jobs, such as on what node is it running, user and how long has it has been running.

```bash
[root@master ~]# squeue
JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
	1      debug   test_job    root  R       0:02      1 node1
```

Here we can see the job we previously executed job via `sbatch`

1. `scancel`:

Used to cancel a submitted job, this command does not explicitly return an output.

```bash
[root@master ~]# scancel 1
[root@master ~]# squeue
JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
[root@master ~]#
```

1. `scontrol` :

Provides a detailed information about the nodes the usage of this command is a little vast some common example are:

```bash
[root@master ~]# scontrol show job 1
JobId=1 JobName=test_job
UserId=root(0) GroupId=root(0) MCS_label=N/A
Priority=4294901750 Nice=0 Account=(null) QOS=normal
JobState=RUNNING Reason=None Dependency=(null)
Requeue=1 Restarts=0 BatchFlag=1 Reboot=0 ExitCode=0:0
RunTime=00:00:04 TimeLimit=00:01:00 TimeMin=N/A
SubmitTime=2025-05-19T02:34:05 EligibleTime=2025-05-19T02:34:05
AccrueTime=2025-05-19T02:34:05
StartTime=2025-05-19T02:34:05 EndTime=2025-05-19T02:35:05 Deadline=N/A
SuspendTime=None SecsPreSuspend=0 LastSchedEval=2025-05-19T02:34:05 Scheduler=Backfill
Partition=debug AllocNode:Sid=master:20850
ReqNodeList=(null) ExcNodeList=(null)
NodeList=node1
BatchHost=node1
NumNodes=1 NumCPUs=1 NumTasks=1 CPUs/Task=1 ReqB:S:C:T=0:0:*:*
TRES=cpu=1,node=1,billing=1
Socks/Node=* NtasksPerN:B:S:C=0:0:*:* CoreSpec=*
MinCPUsNode=1 MinMemoryNode=0 MinTmpDiskNode=0
Features=(null) DelayBoot=00:00:00
OverSubscribe=OK Contiguous=0 Licenses=(null) Network=(null)
Command=/root/testjob.sh
WorkDir=/root
StdErr=/root/slurm_test_output.txt
StdIn=/dev/null
StdOut=/root/slurm_test_output.txt
Power=
```

6.1 `scontrol show job 1`: Displays details of the running job

```bash
[root@master ~]# scontrol show partition
PartitionName=debug
AllowGroups=ALL AllowAccounts=ALL AllowQos=ALL
AllocNodes=ALL Default=YES QoS=N/A
DefaultTime=NONE DisableRootJobs=NO ExclusiveUser=NO GraceTime=0 Hidden=NO
MaxNodes=UNLIMITED MaxTime=UNLIMITED MinNodes=0 LLN=NO MaxCPUsPerNode=UNLIMITED
Nodes=node[1-2]
PriorityJobFactor=1 PriorityTier=1 RootOnly=NO ReqResv=NO OverSubscribe=NO
OverTimeLimit=NONE PreemptMode=OFF
State=UP TotalCPUs=12 TotalNodes=2 SelectTypeParameters=NONE
JobDefaults=(null)
DefMemPerNode=UNLIMITED MaxMemPerNode=UNLIMITED
TRES=cpu=12,mem=11339M,node=2,billing=12
```

6.2 `scontrol show partition`: Displays information about the partitions (here referring to the node config).

6.3 `scontrol cancel jobid=<jobid>`: Another way to cancel jobs

6.4 `scontrol requeue <jobid>`: Requeues the an errored out job

6.5 `scontrol update jobid=<job_id> priority=100000`: Increases the priority of a running job

6.6 `scontrol update NodeName=node01 State=RESUME`: Resume a drained or a disabled node

## **Chapter 4: Setting up the NFS storage.**

It is a good idea to have a shared storage on slurm, since you may run into issues like not having enough space, or when you want both the nodes accessing a same file or folder. We’ll be using nfs-utils package to achieve this, install on NFS device and 

```bash
sudo dnf install nfs-utils
```

**On the Linux machine to be used as NFS (here testing-server)**

Creating a directory in /srv/slrum_share using:

```bash
mkdir /srv/slurm_share
```

Open the file **/etc/exports** in nano and add the following line

```bash
nano /etc/exports
```

`/srv/slurm_share 10.10.40.0/24(rw,sync,no_subtree_check,no_root_squash)`

Save and quit the file.

Open Ports using:

```bash
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --permanent --add-port={5555/tcp,5555/udp,6666/tcp,6666/udp}
firewall-cmd --reload
```

Export the share and enable the server using the following command:

```bash
exportfs -v   
systemctl enable —now nfs-server 
```

**On the master and compute nodes:**

Create a directory in /mnt/slurm_share using:

```bash
sudo mkdir /mnt/slurm_share
```

To the mount the share on boot open the following file in nano with /etc/nano and paste the line:

`/mnt/slurm_share 10.10.40.0/24(rw,sync,no_subtree_check,no_root_squash)`

Reboot the machines and you should have the shared folder mounted to **/mnt/slurm_share**

## **Chapter 5: Setting up the Compile/Build Environment**

There are certain development dependencies that are required to build the kernel, these need to be installed on all the nodes including the master, use this command to parallelly install them:

```bash
srun -n2 -N2 sudo dnf groupinstall "Development Tools" -y &&  sudo dnf install ncurses-devel bison flex elfutils-libelf-devel openssl-devel wget bc dwarves -y
```

This will install them in on the nodes and the same command without **srun -n2 -N2** to install on master.

Download the Linux kernel source from [kernel.org](http://kernel.org) as for writing this the latest stable kernel version is 6.14.8. Download this tar and extract on the shared folder using.

```bash
wget [https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.14.8.tar.xz](https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.14.8.tar.xz)
tar xvf linux-6.14.8.tar.xz
```

Use the below command to define an architecture specific **.config** file 

```bash
make defconfig
```

**On master:**

create a file named compile_kernel.sh on the share mounted directory and paste the code in it:

```bash
#!/bin/bash
#
# SLURM Batch Script for Kernel Compilation
#
#SBATCH --job-name=kernel_build       # Name of your job
#SBATCH --output=kernel_build_%j.out  # Standard output file (%j expands to jobID)
#SBATCH --error=kernel_build_%j.err   # Standard error file
#SBATCH --time=03:00:00               # Max runtime (3 hours, adjust as needed)
#SBATCH --nodes=1                     # Request 1 compute node
#
# --- RESOURCE REQUESTS (CHOOSE ONE SET) ---
#
# Option A: For node1 (4 cores, 4GB RAM)
# #SBATCH --cpus-per-task=4           # Request 4 CPU cores for make -j4
# #SBATCH --mem=4G                    # Request 4GB RAM
#
# Option B: For node2 (8 cores, 8GB RAM) - RECOMMENDED FOR FASTER BUILDS
#SBATCH --cpus-per-task=8             # Request 8 CPU cores for make -j8
#SBATCH --mem=8G                      # Request 8GB RAM
#
# --- END RESOURCE REQUESTS ---

# Set environment variables for the kernel build
# Replace 'linux-6.x.y' with your actual kernel source directory name
KERNEL_SOURCE_PATH="/mnt/slurm_share/linux-6.8.9"
BUILD_OUTPUT_DIR="/mnt/slurm_share/kernel_builds/${SLURM_JOB_ID}"

echo "-----------------------------------------------------"
echo "SLURM Job ID: ${SLURM_JOB_ID}"
echo "Job Name: ${SLURM_JOB_NAME}"
echo "Running on host: $(hostname)"
echo "Assigned CPUS: ${SLURM_CPUS_PER_TASK}"
echo "Assigned Memory: ${SLURM_MEM_PER_NODE}MB" # Note: SLURM_MEM_PER_NODE is in MB if --mem was in GB
echo "Kernel Source Path: ${KERNEL_SOURCE_PATH}"
echo "Build Output Dir: ${BUILD_OUTPUT_DIR}"
echo "Current working directory: $(pwd)"
echo "-----------------------------------------------------"

# Create a unique directory for the build output (artifacts)
mkdir -p "$BUILD_OUTPUT_DIR"
if [ $? -ne 0 ]; then
    echo "ERROR: Could not create output directory $BUILD_OUTPUT_DIR"
    exit 1
fi

# Change to the kernel source directory
cd "$KERNEL_SOURCE_PATH"
if [ $? -ne 0 ]; then
    echo "ERROR: Could not change to kernel source directory $KERNEL_SOURCE_PATH"
    exit 1
fi

# Load any necessary modules (e.g., specific GCC versions, cross-compilers)
# module load gcc/12.2.0
# module load some-toolchain

# Set the number of parallel jobs for 'make'.
# This uses the number of CPUs SLURM allocated to your job.
NUM_MAKE_JOBS=${SLURM_CPUS_PER_TASK}

echo "Starting kernel compilation using 'make -j${NUM_MAKE_JOBS}'..."
echo "Compiling on node: $(hostname) with $(nproc) physical cores and $(lscpu | grep 'Thread(s) per core' | awk '{print $NF}') threads per core."

# Execute the kernel compilation
# Assuming you want to build the 'Image' (compressed kernel), 'modules', and 'device tree blobs'
# Adjust ARCH and CROSS_COMPILE if you are cross-compiling for a different architecture
make -j"${NUM_MAKE_JOBS}" \
    ARCH=x86_64 \
    # CROSS_COMPILE=arm-linux-gnueabihf- \ # Uncomment and adjust for cross-compilation
    Image modules dtbs

# Check the exit status of the make command
if [ $? -eq 0 ]; then
    echo "Kernel compilation successful!"
    echo "Copying build artifacts to $BUILD_OUTPUT_DIR"

    # Copy relevant build artifacts to your output directory
    cp "$KERNEL_SOURCE_PATH/arch/x86/boot/bzImage" "$BUILD_OUTPUT_DIR/"
    # For modules, you might want to install them into a temporary location
    # make modules_install INSTALL_MOD_PATH="${BUILD_OUTPUT_DIR}/modules_install"
    # cp -r "$KERNEL_SOURCE_PATH/System.map" "$BUILD_OUTPUT_DIR/"
    # cp -r "$KERNEL_SOURCE_PATH/.config" "$BUILD_OUTPUT_DIR/"

else
    echo "Kernel compilation FAILED. Check the output in ${SLURM_JOB_NAME}_${SLURM_JOB_ID}.err"
    echo "Also check the detailed logs in $BUILD_OUTPUT_DIR/."
fi

echo "Job finished at $(date)"
echo "-----------------------------------------------------"
```
