# PSA: Kernel 6.17 causes 100% reproducible hard freeze on AMD Ryzen 5000U laptops (Barcelo/Renoir APU)

If you have an AMD Ryzen 5000 U-series laptop on Ubuntu 24.04 and are experiencing random hard freezes after a kernel update — this is a confirmed kernel bug.

## My testing

- **15 boots on kernel 6.17.0-14-generic** → ALL hard-froze (mouse, keyboard, SSH, SysRq — nothing responds)
- **2 boots on kernel 6.8.0-100-generic** → both completely stable
- Same hardware, same boot parameters, same day

The freeze happens randomly between 27 seconds and 43 minutes after boot, regardless of workload. The system is completely healthy at the moment of the crash (I had monitoring running at 5-second intervals capturing everything).

## My hardware

- VAIO VJFE69F11X-B0121H
- AMD Ryzen 7 5825U (Barcelo APU)
- Ubuntu 24.04.4 LTS, GNOME/Wayland, Mesa 25.2.8

## Fix

Pin kernel 6.8 as your default:

```bash
sudo grub-set-default "Advanced options for Ubuntu>Ubuntu, with Linux 6.8.0-100-generic"
sudo update-grub
```

## Full analysis

I've documented everything with infographics, monitoring data, and root cause analysis:

https://github.com/DiegoNogueiraDev/kernel-617-amdgpu-freeze-bug

The root cause is a regression in the amdgpu driver that causes the GPU to lock up and hang the PCIe bus, which freezes the entire system (since it's an integrated APU). No kernel watchdog catches it because the freeze happens below the kernel level.

Bug reported upstream on freedesktop.org GitLab and Launchpad.

If you have the same hardware, please upvote the bug report — it helps prioritize the fix.
