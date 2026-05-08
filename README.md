# Introduction
## Linux AMD Gaming OptimizationGuide
This Repo has descriptions of how to optimize the gaming experience on Linux with AMD cpu and amd gpu

# Bios Settings

## Settings Bios Settings
Lets start off with bios settings. These play a really big role in performance.

First of all you want to enable your XMP profile for your RAM. 

Then do these settings:

Resizeable Bar - Auto/Enable
Above 4G Decoding - Enable
Amd fTPM - Disable // There has been some bugs where this caused some stutters, but i don't know if this is fixed already. Better to turn it off.
CSM Supoort - DISABLE !!! This is needed in disable for resizable bar support
Core Performance Boost - auto/enable
CPPC - ENABLE
CPPC preferred cores - ENABLE
Global C state Control - ENABLE
AMD cool & quiet - ENABLE // this and some others are very important to being able to use the amd-pstate or amd-pstate-epp cpufreq driver
and right around amd cool & quite also needs to be something like this:
PState - PState0

Also dont forget to set the Cooler graphs of your fans in the case and cpu cooler in the cooler section in the bios.

## Validating some of them 

### Resizable Bar

This command should say more then 128MB
`sudo dmesg | grep -i 'BAR.*VRAM\|amdgpu.*BAR'`

output example:
`[    5.237537] amdgpu 0000:0d:00.0: [drm] Detected VRAM RAM=24560M, BAR=32768M`

Here it is enabled.

### Ram speed

you can easily see this in mission-center, or other task managers, but lets do this also in commandline.

`inxi -mxx`

and you will see something like this:

```
Memory:
  System RAM: total: 32 GiB available: 31.25 GiB used: 11.76 GiB (37.6%)
  Message: For most reliable report, use superuser + dmidecode.
  Array-1: capacity: 128 GiB slots: 4 modules: 4 EC: None
    max-module-size: 32 GiB note: est.
  Device-1: Channel-A DIMM 0 type: DDR4 size: 8 GiB speed: 3600 MT/s
    volts: 1 note: check manufacturer: G.Skill part-no: F4-3600C16-8GTZNC
  Device-2: Channel-A DIMM 1 type: DDR4 size: 8 GiB speed: 3600 MT/s
    volts: 1 note: check manufacturer: G.Skill part-no: F4-3600C16-8GTZNC
  Device-3: Channel-B DIMM 0 type: DDR4 size: 8 GiB speed: 3600 MT/s
    volts: 1 note: check manufacturer: G.Skill part-no: F4-3600C16-8GTZNC
  Device-4: Channel-B DIMM 1 type: DDR4 size: 8 GiB speed: 3600 MT/s
    volts: 1 note: check manufacturer: G.Skill part-no: F4-3600C16-8GTZNC

```
here you can then check if the desired ram speed is applied.

### Cpu Freq Driver

with this command you can see your used cpu freq driver. For amd you want to see amd-pstate or amd-pstate-epp

`cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver`

output example:

`amd-pstate`

This will be very imporant for the kernel commandline parameter amd_pstate, which can be passive,active or guided.

Here the best performant is active. Which lets the hardware handle boosting of the cpu frequency BUT this does not always work, some hardware just cannot boost itself.
That is why its important to have some tools like mission-center with which you can check if the cpu frequency is boosted above the base clock. If amd_pstate=active does not work, then use amd_pstate=passive.

# Kernel Commandline parameter

First of all i want to show my command line parameters:

```
amd-pstate=passive amdgpu.aspm=0 amdgpu.audio=0 nmi_watchdog=0 nowatchdog processor.max_cstate=1 transparent_hugepage=never vm.zone_reclaim_mode=0 audit=0 pcie_aspm=off ignore_rlimit_data split_lock_detect=off split_lock_mitigate=0 preempt=full libahci.ignore_sss=1 loglevel=3 rd.systemd.show_status=false transparent_hugepage_tmpfs=never amdgpu.dcdebugmask=0x4
```

## Descriptions of each parameter I used and why

`amd-pstate` as already described above tells the kernel if the kernel should use control the cpu frequency or the hardware itself. I have amd-pstate=passive because my hardware is not able to boost itself and is otherwise locked to the base frequency.

`amdgpu.aspm=0` is used to disable the PCI Express power-saving for amdgpus otherwise the PCI Express link has a wake up latency when it goes into idle state and it wants to wakeup

`amdgpu.audio=0` to disable the audio over HDMI/DP, i dont need it. I have an external DAC.

`nmi_watchdog=0` is for Hard lockup detector and Soft lockup detection of Non-Maskable Interrupts, which uses a watchdog to detect that which needs to run in a set interval. When you disable it you free up some cpu time.

