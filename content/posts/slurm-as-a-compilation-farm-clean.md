---

title: "SLURM as a Compilation Farm"
date: 2025-11-06
draft: false
------------

> **Note:** 📢 This document is based on my understanding of SLURM and is in no way a detailed guide covering every single topic. Take this as a practical guide from a noob’s perspective diving into it.

## **Introduction:**

This guide is designed to help you effectively use the SLURM scheduler on Rocky Linux 9.5 server. The Server allows you to run computational jobs using both interactive and non-interactive modes. The goal here is to make a compilation farm, although this guide specifically focuses on compiling the Linux kernel, one should note that this may also be used to compile any other tool given that the prerequisites and dependencies are known.

<!---**(STILL IN PROGRESS, WILL BE REMOVED IF NOT PRACTICAL)** Additionally, We’d also setup distributed compilation using **distcc** --->

## **Lab Infrastructure:**

The following are all on VMware ESXI

1. **Master:**

   * CPUs 4
   * Memory 4 GB
   * Hard disk 20 GB
   * Hostname: master

2. **Node 1:**

   * CPUs 4
   * Memory 4 GB
   * Hard disk 40 GB
   * Hostname: node1

3. **Node 2:**

   * CPUs 8
   * Memory 8 GB
   * Hard disk 40 GB
   * Hostname: node2

4. **Network File Storage**

   * Since compiling creates dozens of files, at least 30 GB is required for a successful compilation.
   * Used the existing testing server assigned to me.
   * NFS share path located in **/mnt/slrum_share**

Every instance has Rocky Linux 9.5 installed with SSH, root login and defined IP of all 4 nodes in the `/etc/hosts` file.

The Architecture diagram looks like this:

{{< figure src="/static/images/slurm-as-a-compilation-farm/SLUM_arch.drawio.png" alt="SLURM Architecture Diagram" >}}

## **Chapter 1: The installation:**

1. **Install and configure dependencies**

   Installation of slurm requires EPEL repo to be installed across all instances, install and enable it via:

   {{< highlight bash >}}
   dnf config-manager --set-enabled crb
   dnf install epel-release
   sudo dnf groupinstall "Development Tools"
   sudo dnf install munge munge-devel rpm-build rpmdevtools python3 gcc make openssl-devel pam-devel
   {{< /highlight >}}

   MUNGE is an authentication mechanism for secure communication between Slurm components. Configure it on all instances using:

   {{< highlight bash >}}
   sudo useradd munge
   sudo mkdir -p /etc/munge /var/log/munge /var/run/munge
   sudo chown munge:munge /usr/local/var/run/munge
   sudo chmod 0755 /usr/local/var/run/munge
   {{< /highlight >}}

   On Master:

   {{< highlight bash >}}
   sudo /usr/sbin/create-munge-key
   sudo chown munge:munge /etc/munge/munge.key
   sudo chmod 0400 /etc/munge/munge.key
   {{< /highlight >}}

   Copy the key to both nodes:

   {{< highlight bash >}}
   scp /etc/munge/munge.key root@node1:/etc/munge/
   scp /etc/munge/munge.key root@node2:/etc/munge/
   {{< /highlight >}}

   Start and enable the service:

   {{< highlight bash >}}
   sudo systemctl enable --now munge
   {{< /highlight >}}

