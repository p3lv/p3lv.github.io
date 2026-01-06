---
title: 'Containers, where did my perf stats go!?'
pubDate: TODO
description: 'Linux Containers (Apptainer), Performance counters (perf), what the hell is going on'
author: 'xypp3'
tags: []
---

_This blog will have a few "sidequest" sections, they are opional so skip for a
read._

Linux has a lovely tool called [perf](https://perfwiki.github.io/main/) (some fun
perf examples can be found [here](https://www.brendangregg.com/perf.html))
that can extract kernel stats of processes. Things like CPU cycles or number of
cache misses. But then I run the same program
inside a container, measuring from the host with perf, and get
[bar plot showing container run having orders of maginitued less instructions/cycles]()
Who ate all my instructions, cycles, cache referenecs and misses !?? Like some discrepency is expected, but not orders
of maginitued difference. Either perf is asleep at the wheel or I don't understand
how containers work. If I were a betting man, I'd bet on the latter.

## Sidequest: but really, what are containers?
I first encountered containers as a light weight alternative to Virtual Machines (VMs).
But how?? Like a VM encapuslates the code by running a whole guest kernel (sometimes hardware)
and guest OS ontop of your host OS. Like sure we we somehow glue the guest to the host
with software. Isolation is achieved! But how do you make that lighter?
What magic is being used? The secret is the Linux kernel provides isolation
primitives. So then, docker, podman, apptainer are just wrappers around those
kernel isolation features. For example it manages the isolation levels, creates
an isolated environment, manages resources etc. The two main kernel features
are **namespaces** for isolation and **cgropus** for resource management and
accounting. (tangent: For creating secure/hardened containers look containers
also use capabilities for permissions, AppArmour/LinuxSE for security system,
and setcomp bpf for banning dangerouse syscalls)

---

Ok so it seems that when perf tries to trace the container application's PID
it doesn't trace the subprocesses. So then if containers use cgroups for resource allocation and
monitoring, how do we check what it's monitoring? After traversing documentation
on cgroups, perf and alike, I found the `perf stat`flag `--cgroup` that can
filter perf events by cgroup name. AMAZING! but... ummm what's a cgroups name?
It turns out to simply be the filepath of the container direcory after `/sys/fs/cgroup`.
So for the cgroup found in `/sys/fs/cgroup/user.slice/user-1000.slice/user@1000.service/user.slice/ apptainer-30443`, the name is `user.slice/user-1000.slice/user@1000.service/user.slice/ apptainer-30443`

## Sidequest: cgroup names and management
This long name above, why? Well that is because Apptainer uses systemd to manage
cgroups. Docker for example has a docker directory in /sys/fs/cgroup because the
docker daemon does its cgroup management.

---

So then the final perf command will be:
```bash
perf stat -a \ # record system wide
  -e cycles,instructions,cache-misses,minor-faults,major-faults \ # an example selection of counters
  -G $CGROUP_NAME \  # filter by cgroup name
  -- sleep 60 # wait some time for program to complete
```

## Sidequest: to sudo or not to sudo
Running perf with admin privellages gets us over the hump and gets perf running.
But now, we don't understand why perf needs those privellages in the first
place. The privellages are needed because performance counters are A. a limited
resource for the kernel and B. can be considered sensitive data. So not every
user program should have access for those reasons. However, instead of sudo,
perf privellages can be granted by either changing the system setting:
`kernel.perf_event_paranoid` to be more permissive or by giving perf the cap
CAP_PERFMON, CAP_SYS_PTRACE or CAP_SYS_ADMIN. These same settings are also
referenced when perf runs without sudo and aborts on permissions error.

---

If you're interested you can see this [old talk](https://www.youtube.com/watch?v=NYLXZ58EboM)
