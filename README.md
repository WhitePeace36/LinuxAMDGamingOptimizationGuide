# Introduction

## Disclaimer

If you found some issues here or have other optimizations which might be useful then you can create a Issue/PR if you want to.

Furthermore NOTHING of this is mandatory and you are free to apply and use what you want. I am just documenting what i found, use and some things which made problems for me.

## Linux AMD Gaming OptimizationGuide

This Repo has descriptions of how to optimize the gaming experience on Linux with AMD cpu and amd gpu.

This configuration i use is NOT meant to save power or be extra secure. Its also NOT meant to drastically improve performance.

The most imporant part i want to achieve is lower input/output latency and consistency !

Because a few more FPS won't matter at all, you might not even notice it, but when we optimize for consistency then we have the goal to keep the frametime as steady as possible and that is very noticeable.

Like the latency, the latency for the mouse or generally the system is also very noticable but not needed at all for server, so this is for desktop use only. 

Even when you still can use some stuff fore the server, a lot of it would need to be changed, even into the other direction (so for more throughput instead of latency).

### Generally

Latency and throughput are always tradeoffs. It either goes into one direction or the other, so you will have to decide for yourself.

For some of the settings here especially sysctl you will have to uninstall the `cachyos-settings` as this package already applies settings which will override the ones below.

## My System Specs

For context i will show my pc specs here.

```
Operating System: CachyOS Linux 
KDE Plasma Version: 6.6.4
KDE Frameworks Version: 6.25.0
Qt Version: 6.11.0
Kernel Version: 7.1.0-rc2-273-tkg-eevdf-llvm-ga293ec25d59d (64-bit)
Graphics Platform: Wayland
Processors: 16 × AMD Ryzen 7 5800X3D 8-Core Processor
Memory: 32 GiB of RAM (31,3 GiB usable)
Graphics Processor: AMD Radeon RX 7900 XTX
Manufacturer: Gigabyte Technology Co., Ltd.
Product Name: X570 AORUS MASTER
System Version: -CF
```

For the CPU scheduler i am using. I am switching between my own Lunar scheduler which i wrote, LAVD in gaming mode and PANDEMONIUM. Just try which you like the most. All 3 are great choices.

But i would recommend Lunar the most of course and it is also the best in my opinion for the desktop use case.

# Bios Settings

## Settings Bios Settings
Lets start off with bios settings. These play a really big role in performance.

First of all you want to enable your XMP profile for your RAM. 

Then do these settings:

`Resizeable Bar - Auto/Enable`

`Above 4G Decoding - Enable`

`Amd fTPM - Disable` There has been some bugs where this caused some stutters, but i don't know if this is fixed already. Better to turn it off.

`CSM Supoort - DISABLE `!!! This is needed in disable for resizable bar support !

`Core Performance Boost - auto/enable`

`CPPC - ENABLE`

`CPPC preferred cores - ENABLE`

`Global C state Control - ENABLE`

`AMD cool & quiet - ENABLE` this and some others are very important to being able to use the amd-pstate or amd-pstate-epp cpufreq driver, and no it is no energy save mode. I know. I also thought so at the beginning until i saw an interview of a amd engineer which said that this option should strictly speaking not exist anymore because this was some feature from FX series cpus but some motherboard manufacturers just kept the setting in the bios and used it to enable the PState modes for the CPU, so control the performance states. Which makes no sense at all but it is like that.

Right around amd cool & quite also needs to be something like this:

`PState - PState0`

`XHCI Handoff - DISABLE` This setting should be somewhere in the usb settings. This can cause really nasty stutters in some games.

Also dont forget to set the your fan curves of the case and cpu cooler in the fan section in the bios.

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

This will be very imporant for the kernel commandline parameter amd_pstate, which can be `passive`,`active` or `guided`.

Here the best performant is `active`. Which lets the hardware handle boosting of the cpu frequency BUT this does not always work, some hardware just cannot boost itself.
That is why its important to have some tools like mission-center with which you can check if the cpu frequency is boosted above the base clock. If `amd_pstate=active` is set and the cpu frequency is not being boosted above the base frequncy, then use `amd_pstate=passive`.

# CPU Frequency Scaling

The cpu governer is like a power profile which you can set for the cpu to tell him how aggressive to boost the cpu frequencies.

You can check your used governers for every core like this:

`cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor`

might look something like this:

