---

title: "Setting up the Community Edition of IBM LSF (Load Sharing Facility)"
date: 2026-01-14
draft: false
------------


## Introduction

I wanted to understand how LSF actually works, not just at a surface “submit job, job runs” level, but the *real* mechanics. Most places that use LSF don’t document it well, and when they do, it’s written assuming you already run a production cluster.

So I decided to build a small LSF Community Edition cluster at home and treat it like an EDA-style environment: master node, execution nodes, login-style submission, interactive jobs, X11, the whole thing.

This post is not a clean guide. It’s more like notes from what actually happened.

---

## Chapter 1: Why LSF and Not Slurm

I already have experience with Slurm, and I even wrote about using Slurm as a compilation farm. Slurm is straightforward. LSF is not.

But that’s exactly why I wanted to learn it.

LSF still shows up in EDA, semiconductor companies, and older HPC environments. And the way it thinks about hosts, submission, and execution is *very different* from Slurm.

In Slurm, a login node is mostly just a machine that can talk to `slurmctld`.

In LSF, **every host must be known, typed, and classified**. Even submit-only machines.

That difference alone is worth learning.

---

## Chapter 2: The Cluster Layout I Used

I kept the setup intentionally small.

- `lsf-master`  
  Runs LIM, mbatchd, acts as master candidate.

- `lsf-node1`  
  Execution node.

- `fedora`  
  My Fedora workstation, acting as a login / submission host.

All machines could resolve each other via `/etc/hosts`. No DNS magic.

I used Rocky Linux for `lsf-master` and `lsf-node1`, and Fedora for my submission machine.

---

## Chapter 3: Getting LSF

This part deserves its own chapter because this is where things *already* got weird.

There is no obvious “download LSF” button.

