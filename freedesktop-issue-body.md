## Summary

Complete system hard freeze (no kernel panic, no oops, no watchdog trigger) occurs within minutes of booting on kernel 6.17.0-14-generic with AMD Barcelo (Renoir) APU. Kernel 6.8.0-100-generic is completely stable on the same hardware with the same boot parameters. The freeze is 100% reproducible.

## System Information

- **Distro**: Ubuntu 24.04.4 LTS (Noble Numbat)
- **Hardware**: VAIO VJFE69F11X-B0121H
- **CPU**: AMD Ryzen 7 5825U with Radeon Graphics
- **GPU**: AMD Barcelo `[1002:15e7]` (rev c1) — RENOIR, ATOM BIOS `113-BARCELO-004`
- **RAM**: 32 GB DDR4
- **BIOS**: V1.20.X (2024-10-11)
- **Mesa**: 25.2.8
- **Display**: GNOME / Wayland

## Kernels Tested

| Kernel | Status | Boots | Crash Rate |
|--------|--------|:-----:|:----------:|
| 6.8.0-100-generic | **STABLE** | 2 | 0% |
| 6.17.0-14-generic | **CRASHES** | 15 | **100%** |

Both kernels tested with identical boot parameters: `amdgpu.runpm=0 amd_pstate=passive`

## Reproduction

15 consecutive boots on 6.17 — ALL ended in hard freeze requiring power reset:

| Boot | Duration before freeze |
|:----:|:----------------------:|
| 1 | 7m 46s |
| 2 | 10m 03s |
| 3 | 14m 28s |
| 4 | 0m 27s |
| 5 | 42m 57s |
| 6 | 3m 50s |
| 7 | 2m 10s |
| 8 | 6m 50s |
| 9 | 2m 47s |
| 10 | 1m 56s |
| 11 | 3m 38s |
| 12 | 9m 08s |
| 13 | 9m 50s |
| 14 | 8m 38s |
| 15 | 2m 21s |

Median time to freeze: ~7 minutes. Range: 27 seconds to 43 minutes.

2 boots on kernel 6.8 with identical parameters: both stable, no crash.

## Monitoring Data at Time of Freeze

A monitoring script running at 5-second intervals captured system state immediately before every freeze:

- CPU temp: 42-44°C (normal)
- GPU temp: 41-43°C (normal)
- SSD temp: 28°C (normal)
- RAM: 29+ GB available out of 32 GB (no memory pressure)
- Swap: 0% used
- Load: 0.4-0.7 (idle)
- GPU usage: 0-6% (idle), power state D0
- dmesg: No errors, no warnings, no amdgpu messages before freeze
- PCIe: No errors

**The system is completely healthy at the moment of the freeze.**

## Kernel Safety Mechanisms — ALL Silent

| Mechanism | Configuration | Triggered? |
|-----------|--------------|:----------:|
| NMI watchdog | `nmi_watchdog=1` | **NO** |
| Softlockup detector | `kernel.softlockup_panic=1` | **NO** |
| Hung task detector | `kernel.hung_task_panic=1` | **NO** |
| kdump | `USE_KDUMP=1`, crashkernel allocated | **NO** |
| pstore | Configured | **EMPTY** |
| Journal | Persistent + 5s sync | **Cuts off abruptly** |

The fact that the NMI watchdog (a non-maskable interrupt) does not fire indicates the freeze occurs **below the kernel level** — consistent with a PCIe bus hang caused by a GPU firmware lockup. On an APU where CPU and GPU share the die, this freezes the entire system.

## amdgpu Init

Both kernels show `MODE2 reset` during initialization (normal for this hardware after power cycle):

**6.17 (crashes after boot):**
```
amdgpu 0000:04:00.0: amdgpu: initializing kernel modesetting (RENOIR 0x1002:0x15E7 0x2782:0x1100 0xC1)
amdgpu: ATOM BIOS: 113-BARCELO-004
amdgpu 0000:04:00.0: amdgpu: Trusted Memory Zone (TMZ) feature enabled
amdgpu 0000:04:00.0: amdgpu: MODE2 reset
amdgpu 0000:04:00.0: amdgpu: VRAM: 512M
```

**6.8 (stable):**
```
[drm] initializing kernel modesetting (RENOIR 0x1002:0x15E7 0x2782:0x1100 0xC1)
amdgpu: ATOM BIOS: 113-BARCELO-004
amdgpu 0000:04:00.0: amdgpu: Trusted Memory Zone (TMZ) feature enabled
amdgpu 0000:04:00.0: amdgpu: MODE2 reset
amdgpu 0000:04:00.0: amdgpu: VRAM: 512M
```

Init sequence is identical. 6.8 remains stable afterward; 6.17 does not.

## Related Issues

- #1318 — Renoir instability & graphics stack freeze
- Arch Linux: [Total freeze since kernel 6.15 on Cezanne (same die as Barcelo)](https://bbs.archlinux.org/viewtopic.php?id=306429)
- Upstream: [GPU not detected since 6.16.4 — commit c97636cc83d4](https://www.mail-archive.com/amd-gfx@lists.freedesktop.org/msg128816.html)
- Upstream: [PATCH v5 — Address amdgpu reload issues in APUs](http://www.mail-archive.com/amd-gfx@lists.freedesktop.org/msg132803.html) (touches Renoir reset handling)

## Steps to Reproduce

1. Boot Ubuntu 24.04.4 on a system with AMD Ryzen 7 5825U (Barcelo/Renoir APU)
2. Use kernel 6.17.0-14-generic with parameters: `amdgpu.runpm=0 amd_pstate=passive`
3. Use the desktop normally (GNOME/Wayland)
4. System will hard-freeze within 0-43 minutes (median ~7 minutes)
5. Boot kernel 6.8.0-100-generic with identical parameters — no freeze

## Bisect Suggestion

The regression is between kernel 6.8 and 6.17. A bisect of the amdgpu driver changes in this range — particularly for gfx9/Renoir/Barcelo, SMU, or PSP code paths — would likely identify the offending commit.

## Attachments

Full journal logs, monitor logs (5-second interval snapshots), and complete dmesg from both kernels available on request.