```
ondemand
ondemand
ondemand
ondemand
ondemand
ondemand
ondemand
ondemand
ondemand
ondemand
ondemand
ondemand
ondemand
ondemand
ondemand
ondemand
```

With this you can see all available governers for your cpu:

`cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors`

might look like this:

`conservative ondemand userspace powersave performance schedutil`

What we want for maximal performance is the performance profile.

we can set that with:

`echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor` or `cpupower` or some other tool you might have.

the we can read again the used governer with `cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors` to validate if it changed correctly.

and might look like this:

```
performance
performance
performance
performance
performance
performance
performance
performance
performance
performance
performance
performance
performance
performance
performance
performance
```

But now it is only temporary until the next restart of the pc. What we can do is make a systemd service file which executes on startup and then starts a script with root privileges.
Which then in turn sets the governer to performance.

## Make permanent 

My Service file looked like this:

```
[Unit]
Description=Set some system tweaks
[Service]
ExecStart=/home/someone/Autostart/tweaks.sh
[Install]
WantedBy=multi-user.target
```

you can make a file like `tweaks.service` and save the stuff from above in it. Then save it in `/etc/systemd/system/`.

Then you need to exchange the path at `ExecStart` with your own of the script. The script should have as owner and group to `root` and permissions `744` for security, because this file is being executed as root and when don't want anyone not being root to change the file.

then you can make the script:

```
#! /bin/bash

echo performance | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

and save it in some file like `tweaks.sh`.

But you might also be able to add in to a file in `/etc/tmpfiles.d/` like for example `/etc/tmpfiles.d/gpu.conf` like described below in the Tmpfiles section.

Then we just need to start and enable it with `sudo systemctl enable --now tweaks.service`

and you should be good to go. you can check the status of the script with `systemctl status tweaks.service`

## Additional information

https://wiki.archlinux.org/title/CPU_frequency_scaling

# Kernel Commandline parameter

First of all i want to show my command line parameters:

```
amd-pstate=passive amdgpu.aspm=0 amdgpu.audio=0 nmi_watchdog=0 nowatchdog processor.max_cstate=1 transparent_hugepage=always vm.zone_reclaim_mode=0 audit=0 pcie_aspm=off ignore_rlimit_data split_lock_detect=off split_lock_mitigate=0 preempt=full libahci.ignore_sss=1 loglevel=3 rd.systemd.show_status=false transparent_hugepage_tmpfs=always amdgpu.dcdebugmask=0x4
```

Important note!!! The transparent hugepage settings need most of the virtual memory(vm) settings from the sysctl section and the settings from the tmpfiles section to work the best. Otherwise they can be counterproductive.

## Descriptions of each parameter I used and why

`amd-pstate` as already described above tells the kernel if the kernel should use control the cpu frequency or the hardware itself. I have amd-pstate=passive because my hardware is not able to boost itself and is otherwise locked to the base frequency.

`amdgpu.aspm=0` is used to disable the PCI Express power-saving for amdgpus otherwise the PCI Express link has a wake up latency when it goes into idle state and it wants to wakeup

`amdgpu.audio=0` to disable the audio over HDMI/DP, i dont need it. I have an external DAC.

`nmi_watchdog=0` is for Hard lockup detector and Soft lockup detection of Non-Maskable Interrupts, which uses a watchdog to detect that which needs to run in a set interval. When you disable it you free up some cpu time.

`nowatchdog` same as above but for all watchdogs

`processor.max_cstate=1` sets the max C state or better said idle state to C1. It normally goes until C6 or even lower. The higher the number the deeper the sleep state. The deeper the sleep state the higher the wakup latency, to minimize the latency we do max cstate to 1 and i think you need the Global C state Control option enabled in the bios to really benefit from it.

`transparent_hugepage=always` always enable thp pages. When we combine them with the tmpfile settings and the sysctl virtual memory (vm) settings then they cooperate very much.

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

`transparent_hugepage_tmpfs=always` always enable thp pages. When we combine them with the tmpfile settings and the sysctl virtual memory (vm) settings then they cooperate very much.

`amdgpu.dcdebugmask=0x4` disable Display Stream Compression DSC. For this you would need to check if the DP or HMDI cable you are using and the port on your Monitor and GPU are fast enough for the resolution and refresh rate your are using. Because it could be that you can use the current configuration of resolution and resfresh rate only with DSC. So in this case you should not disable it.

## Validating

You can check with which kernel paramters the kernel you are booted into was started with/ is using right now. 

With this command:

`cat /proc/cmdline `

# Sysctl settings
## My Settings

First of all i want to show you my sysctl:

```
vm.swappiness=150
net.core.busy_read=0
vm.max_map_count=2147483642
vm.vfs_cache_pressure=50
vm.dirty_bytes = 536870912
vm.dirty_background_bytes = 67108864
net.ipv4.tcp_mtu_probing=1
vm.page_lock_unfairness=3
kernel.printk_devkmsg=off
vm.stat_interval=10
vm.zone_reclaim_mode=0
vm.page-cluster = 0