I actually found the [IBM Spectrum LSF Community Edition](https://epwt-www.mybluemix.net/software/support/trial/cst/programwebsite.wss?siteId=680&h=null&p=null) through a Reddit post. That post linked to IBM’s site, which then required me to:

1. Create an IBM account
2. Log in
3. Accept licensing terms
4. Navigate through IBM’s download portal

Only *after* all that was I able to download the tarball.

What I ended up with was something like:

```

lsfsce10.2.0.15-x86_64.tar.Z
```

This already tells you something important:
- IBM ships **prebuilt binaries**
- They are tied to a *specific* glibc and kernel baseline

That becomes important later when Fedora starts complaining.

---

## Chapter 3.1: Extracting the Tarball

On the master node (`lsf-master`), I extracted it under `/tmp` first:

```

tar -xvf lsfsce10.2.0.15-x86_64.tar.Z
cd  lsfsce10.2.0.15-x86_64

```

Inside, you don’t get a fancy installer. You get scripts.

The real entry point is:

```

./lsfinstall 
```

But **do not run it blindly**.

---

## Chapter 3.2: `install.config` Is the Real Installer

LSF installation is driven almost entirely by a file called:

```

install.config

```

If you mess this up, the installer *will still run*, but you’ll regret it later.

This is roughly what I used (simplified to the important parts):

```

LSF_TOP="/usr/local/lsf"
LSF_ADMINS="edauser"
LSF_CLUSTER_NAME="lsfcluster"

LSF_MASTER_LIST="lsf-master"
LSF_SERVER_HOSTS="lsf-master lsf-node1"

ENABLE_EGO="Y"
EGO_DAEMON_CONTROL="Y"

LSF_LOCAL_RESOURCES="[resource mg]"

```

Things that matter here:

- `LSF_TOP`  
  This decides **everything**. Binaries, configs, logs. I kept it simple.

- `LSF_ADMINS`  
  This user *must exist*. I created `edauser` beforehand.

- `LSF_CLUSTER_NAME`  
  This name shows up everywhere later. Pick once, don’t change it.

- `LSF_MASTER_LIST`  
  This is *not optional*. If this is wrong, LIM will never behave.

- `LSF_SERVER_HOSTS`  
  These are execution-capable hosts. Submission-only hosts do NOT go here.

At this stage, **Fedora was not included anywhere**.

---

## Chapter 3.3: Running the Installer

Only after editing `install.config` did I run:

```

./lsfinstall -f install.config

```

The installer:
- creates `/usr/local/lsf`
- drops binaries under versioned directories
- writes configs under `/usr/local/lsf/conf`

When it finished, I had:

```

/usr/local/lsf/10.1/
/usr/local/lsf/conf/
/usr/local/lsf/work/

```

At this point, LSF was technically “installed”.

That does not mean usable.

---

## Chapter 3.4: Environment Setup

LSF does *nothing* unless you source its environment and start the daemons.

On the master and nodes:

```

source /usr/local/lsf/conf/profile.lsf

```

Without this:
- `bhosts` won’t work
- `lsid` won’t work
- errors will be extremely misleading

I added this to the admin user’s shell profile later, but initially I sourced it manually.

To start the daemons I used the commad:
```
lsf_daemons start
```
---

## Chapter 4: The First Big Reality Check — Hosts Must Be Known

One mistake I made early was assuming that copying the tarball to another machine and sourcing `profile.lsf` was enough.

It is not.

LSF does not work like that.

When I ran:

```

bhosts

```

on my Fedora machine, I kept getting:

```

Failed in an LSF library call: LIM is down; try later

```

Even though:
- I could ping the master
- LIM ports were reachable
- SSH worked

The problem wasn’t networking.

The problem was that **LSF didn’t know my Fedora machine existed**.

---

## Chapter 5: Teaching LSF That Fedora Exists (and What It Is)

This was the first major “oh” moment.

I kept assuming that since Fedora was only a submit host, LSF wouldn’t really care about it. That assumption was wrong.

If a machine is going to submit jobs, it **must** appear in the cluster definition file:

```

/usr/local/lsf/conf/lsf.cluster.<clustername>

```

Even if:
- it never runs jobs
- it never runs RES
- it is just a login box

LSF is very literal about this.

My `Host` section eventually looked like this:

```

Begin Host
HOSTNAME    model   type    server  RESOURCES
lsf-master  !       !       1       (mg)
lsf-node1   !       !       1       ()
fedora      !       !       0       ()
End Host

```

That `server 0` is important. Fedora is *not* an execution host. It only submits jobs.

Once I added Fedora here and ran:

```

lsadmin reconfig

```

the errors changed. They didn’t disappear, but they changed, which is usually how you know you’re moving forward with LSF.

---

At this point, I thought I was done.

I wasn’t.

---

Even after Fedora was listed in `lsf.cluster`, I kept getting this error:

```

Cannot find restarted or newly submitted job's submission host and host type

```

This one took time to understand.

LSF doesn’t just want to know **that** a host exists.  
It also wants to know **what kind of host it is**:
- architecture
- operating system

That information does **not** live in `lsf.cluster`.

It lives in:

```

/usr/local/lsf/conf/lsf.shared

```

This is the file almost everyone forgets.

I had to explicitly define:
- the host model (`X86_64`)
- the host type (`LINUX`)
- and map *every* host, including Fedora

This is what finally fixed it:

```

Begin Host
HOSTNAME    model    type
lsf-master  X86_64   LINUX
lsf-node1   X86_64   LINUX
fedora      X86_64   LINUX
End Host

```

After saving this file, I ran:

```

lsadmin reconfig

```

Only after this did `bhosts` stop rejecting Fedora and job submission start behaving like it should.

This was a big lesson for me:  
LSF will happily run with half the information missing, but it will fail in ways that make it feel like something much deeper is broken.

---

## Chapter 6: Interactive Jobs Work… Until X11 Shows Up

Once submission worked, I tried:

```

bsub -Is -XF xterm

```

And hit a whole new class of errors:

```

X11 connection rejected because of wrong authentication

```

At this point, LSF was fine.  
The cluster was fine.

This was **pure SSH + X11 pain**.

What Was Actually Missing

The main issues were:

- `xauth` not installed on all nodes
- X11 forwarding not consistently enabled
- SSH key-based auth not fully set up

Installing `xauth` everywhere and making sure:

```

X11Forwarding yes
X11UseLocalhost no

```

was set on **all** machines finally fixed it.

Only after this did:

```

bsub -Is -XF -m lsf-master xterm

```

actually open a window.

---

## Closing Thoughts

At this point, the cluster was stable enough for what I wanted to test.

I initially planned to go one step further and add a proper license server using FlexLM, similar to how it’s done in real EDA environments. The idea was to tie license availability into scheduling and see how LSF behaves under those constraints.

But I decided to stop here.

The goal of this setup was to understand:
- how LSF components talk to each other
- how submit hosts differ from execution hosts
- and what minimum configuration is actually required for things to work

That part was done.

License servers can come later as a separate iteration.