`nowatchdog` same as above but for all watchdogs

`processor.max_cstate=1` sets the max C state or better said idle state to C1. It normally goes until C6 or even lower. The higher the number the deeper the sleep state. The deeper the sleep state the higher the wakup latency, to minimize the latency we do max cstate to 1 and i think you need the Global C state Control option enabled in the bios to really benefit from it.

`transparent_hugepage=never` disable transparent hugepages, normal pagesize is 4KB when THP is enabled every page is handles as a Hugepage and has around 2MB which might enhance the performance in some cases but it also leads to stutters in other cases and we want to optimize for smoothness so we disable it to reduce stutter/lag. When enabled the kernel starts a process in the background called khugepaged which tries to compress the THP pages in RAM, which causes stutters, lags and hangs and we dont want that.

`vm.zone_reclaim_mode=0` tells the kernel to not reclaim already used ram from other application hastly

`audit=0` disable logging of syscalls, logins, logouts, file access stuff and other stuff. TLDR: reduce cpu overhead

`pcie_aspm=off` same as amdgpu.aspm but more

`ignore_rlimit_data` tells the kernel to ignore resource limits for applications

`split_lock_detect=off` disables detection of splitlocks, which will be logged when detected with the other settings

`split_lock_mitigate=0 `when enabled the kernel throttels applications which makes splitlocks, when disabled the application will run at fullthrottle and the kernel imposes no penalty

`preempt=full` sets the preemption model for the kernel reduces latency of input/output devices But needs kernel which is compiled with the PREEMPT_DYNAMIC option enabled. But every kernel should have this by default so no worries. you can check with `uname -a`

`libahci.ignore_sss=1` tells the SATA controller to ignore the Staggered Spin-Up bit for HDD and also has some benefits for SSDs TLDR: faster spinup and less latency.

`loglevel=3` tells the kernel to only output logs with level 3 or higher at boot process

`rd.systemd.show_status=false` Hides service start messages during the ramdisk phase.

`transparent_hugepage_tmpfs=never` Disables THP tmpfs (temporary file systems stored in RAM) So that there are no files which are stored as THPs in RAM otherwise same as with transparent_hugepage happens.

`amdgpu.dcdebugmask=0x4` disable Display Stream Compression DSC

# Sysctl settings
## My Settings

First of all i want to show you my sysctl:

```
vm.swappiness=1
net.core.busy_read=50
vm.max_map_count=2147483642
vm.vfs_cache_pressure=50
vm.dirty_ratio=80
vm.dirty_background_bytes = 67108864
net.ipv4.tcp_mtu_probing=1
vm.page_lock_unfairness=3
kernel.printk_devkmsg=off
vm.stat_interval=10
vm.zone_reclaim_mode=0
vm.compaction_proactiveness=0
vm.overcommit_memory=1
kernel.threads-max=1073741823
kernel.split_lock_mitigate=0
vm.dirty_writeback_centisecs=60
net.core.rmem_max=16777216
net.core.wmem_max=16777216
net.core.rmem_default=8388608
net.core.wmem_default=8388608
vm.unprivileged_userfaultfd=1
kernel.nmi_watchdog=0
kernel.unprivileged_userns_clone=1
kernel.printk = 3 3 2 3
kernel.kptr_restrict = 1
net.ipv4.tcp_congestion_control=bbr
net.core.netdev_max_backlog = 16384
```

## Descriptions of each parameter I used and why

`vm.swappiness` Says how strong the pressure is to put stale stuff in memory into Swap the lower the less the pressure

`net.core.busy_read=50` By setting this value to 50 (which represents 50 microseconds), you are telling the kernel: "When a process asks to read from a network socket and no data is there, don't put the process to sleep immediately. Instead, keep the CPU actively looping (polling) for up to 50 microseconds to see if data arrives."

`vm.max_map_count=2147483642` extend max available Virtual Memory Areas per process

`vm.vfs_cache_pressure=50` lower memory reclaim pressure from reclaiming memory used for caching directory and inode objects

`vm.dirty_ratio=80` kernel can use up to 80% of the ram for delaying of writing of files to disk before hanging and writing everything to disk. The pro is smoother operation but the downside is possible dataloss on powerloss.

`vm.dirty_background_bytes = 67108864` bytes from what point on the delayed file writes to disk from ram start in the background without hanging the system

`net.ipv4.tcp_mtu_probing=1` tells pc to probe mtu size over the network

`vm.page_lock_unfairness=3` this tells how often the same cpu core can steal the lock to a specific memory page one after another if that memory page is shared between more than 1 core. The lower the value the more fair the locking of the memory page is and less latency and the higher the more unfair and more throughput. Default is 5, i set it to 3 for nice tradeoff.