vm.compaction_proactiveness = 40
vm.watermark_scale_factor = 500
vm.watermark_boost_factor = 15000
vm.defrag_mode = 0

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

Some of these like `vm.dirty_ratio=80`, `net.core.rmem_default=8388608` and `net.core.wmem_default=8388608` are more meant for systems with higher RAM amount like 32GB because they essentialy make programms use a little more ram in the case of the receive and sende buffer, or profit from more available/unused ram like `vm.dirty_ratio=80` but you are free to adjust them.

The only thing i can recommend when you adjust `vm.dirty_ratio` then maybe also think about `vm.dirty_background_bytes` and `vm.dirty_writeback_centisecs`.
## Descriptions of each parameter I used and why

`vm.swappiness=150` Says how strong the pressure is to put stale stuff in memory into Swap the lower the less the pressure. This makes only sense to use with a swapfile with zswap or some other swap method like zram. Where zram is the best and fastest method.

`net.core.busy_read=0` By setting this value to 50 (which represents 50 microseconds), you are telling the kernel: "When a process asks to read from a network socket and no data is there, don't put the process to sleep immediately. Instead, keep the CPU actively looping (polling) for up to 50 microseconds to see if data arrives." This setting is a tradeoff. As little network latency as possible for network heavy applications but you are sacrificing cpu time which could be spent on other stuff.
I had it on 50 microseconds for a long time but now that i write a cpu scheduler i have turned it back to 0 because for the usecase i am chasing, which is as smooth desktop experience as possible, these 50 microseconds make a huge difference.

`vm.max_map_count=2147483642` extend max available Virtual Memory Areas per process

`vm.vfs_cache_pressure=50` lower memory reclaim pressure from reclaiming memory used for caching directory and inode objects

`vm.dirty_bytes = 536870912` kernel can use up to 536870912 Bytes of the ram for delaying of writing of files to disk before hanging and writing everything to disk. You can go lower or higher. Doesn't really make that much of a difference. The only reason why i found to keep it low is that when a programm calls fsync that it does not need to flush out gigabytes at a time.

`vm.dirty_background_bytes = 67108864` bytes from what point on the delayed file writes to disk from ram start in the background without hanging the system

`net.ipv4.tcp_mtu_probing=1` tells pc to probe mtu size over the network

`vm.page_lock_unfairness=3` this tells how often the same cpu core can steal the lock to a specific memory page one after another if that memory page is shared between more than 1 core. The lower the value the more fair the locking of the memory page is and less latency and the higher the more unfair and more throughput. Default is 5, i set it to 3 for nice tradeoff.

`kernel.printk_devkmsg=off` disable kernel's logging system for userspace applications

`vm.stat_interval=10` This parameter controls how often (in seconds) the kernel's virtual memory (VM) subsystem wakes up to recount and update its internal statistics.

`vm.zone_reclaim_mode=0` same as the command line parameter, maybe you dont even need this here if you have the commandline parameter

`vm.page-cluster = 0` page-cluster controls the number of pages up to which consecutive pages are read in from swap in a single attempt. This is the swap counterpart to page cache readahead. We don't want to have this when we use zram because we don't need to readahead any pages because the zram is memory compressed in memory. So reading is really fast.

`vm.compaction_proactiveness=40` controls how eagerly the kernel defragments free RAM in the background with the `kcompactd` process. Over time free memory gets scattered into small pieces, so even with many GB free there might be no larger contiguous block left. Some allocations (for example from the GPU driver) need such larger blocks. When none is available, the program that asked for it has to wait while the kernel defragments memory right on the spot (direct compaction). THIS is what causes the stutters, not kcompactd itself. The value goes from 0 to 100, default is 20. With 40, kcompactd starts working when the fragmentation score goes above ~70 and stops at ~60. `0` disables background compaction completely, which moves all that work into your game. For me `0` caused hundreds of stalls during gameplay, with 40 they were gone. Don't go too high though, because moving memory pages also briefly disturbs the programs which own them.

