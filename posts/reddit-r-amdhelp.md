# AMD Ryzen 7 5825U (Barcelo) — Hard freeze on kernel 6.17, stable on 6.8. Confirmed amdgpu driver regression.

## Hardware
- AMD Ryzen 7 5825U (Barcelo APU, PCI ID `1002:15e7`)
- 32GB DDR4, Ubuntu 24.04.4 LTS, GNOME/Wayland

## Problem
Kernel 6.17.0-14-generic causes complete system hard freeze within minutes. No panic, no oops, no SysRq response. Only hard power reset works.

## Testing (same day, same hardware, same boot parameters)
- **Kernel 6.17**: 15 boots → 15 crashes (100%)
- **Kernel 6.8**: 2 boots → 0 crashes (0%)

I had full monitoring running (temps, RAM, GPU, dmesg) at 5-second intervals. System was perfectly healthy at every crash. NMI watchdog, softlockup panic, hung task panic — all enabled, none triggered. This indicates a PCIe bus hang from GPU firmware lockup.

## Fix
Revert to kernel 6.8:
```bash
sudo grub-set-default "Advanced options for Ubuntu>Ubuntu, with Linux 6.8.0-100-generic"
sudo update-grub
```

## Full write-up
https://github.com/DiegoNogueiraDev/kernel-617-amdgpu-freeze-bug

This likely affects all Renoir-family APUs (Barcelo, Cezanne, Lucienne). If you have similar hardware and freezes on recent kernels, check the link above.