`kernel.printk_devkmsg=off` disable kernel's logging system for userspace applications

`vm.stat_interval=10` This parameter controls how often (in seconds) the kernel's virtual memory (VM) subsystem wakes up to recount and update its internal statistics.

`vm.zone_reclaim_mode=0` same as the command line parameter, maybe you dont even need this here if you have the commandline parameter

`vm.compaction_proactiveness=0` disables ram memory page defragmentation, this disables kcompactd which repeatedly checks and defragments the ram. Which can cause stutter, lag and hangs.

`vm.overcommit_memory=1` tell applications they can commit as much as they want. TLDR: Always say yes to memory allocations.

`kernel.threads-max=1073741823` sets max system thread count.

`kernel.split_lock_mitigate=0` same as in kernl parameters, very likely not even needed here

`vm.dirty_writeback_centisecs=60` tries to flush dirty pages each 0.6 seconds to the disk. This is a lot more often than the default but this is the best option when combined with `vm.dirty_ratio=80` where we never really are hanging hard to write everything but with `vm.dirty_background_bytes = 67108864` which is very low when we start writing stuff to disk early and frequently but always in the background, so we keep stuff consistent and can guarantee no lag spikes, sutters or hangs from this. It does not happen until 80% of ram is full with it, which will never happen. Btw the kernel can reclaim these dirty memory pages all the time when ram is needed.

`net.core.rmem_max=16777216` set max TCP/UDP receive buffer size. 

`net.core.wmem_max=16777216` set max TCP/UDP send buffer size. 

`net.core.rmem_default=8388608` set default TCP/UDP receive buffer size. 

`net.core.wmem_default=8388608` set default TCP/UDP send buffer size. 

`vm.unprivileged_userfaultfd=1` lets non-root applications to handle their own page faults, its is faster, but is also a security risk. So decide for yourself. But it brings less stutter and faster performance.

`kernel.printk = 3 3 2 3` limits log messages to the important stuff, less overhead

`kernel.kptr_restrict = 1` restricts viewing of kernel pointers to application which have CAP_SYSLOG capability or have root rights.

`net.ipv4.tcp_congestion_control=bbr` tcp congestion control algorithm bbr, better latency

`net.core.netdev_max_backlog = 16384` increasing buffer where the kernel stores packets after they’ve been pulled off the physical network card (NIC) but before the CPU has had a chance to process them.

# Sched Ext schedulers

There are a lot of different schedulers which can help for specific usecases you might have.
The default EEVDF is more build for throughput than responsiveness and gaming.

So for gaming i would recommend LAVD in performance mode or PANDEMONIUM, you can find them in the `scx-scheds` package in arch. Or just build them from source for example: https://github.com/sched-ext/scx or https://github.com/wllclngn/PANDEMONIUM

By default cachyOs uses `scx-manager` which is a gui which makes configuring them easier.

What you have to look out for is that you don't use ananicy while using most of the scx sched-ext scheduler because ananicy will change nice levels, io levels and so on. CachyOs has ananicy enabled by default. Here i would recommend to only use it when you use EEVDF scheduler and not with `scx-scheds` 

You can disable ananicy with: `systemctl disable --now ananicy.service`

# Lact

We also want to optimize the GPU performance.

With lact this is very easy.

just dowload it and then max the power usage limit to max or however you desire.
And for the most imporant part, you need to set the performance level to `manual` and then the power profile mode to `3D_FULLSCREEN` or `COMPUTE`.

Or even better `CUSTOM` when you want to configure it yourself.

Important is here to set performance level to `manual` otherwise power profile mode does nothing.

You can of course also play around with the other stuff in lact.

## Important

performance level to `high` does NOT always boost frequencies to the highest clock rate.


# Other optimizations

Another thing i noticed is that some application like firefox (in my case librewolf) set some threads/processes to Realtime priority, which can then not be handelt by EEVDF or sched ext schedulers and can lead to a lot of stutters.
And they use RTKit to set this RT priority. So we can just mask it with `sudo systemctl mask rtkit-daemon.service` but pay attention that your compositor still has RT prio, to have low latency otherwise that can be a problem. For KDE plasma this is no Problem because it sets its PRIO to RT with the Capabilites of linux. So they are not dependent on rtkit.
The only thing is that pipewire also uses rtkit and will now not have no RT prio. But this should not be problem with LAVD or PANDEMONIUM.
But if you still have issues then try increasing the Quantum size of the buffers in pipewire. 

# Disclaimer

If you found some issues here or have other optimizations which might be useful then you can create a Issue/PR if you want to.