`vm.watermark_scale_factor=500` sets how early kswapd starts to free memory in the background. The kernel has 3 watermarks for free memory: min, low and high. When free memory drops below low, kswapd wakes up and frees memory (mostly file cache) until free memory is above high again. Only when free memory reaches min, the program itself has to free memory and waits for it (direct reclaim). The unit is 1/10000 of the RAM, so the default 10 is 0.1% and 500 is 5%. With 32GB RAM that means kswapd starts at ~1.7GB free and stops at ~3.3GB free. This gives the background threads enough headroom, so programs practically never have to free memory themselves. The downside is that cache is dropped a bit earlier and when your RAM is really full with programs the system starts struggling earlier, so it's good to combine it with zswap + swapfile or zram.

`vm.watermark_boost_factor=15000` is the kernel default (some guides set it to 0). RAM is managed in blocks of ~2MB which are reserved either for movable or unmovable memory. When the kernel has to take memory from a block of the wrong type (fallback), that block gets mixed and can never be fully defragmented again. This setting tells the kernel to react to such an event by temporarily raising the high watermark, so kswapd frees extra memory and then wakes kcompactd to clean up. The unit is 1/10000 of the high watermark, so 15000 means up to 150%. `0` disables this reaction.

`vm.defrag_mode=0` only exists on newer kernels (check with `sysctl vm.defrag_mode`). It makes the kernel try much harder to not mix memory blocks in the first place. Instead of falling back into a block of the wrong type, it frees and defragments memory first. And that is the issue. The trying to defragment memory first which leads to stalls. Which is want we don't want.

`vm.overcommit_memory=1` tell applications they can commit as much as they want. TLDR: Always say yes to memory allocations.

`kernel.threads-max=1073741823` sets max system thread count.

`kernel.split_lock_mitigate=0` same as in kernl parameters, very likely not even needed here

`vm.dirty_writeback_centisecs=100` tries to flush dirty pages each 1 seconds to the disk. This is a lot more often than the default but this is the best option when combined with `vm.dirty_bytes = 536870912` where we rarely are hanging hard to write everything but with `vm.dirty_background_bytes = 67108864` which is very low when we start writing stuff to disk early and frequently but always in the background, so we keep stuff consistent and can guarantee as little lag spikes as possible. It does not happen until 536870912 bytes of ram is full with it or a programm calls fsync. Btw the kernel can reclaim these dirty memory pages all the time when ram is needed. But the dirty pages first need to get written to disk which will cause stutters for the programm which tries to allocate memory.

`net.core.rmem_max=16777216` set max TCP/UDP receive buffer size. 

`net.core.wmem_max=16777216` set max TCP/UDP send buffer size. 

`net.core.rmem_default=8388608` set default TCP/UDP receive buffer size. 

`net.core.wmem_default=8388608` set default TCP/UDP send buffer size. 

`vm.unprivileged_userfaultfd=1` lets non-root applications to handle their own page faults, its is faster, but is also a security risk. So decide for yourself. But it brings less stutter and faster performance.

`kernel.printk = 3 3 2 3` limits log messages to the important stuff, less overhead

`kernel.kptr_restrict = 1` restricts viewing of kernel pointers to application which have CAP_SYSLOG capability or have root rights.

`net.ipv4.tcp_congestion_control=bbr` tcp congestion control algorithm bbr, better latency

`net.core.netdev_max_backlog = 16384` increasing buffer where the kernel stores packets after they’ve been pulled off the physical network card (NIC) but before the CPU has had a chance to process them.

### Validating

You can check how often programs had to wait for memory with:

```
grep -E "compact_stall|allocstall" /proc/vmstat
```

The counters start at 0 on every boot and only go up. `compact_stall` counts how often a program had to wait for on-the-spot defragmentation, `allocstall_*` how often a program had to free memory itself. Note the values before and after a gaming session. With good settings they should barely or not at all increase.

If the counters increase at all or very fast then some programms experience stutters, hitches or mini freezes.

This can be tuned with these values:

```
vm.compaction_proactiveness = 40
vm.watermark_scale_factor = 500
vm.watermark_boost_factor = 15000
vm.defrag_mode = 0
```

The options are described above.
To use these options the best i would recommand zram instead of any other swap method.

# Tmpfiles

For these settings we need to create the file /etc/tmpfiles.d/thp.conf for systemd systems.

This settings are complementative with these kernel command line parameter:

```
transparent_hugepage=always transparent_hugepage_tmpfs=always
```
and all the virtual memory(vm) settings from above.

These settings make sure to have as little stalls as possible when a programm tries to allocate memory. 
So we try to make use of `khugepaged, kswapd, kcompactd` as much as possible. To keep the memory defragmented and ready in the background instead at the time of allocation. This will reduce memory related stutters to a minimum or completely eliminate them.

Here we put these settings:

```
w /sys/kernel/mm/transparent_hugepage/defrag            - - - - defer
w /sys/kernel/mm/transparent_hugepage/shmem_enabled     - - - - always
w /sys/kernel/mm/transparent_hugepage/shrink_underused  - - - - 0
w /sys/kernel/mm/transparent_hugepage/khugepaged/pages_to_scan - - - - 8192
w /sys/kernel/mm/transparent_hugepage/khugepaged/scan_sleep_millisecs - - - - 2500
w /sys/module/zswap/parameters/enabled - - - - 0
```

With the `w` we write values into these files at startup.


`/sys/kernel/mm/transparent_hugepage/defrag            - - - - defer ` this controls what happens when a programm tries to allocate a thp page and none is available. In this case it will not block and try to defragment the memory but will for the time being give the programm a 4kb page and prepares the thp page in the background. So we can garuantee as little stutters as possible.

`/sys/kernel/mm/transparent_hugepage/shmem_enabled     - - - - always` always use thp for shared memory.

`/sys/kernel/mm/transparent_hugepage/shrink_underused  - - - - 0` don't shrink underused thp pages. This will lead to a little more ram usage but will help with smoothness.

`/sys/kernel/mm/transparent_hugepage/khugepaged/pages_to_scan - - - - 8192` This is a setting for the `khugepaged` daemon which is the daemon which does compress normal memory pages to THP pages in the background. It says how much 4kb pages it will collaps to thp pages per wake.

`/sys/kernel/mm/transparent_hugepage/khugepaged/scan_sleep_millisecs - - - - 2500` This changes the frequency with which the `khugepaged` is woken and will create thp pages in the background.

`/sys/module/zswap/parameters/enabled - - - - 0` With this we disable zswap to make use of zram.

# Sched Ext schedulers

There are a lot of different schedulers which can help for specific usecases you might have.
The default EEVDF is more build for throughput than responsiveness and gaming.

So for gaming i would recommend https://github.com/WhitePeace36/Lunar_sched or https://github.com/WhitePeace36/Elara . `Lunar_sched` is intended to be used without ananicy and works with classifying threads by their behavior.

But this is not 100% perfect but you don't need todo anything you don't need to nice anything or such. It handles everything automatically. Except realtime threads. It cannot handle these. Same as every other scx scheduler.

Then there is `Elara`. Which is intended to be used with ananicy. It was little inspired from the windows scheduler but a better version of it. Where the nice value is used for prio not for cpu time like with the default schedulers. If you want to know more you can look into the description of the scheduler. You might also want to adjust the ananicy profiles. The preconfigured stuff will not fit great with this scheduler.

`Band 0` should be something like xwayland, kwin, pipewire or such stuff. With a nice value of -12 or higher.
`Band 1` should be somthing like your game. With a nice value of -5 or such.
`Band 2` are the most default applications like browser and other stuff.
`Band 3` and `Band 4` is for the low prio stuff which you want to run in the background but you don't want to interrupt your other work on the pc. Like for example compiling the kernel.

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

# Custom Kernels

Nowadays its very easy to use custom kernels, because cachyos already offers a wide variety of kernels with different cpu schedulers and optimizations applied.

Therefore you can just use one of them or just try a few of them and see what fits your use case.

If you still want to compile your own kernel because you want to change some things but don't want it to be to complicated then i can only recommend: linux-tkg https://github.com/Frogging-Family/linux-tkg

They make it really easy to configure basic stuff in the `customization.cfg` file. There is even a way to very simply apply your own patched if you want to.

## Tick modes

A kernel can have different tick modes. There are:

`NO_HZ_IDLE` A kernel which lets run idle cpu cores as tickless and as soon as they are used they switch to `HZ_PERIODIC` and get a tick every tick rate you have set.

`HZ_PERIODIC` This always sends the tick to all cores.

`NO_HZ_FULL` when you have a kernel which is compiled with this flag and you have this cmdline parameter `nohz_full` set then the specified core run in tickless mode and get no ticks at all. But you have to at least have 1 core which is not in the list because the kernel needs 1 housekeeping core, which does the work which comes with a tick.

