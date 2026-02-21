# [kernel 6.17] Hard freeze regression on AMD Barcelo (Renoir) APU - amdgpu driver

## Summary

Complete system hard freeze (no kernel panic, no oops, no watchdog trigger) occurs
within minutes of booting on kernel 6.17.0-14-generic with AMD Barcelo (Renoir) APU.
Kernel 6.8.0-100-generic is completely stable on the same hardware with the same
boot parameters. The freeze is 100% reproducible.

## Impact

System becomes completely unresponsive — no keyboard, no mouse, no SysRq, no SSH.
Only a hard power reset recovers the machine. This makes kernel 6.17 entirely
unusable on this hardware.

## System Information

- **Distro**: Ubuntu 24.04.4 LTS (Noble Numbat)
- **Hardware**: VAIO VJFE69F11X-B0121H (Positivo Bahia)
- **CPU**: AMD Ryzen 7 5825U with Radeon Graphics
- **GPU**: AMD Barcelo [1002:15e7] (rev c1) — RENOIR, ATOM BIOS 113-BARCELO-004
- **RAM**: 32 GB DDR4
- **SSD**: NVMe SM2P32A8-512GC1 512GB
- **BIOS**: V1.20.X (2024-10-11)
- **Mesa**: 25.2.8
- **Display**: GNOME/Wayland

## Kernels Tested

| Kernel | Status | Notes |
|--------|--------|-------|
| 6.8.0-100-generic | **STABLE** | No crashes in any session |
| 6.17.0-14-generic | **CRASHES** | 15/15 boots ended in hard freeze |

## Reproduction Data (2026-02-21)

**Same boot parameters for both kernels:**
```
quiet splash amdgpu.runpm=0 amd_pstate=passive
```

### 15 consecutive 6.17 boots — ALL crashed:

| Boot | Kernel | Duration before freeze |
|------|--------|-----------------------|
| -16 | 6.17.0-14 | 7m 46s |
| -15 | 6.17.0-14 | 10m 3s |
| -14 | 6.17.0-14 | 14m 28s |
| -13 | 6.17.0-14 | 0m 27s |
| -12 | 6.17.0-14 | 42m 57s |
| -11 | 6.17.0-14 | 3m 50s |
| -10 | 6.17.0-14 | 2m 10s |
| -9 | 6.17.0-14 | 6m 50s |
| -8 | 6.17.0-14 | 2m 47s |
| -7 | 6.17.0-14 | 1m 56s |
| -6 | 6.17.0-14 | 3m 38s |
| -4 | 6.17.0-14 | 9m 8s |
| -3 | 6.17.0-14 | 9m 50s |
| -2 | 6.17.0-14 | 8m 38s |
| -1 | 6.17.0-14 | 2m 21s |

### 2 boots on 6.8 — both stable:

| Boot | Kernel | Duration | Outcome |
|------|--------|----------|---------|
| -5 | 6.8.0-100 | 11m 56s | Clean shutdown by user |
| 0 | 6.8.0-100 | Running | Stable, still up |

**Crash rate: 6.17 = 100% (15/15), 6.8 = 0% (0/2)**

## Monitoring Data at Time of Freeze

A monitoring script running at 5-second intervals captured system state immediately
before the freeze. The last readings before crash showed:

- **CPU temp**: 42-44°C (normal)
- **GPU temp**: 41-43°C (normal)
- **SSD temp**: 28°C (normal)
- **RAM**: 29+ GB available out of 31 GB (no memory pressure)
- **Swap**: 0% used
- **Load**: 0.4-0.7 (idle)
- **GPU usage**: 0-6% (idle)
- **GPU power state**: D0 (active)
- **dmesg**: No errors, no warnings, no amdgpu messages before freeze

**The system was completely healthy at the moment of the freeze.**

## What Does NOT Trigger

The following kernel safety mechanisms did NOT fire:

- NMI watchdog (enabled via `nmi_watchdog=1`)
- Soft lockup detector (`kernel.softlockup_panic=1`)
- Hung task detector (`kernel.hung_task_panic=1`)
- No kernel panic in journal
- No oops or BUG in journal
- kdump was configured but did not trigger (no crashdump generated)
- pstore was empty

**This pattern is characteristic of a PCIe bus hang or GPU firmware lockup that
freezes the entire system below the NMI level.**

## amdgpu Init Comparison

Both kernels show `MODE2 reset` during initialization, which is normal for this
hardware after a power cycle. The init sequence is otherwise identical.

### 6.17 init (crashes):
```
amdgpu 0000:04:00.0: amdgpu: initializing kernel modesetting (RENOIR 0x1002:0x15E7 0x2782:0x1100 0xC1)
amdgpu: ATOM BIOS: 113-BARCELO-004
amdgpu 0000:04:00.0: amdgpu: MODE2 reset
amdgpu 0000:04:00.0: amdgpu: VRAM: 512M
```

### 6.8 init (stable):
```
[drm] initializing kernel modesetting (RENOIR 0x1002:0x15E7 0x2782:0x1100 0xC1)
amdgpu: ATOM BIOS: 113-BARCELO-004
amdgpu 0000:04:00.0: amdgpu: MODE2 reset
amdgpu 0000:04:00.0: amdgpu: VRAM: 512M
```

## Steps to Reproduce

1. Install Ubuntu 24.04.4 on a system with AMD Ryzen 5825U (Barcelo/Renoir APU)
2. Boot kernel 6.17.0-14-generic with parameters: `amdgpu.runpm=0 amd_pstate=passive`
3. Use the desktop normally (GNOME/Wayland)
4. System will hard-freeze within 0-43 minutes (median ~7 minutes)
5. Boot kernel 6.8.0-100-generic with identical parameters — no freeze

## Expected vs Actual

- **Expected**: System should be stable under normal desktop use
- **Actual**: Complete hard freeze requiring power reset, 100% reproducible

## Bisect suggestion

The regression is between kernel 6.8 and 6.17. A bisect of the amdgpu driver
changes in this range (particularly for gfx9/Renoir/Barcelo) would likely
identify the offending commit.

## Attachments

- Full journal from crashed 6.17 boot available on request
- Monitor logs (5-second interval system snapshots) available on request
- `lspci -vnn`, `dmidecode`, full `dmesg` from both kernels available on request
