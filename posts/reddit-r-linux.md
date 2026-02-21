# [Bug Report] Kernel 6.17 is unusable on AMD Barcelo (Renoir) APU — 15/15 boots crash, 0 watchdog triggers

I've spent a full day methodically testing and documenting a hard freeze regression in kernel 6.17 on my AMD Ryzen 7 5825U (Barcelo/Renoir APU, PCI ID `1002:15e7`).

## The numbers

- **15 consecutive boots on kernel 6.17** → ALL hard-froze (100% crash rate)
- **2 boots on kernel 6.8** → both stable (0% crash rate)
- Same hardware, same boot parameters (`amdgpu.runpm=0 amd_pstate=passive`), same day
- Median time to freeze: ~7 minutes (range: 27s to 43min)

## What makes this report different

I didn't just report "my computer freezes." I set up comprehensive monitoring:

- 5-second interval system snapshots (CPU/GPU temp, RAM, load, dmesg, PCIe)
- NMI watchdog, softlockup panic, hung task panic — ALL enabled
- kdump configured, pstore configured, persistent journal with 5s sync

**Every single mechanism failed to detect the freeze.** The NMI watchdog — a non-maskable interrupt — didn't fire. This means the freeze happens below the kernel level, consistent with a PCIe bus hang caused by GPU firmware lockup. On an APU where CPU and GPU share the die, this kills the entire system.

The system was completely healthy at the moment of each freeze: 42°C CPU, 41°C GPU, 92% RAM free, load 0.4, zero dmesg errors.

## Full write-up with infographics and data

I've published the complete analysis with 5 infographics, all monitoring data, and root cause analysis:

**GitHub**: https://github.com/DiegoNogueiraDev/kernel-617-amdgpu-freeze-bug

## Affected hardware

- AMD Barcelo (Renoir family) — `1002:15e7`
- Likely also affects: Cezanne, Lucienne (same gfx9/Vega architecture)
- Similar reports exist for Cezanne on kernel 6.15 (Arch forums)

## Workaround

Revert to kernel 6.8. Pin it in GRUB:
```bash
sudo grub-set-default "Advanced options for Ubuntu>Ubuntu, with Linux 6.8.0-100-generic"
sudo update-grub
```

Bug filed on freedesktop.org GitLab (amdgpu team). If you have similar hardware and are experiencing freezes on recent kernels, please add your data to the issue.