2. **Installation of SLURM**

   Slurm is available in the EPEL repo. Install on all 3 instances:

   {{< highlight bash >}}
   sudo dnf install slurm slurm-slurmd slurm-slurmctld slurm-perlapi
   {{< /highlight >}}

   If by any chance packages are not available, download tar file from [SchedMD Downloads](https://www.schedmd.com/download-slurm/), extract, compile, and install using:

   {{< highlight bash >}}
   make -j$(nproc)
   sudo make install
   {{< /highlight >}}

## **Chapter 2: The Configuration:**

1. **Slurm configuration**

   On all 3 instances:

   {{< highlight bash >}}
   sudo useradd slurm
   sudo mkdir -p /etc/slurm /var/spool/slurmctld /var/spool/slurmd /var/log/slurm
   sudo chown slurm:slurm /var/spool/slurmctld /var/spool/slurmd /var/log/slurm
   {{< /highlight >}}

   Edit the configuration on master:

   {{< highlight bash >}}
   sudo nano /etc/slurm/slurm.conf
   {{< /highlight >}}

   Ensure the following key lines are present and correctly configured:

   {{< highlight text >}}
   ClusterName=debug
   SlurmUser=slurm
   ControlMachine=slurm-master
   SlurmctldPort=6817
   SlurmdPort=6818
   AuthType=auth/munge
   StateSaveLocation=/var/spool/slurmctld
   SlurmdSpoolDir=/var/spool/slurmd
   SwitchType=switch/none
   MpiDefault=none
   SlurmctldPidFile=/var/run/slurmctld.pid
   SlurmdPidFile=/var/run/slurmd.pid
   ProctrackType=proctrack/pgid
   ReturnToService=1
   SchedulerType=sched/backfill
   SlurmctldTimeout=300
   SlurmdTimeout=30
   NodeName=node1 CPUs=4 RealMemory=3657 State=UNKNOWN
   NodeName=node2 CPUs=8 RealMemory=7682 State=UNKNOWN
   PartitionName=debug Nodes=node[1-2] Default=YES MaxTime=INFINITE State=UP
   {{< /highlight >}}

   Copy configuration to nodes:

   {{< highlight bash >}}
   scp /etc/slurm/slurm.conf root@node1:/etc/slurm/slurm.conf
   scp /etc/slurm/slurm.conf root@node2:/etc/slurm/slurm.conf
   {{< /highlight >}}

   Start and enable services:

   {{< highlight bash >}}
   sudo systemctl enable --now slurmctld
   sudo systemctl enable --now slurmd
   {{< /highlight >}}

2. **Firewall Configuration:**

   Open required ports:

   {{< highlight bash >}}
   sudo firewall-cmd --permanent --add-port=6817/tcp
   sudo firewall-cmd --permanent --add-port=6818/tcp
   sudo firewall-cmd --permanent --add-port=6819/tcp
   sudo firewall-cmd --reload
   {{< /highlight >}}

## **Chapter 3: Testing and Introduction to the commands:**

**(While this is a short guide on the commands and its flags, you could always use man pages to understand it more deeply)**

1. `sinfo`:

   Displays node and partition information:

   {{< highlight bash >}}
   sinfo
   {{< /highlight >}}

2. `srun`:

   Runs commands interactively on compute nodes:

   {{< highlight bash >}}
   srun -N2 -n2 nproc
   {{< /highlight >}}

3. `sbatch`:

   Submits a job script:

   {{< highlight bash >}}
   sbatch testjob.sh
   {{< /highlight >}}

4. `squeue`:

   Displays details of currently running jobs:

   {{< highlight bash >}}
   squeue
   {{< /highlight >}}

5. `scancel`:

   Cancels a submitted job:

   {{< highlight bash >}}
   scancel 1
   {{< /highlight >}}

6. `scontrol`:

   Displays detailed job and node information:

   {{< highlight bash >}}
   scontrol show job 1
   {{< /highlight >}}

   {{< highlight bash >}}
   scontrol show partition
   {{< /highlight >}}

## **Chapter 4: Setting up the NFS storage.**

It is a good idea to have shared storage for SLURM. Install `nfs-utils`:

{{< highlight bash >}}
sudo dnf install nfs-utils
{{< /highlight >}}

**On the NFS server:**

{{< highlight bash >}}
mkdir /srv/slurm_share
nano /etc/exports
{{< /highlight >}}

Add the following line:

```
/srv/slurm_share 10.10.40.0/24(rw,sync,no_subtree_check,no_root_squash)
```

Open necessary ports:

{{< highlight bash >}}
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --permanent --add-port={5555/tcp,5555/udp,6666/tcp,6666/udp}
firewall-cmd --reload
{{< /highlight >}}

Export and enable the service:

{{< highlight bash >}}
exportfs -v
systemctl enable --now nfs-server
{{< /highlight >}}

**On the master and compute nodes:**

{{< highlight bash >}}
sudo mkdir /mnt/slurm_share
{{< /highlight >}}

Add the mount in `/etc/fstab`:

```
10.10.40.0:/srv/slurm_share /mnt/slurm_share nfs defaults 0 0
```

Reboot machines and verify the share mounts properly.

## **Chapter 5: Setting up the Compile/Build Environment**

Install kernel build dependencies:

{{< highlight bash >}}
srun -n2 -N2 sudo dnf groupinstall "Development Tools" -y && sudo dnf install ncurses-devel bison flex elfutils-libelf-devel openssl-devel wget bc dwarves -y
{{< /highlight >}}

Download the Linux kernel source from [kernel.org](https://kernel.org):

{{< highlight bash >}}
wget [https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.14.8.tar.xz](https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.14.8.tar.xz)
tar xvf linux-6.14.8.tar.xz
{{< /highlight >}}

Define architecture-specific config:

{{< highlight bash >}}
make defconfig
{{< /highlight >}}

Create `compile_kernel.sh` on the shared directory:

{{< highlight bash >}}
#!/bin/bash
#SBATCH --job-name=kernel_build
#SBATCH --output=kernel_build_%j.out
#SBATCH --error=kernel_build_%j.err
#SBATCH --time=03:00:00
#SBATCH --nodes=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=8G

KERNEL_SOURCE_PATH="/mnt/slurm_share/linux-6.8.9"
BUILD_OUTPUT_DIR="/mnt/slurm_share/kernel_builds/${SLURM_JOB_ID}"

mkdir -p "$BUILD_OUTPUT_DIR"
cd "$KERNEL_SOURCE_PATH"

NUM_MAKE_JOBS=${SLURM_CPUS_PER_TASK}
make -j"${NUM_MAKE_JOBS}" ARCH=x86_64 Image modules dtbs

if [ $? -eq 0 ]; then
cp "$KERNEL_SOURCE_PATH/arch/x86/boot/bzImage" "$BUILD_OUTPUT_DIR/"
else
echo "Kernel compilation failed."
fi
{{< /highlight >}}