This `nohz_full` cmdline parameter can also be combined with the following others: `rcu_nocbs` and `irqaffinity` to really take advantage of it.

A kernel can be compiled with `NO_HZ_FULL` but uses `NO_HZ_IDLE` by default when `nohz_full` cmdline parameter is not set.

## What we want

We want to have here `NO_HZ_IDLE` which is the default for most kernels or `HZ_PERIODIC`for consistency and regular intervals.

With preferably a tick rate of 1000 for gaming. To make the system more responsive.

To keep stuff consistent. 

Tickless seems nice at first, but it is not really that consistent. You can easily feel it with the mouse. The movement is inconsistent. That comes from the way the rescheduling works there. It does not happen in regular intervals, the rescheduling interval is dynamic in `nohz_full` mode and we don't want that.

## Validate

You can see your tick mode and tick rate the running kernel was compiled with with this command: 

`zcat /proc/config.gz | grep "CONFIG_HZ\|CONFIG_NO_HZ"`


# Graphics driver

For the driver you can just use the default `mesa` one if you want to, except when you want to have bugfixes and other stuff faster then you are also free to try `mesa-git`

One the other hand when you want to really optimize it then i can recommend: `mesa-git` from the frogging family same as the linux-tkg. https://github.com/Frogging-Family/mesa-git

They also make it here very simple to configure it and to compile it yourself.

Here you can apply cpu optimizations and other stuff which you might not have otherwise.

# Limits

## Disclaimer

With limits file you have to be careful to only edit this file if you have some backup live iso usb stick. Because when you set some values too high some application don't know how to behave.
So it can even be that you can't boot into Kde plasma or use the package manager. !!!

But the these things should be save because i am running them myself and it works fine. But DON'T increase them otherwise you can't boot into Kde plasma anymore because the Linux PAM (Pluggable Authentication Modules). Doesn't like too high limits and can lock you out of your system otherwise. You can still edit them from a linux live usb when you messed up. So no worries.

## Optimizations

We might also want to increase some limits.
These are available in `/etc/security/limits.conf`

You might want to add these at the bottom:

```
* soft  nofile  524288
* hard  nofile  524288
* soft  nproc   127461
* hard  nproc   127461
```

- the nofile increases the number of files a process can have open, soft and hard limit. We want this to enable games to have a lot of files open at the same time. per user

- the nproc sets the number of possible open processes, soft and hard limit. per user

The sysctl settings we set above are system wide.


# Realtime prio optimization

Another thing i noticed is that some application like firefox (in my case librewolf) set some threads/processes to Realtime priority, which can then not be handelt by EEVDF or sched ext schedulers and can lead to a lot of stutters.

And they use RTKit to set this RT priority. So we can just mask it with `sudo systemctl mask rtkit-daemon.service` but pay attention that your compositor still has RT prio, to have low latency otherwise that can be a problem. For KDE plasma this is no Problem because it sets its PRIO to RT with the Capabilites of linux. So they are not dependent on rtkit.

The only thing is that pipewire also uses rtkit and will now not have no RT prio. But this should not be problem with LAVD or PANDEMONIUM.

But if you still have issues then try increasing the Quantum size of the buffers in pipewire. 

## General Realtime prio information 

If you are curious which programs have Realtime Priority then you can use `htop` and order the PRI coloumn descendingly. Every process which has a NEGATIVE PRI is a realtime process and either has the SCHED type FIFO or RR (RoundRobin). 

Real time prio processes never get scheduler by the cpu scheduler. They get their timeslice before the rest of the cpu timeslice gets handed to the cpu scheduler to schedule.

 - With RR (RoundRobin) they get always a timeslice from each cpu time slice. So the normal cpu scheduler does not schedule these processes. But the timeslice of RoundRobin processes is still limited so the other processes still get some cpu time. 
 
 - With FIFO (FirstInFirstOut) this changes. Programms which have this sched type can take as much cpu time as they want. So it can starve all other Processess on the system. WE DON'T WANT THIS. 
  TLDR: it can cause freezes, laggs, stutters.

  This is also the reason why we want to disable ananicy because some of the default used rules set some processes to FIFO and RoundRobin which is bad.
  
  Some programs also use the rtkit daemon to set themselves to RT prio and we also don't want that. This is the reason why we want to mask `rtkit-daemon.service`.
