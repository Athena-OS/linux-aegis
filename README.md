# linux-aegis

**Pentesting-patched Linux Kernel**

A custom Linux kernel built for offensive security work. Based on [CachyOS](https://cachyos.org/) linux-cachyos-bore, with Kali WiFi injection patches and dual security profiles optimized for different engagement scenarios.

---

## Profiles

linux-aegis ships two distinct build profiles selected at compile time via `_aegis_profile`.

### `offensive` (default)
Performance-first configuration for active pentesting.

- BORE + sched-ext schedulers - low-latency interactive response under heavy tool load
- Full WiFi injection support (mac80211, cfg80211, zd1211rw, rtl8187, carl9170)
- `perf_event_open` unrestricted - required by hashcat, bcc, bpftrace, perf
- dmesg unrestricted - real-time kernel debug output
- Unprivileged user namespaces enabled - required by Flatpak, Bubblewrap, rootless containers
- `-O3` compiler optimization + native CPU tuning
- 1000Hz timer for sub-millisecond scheduling precision

### `hardened`
Security-first configuration for professional engagements on client infrastructure.

- BORE scheduler only (sched-ext disabled - incompatible with hardened patchset)
- All WiFi injection patches retained
- `perf_event_open` restricted to `CAP_PERFMON` - closes a well-documented local privilege escalation surface
- dmesg restricted to privileged users
- IMA (Integrity Measurement Architecture) enabled
- Module versioning and platform keyring for UEFI DB module signing
- `INIT_ON_FREE` enabled - heap zeroing on free eliminates uninitialized memory exposure
- `BUG_ON_DATA_CORRUPTION` - kernel panics on detected memory corruption rather than continuing
- TIOCSTI restricted - blocks terminal injection via `ioctl`
- Lockdown in EFI Secure Boot mode
- kCFI enabled when built with Clang + LTO - forward-edge control flow integrity

---

## Patches

| Patch | Source | Profiles |
|---|---|---|
| BORE + CachyOS scheduler sauce | [CachyOS kernel-patches](https://github.com/CachyOS/kernel-patches) | Both |
| Hardened Linux | [linux-hardened](https://github.com/anthraxx/linux-hardened) | Hardened only |
| WiFi injection (mac80211, cfg80211) | Kali Linux | Both |
| carl9170 sniffer promisc (AR9170) | Kali Linux | Both |
| Intel IOMMU intgpu-off parameter | Kali Linux | Both |
| UEFI DB platform keyring for module signing | Kali Linux / Fedora | Hardened only |

---

## Build

### Prerequisites

```
bc cpio gettext libelf pahole perl python rust rust-bindgen rust-src tar xz zstd
```

For Clang/LTO builds, also install:
```
clang llvm lld
```

### Clone

```bash
git clone https://github.com/Athena-OS/linux-aegis
cd linux-aegis
```

### Build offensive profile (default, GCC)

```bash
makepkg -s --skippgpcheck
```

### Build with Clang + thin LTO (recommended)

```bash
_use_llvm_lto=thin makepkg -s --skippgpcheck
```

### Build with Clang + full LTO (requires 32GB+ RAM)

```bash
_use_llvm_lto=full makepkg -s --skippgpcheck
```

### Build hardened profile with Clang + thin LTO + kCFI

```bash
_aegis_profile=hardened _use_llvm_lto=thin _use_kcfi=yes makepkg -s --skippgpcheck
```

> **Important Note:**
> Due to a bug on NVIDIA, if the kernel is built by kCFI, on NVIDIA GPU-based systems, kernel will panic at boot: https://github.com/NVIDIA/open-gpu-kernel-modules/issues/439
> Waiting for a patchset.

### Install

```bash
sudo pacman -U linux-aegis-offensive-*.pkg.tar.zst \
               linux-aegis-offensive-headers-*.pkg.tar.zst
```

Update your bootloader after installation.

---

## Build Options

All options are set at the top of the PKGBUILD as shell variables and can be overridden on the command line.

| Variable | Default | Description |
|---|---|---|
| `_aegis_profile` | `offensive` | Build profile: `offensive` or `hardened` |
| `_use_llvm_lto` | `none` | Compiler and LTO mode: `none` (GCC), `thin`, `thin-dist`, `full` |
| `_use_kcfi` | `no` | Kernel Control Flow Integrity - requires Clang + LTO, recommended for hardened |
| `_HZ_ticks` | `1000` | Timer frequency: `100`, `250`, `300`, `500`, `1000` |
| `_processor_opt` | *(native)* | CPU optimization: empty = native, `generic`, `zen4` |
| `_cc_harder` | `yes` | Enable `-O3` compiler optimization |
| `_tcp_bbr3` | `no` | Use BBR congestion control instead of CUBIC |
| `_tickrate` | `full` | Tick type: `full`, `idle`, `periodic` |
| `_preempt` | `full` | Preemption model: `full`, `lazy`, `voluntary`, `none` |
| `_hugepage` | `always` | Transparent hugepages: `always` or `madvise` |
| `_use_llvm_lto` | `none` | Clang LTO: `none`, `thin`, `thin-dist`, `full` |
| `_localmodcfg` | `no` | Build only currently loaded modules (requires [modprobed-db](https://aur.archlinux.org/packages/modprobed-db)) |

---

## Compiler and LTO

### GCC vs Clang

Setting `_use_llvm_lto=none` builds with GCC. Any other LTO value switches to Clang automatically.

In terms of runtime kernel quality, GCC and Clang produce essentially equivalent output on x86-64. Both compilers have been heavily tuned for kernel code and the difference in generated instruction quality is typically under 1% in real workloads. The real advantage of Clang is not performance but capability: it unlocks LTO, kCFI, and better sanitizer integration that GCC cannot provide. For linux-aegis, the practical reason to choose Clang is access to these features, not raw instruction quality.

### LTO modes

LTO (Link Time Optimization) allows the compiler to optimize across translation unit boundaries. Without LTO, each `.c` file is compiled and optimized in isolation. The compiler cannot see across file boundaries when making inlining or dead code decisions. With LTO, the compiler sees the entire codebase at link time and can inline across files, eliminate globally dead code, and propagate constants throughout the whole kernel.

`none` - GCC build, no LTO. Fastest to compile, no special hardware requirements. Each file optimized independently.

`thin` - Clang thin LTO. During compilation each translation unit produces a compact summary of its interface alongside the object file. At link time, cross-module optimizations run fully in parallel using these summaries - each module is optimized in the context of the whole program but only loads what it needs. Build time and RAM usage are manageable. Captures roughly 80-90% of the benefit of full LTO.

`full` - Clang full LTO. The entire kernel's intermediate representation is loaded into memory as a single unit and optimized globally in one pass. The compiler has complete visibility with no approximations or summaries - maximum optimization quality. The cost is significant: requires 32GB+ RAM, the final optimization pass is single-threaded, and build time is substantially longer. The practical performance difference over thin LTO is real but small - typically under 1% in real workloads - because kernel hot paths are already well-optimized within their own translation units and have limited cross-file optimization potential.

`thin-dist` - Thin LTO with distributed build support. Only relevant for large distributed build farms.
In summary: full LTO is strictly better than thin LTO in optimization quality, but the difference is marginal for kernel workloads and comes at a high hardware cost. Thin LTO is the practical optimum for most builders.

### kCFI

kCFI (kernel Control Flow Integrity) instruments every indirect function call with a type check at compile time. If an attacker attempts to redirect an indirect call to an unintended target - a standard kernel exploitation technique - the kernel panics instead of continuing execution. Requires Clang + any LTO variant.
Enabled automatically on the hardened profile in CI. Disabled on offensive because the panic-on-violation behavior can interfere with unusual driver interactions that pentesting tools sometimes trigger, and the ~1-2% overhead is not warranted when the goal is tool compatibility rather than defense.

### CI artifacts

CI builds use Clang + thin LTO. The hardened CI build additionally enables kCFI. These are the artifacts uploaded on each push.

To build locally with full LTO:
```bash
_use_llvm_lto=full _aegis_profile=hardened _use_kcfi=yes makepkg -s --skippgpcheck
```

---

## Post-install Verification

```bash
# Confirm kernel version
uname -r

# Confirm BORE is active
cat /sys/kernel/sched_bore

# Confirm active LSM stack (should include apparmor)
cat /sys/kernel/security/lsm

# Confirm timer frequency
grep CONFIG_HZ= /boot/config-$(uname -r)

# Test WiFi injection - put adapter into monitor mode
sudo ip link set wlan0 down
sudo iw dev wlan0 set type monitor
sudo ip link set wlan0 up
sudo airodump-ng wlan0
```

---

## Recommended System Configuration

Some settings cannot be compiled into the kernel and must be applied at runtime. Add the following to `/etc/sysctl.d/99-aegis.conf`:

```ini
# With zram: keep at 100 (fast swap is acceptable, but active working sets
# for hashcat/scanning should stay resident under moderate pressure)
# Without zram (disk swap): set to 10 to protect active tool state
vm.swappiness = 100

# Increase network buffer sizes for high-speed scanning (nmap, masscan, zmap)
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 5000
```

Apply immediately without rebooting:

```bash
sudo sysctl --system
```

---

## Kernel Base

linux-aegis tracks [CachyOS linux-cachyos-bore](https://github.com/CachyOS/linux) releases. The version scheme follows:

```
{kernel_version}-{pkgrel}-aegis-{profile}
```

Example: `6.19.8-1-aegis-offensive`

CachyOS provides a pre-patched source tarball that already includes:

- BORE scheduler integration
- ADIOS I/O scheduler
- Memory management tweaks (le9uo-style anon/clean ratios, watermark tuning)
- sched-ext (BPF-based scheduler framework)
- Various desktop responsiveness improvements gated behind `CONFIG_CACHY`

---

## Security Notes

### perf_event_open and CVE history

The `SECURITY_PERF_EVENTS_RESTRICT` option (hardened profile only) sets `kernel.perf_event_paranoid=3`, blocking all unprivileged use of `perf_event_open(2)`. This syscall has been the root cause of multiple local privilege escalation CVEs including CVE-2023-6931, CVE-2022-1729, CVE-2021-29657, and CVE-2020-14351.

On the offensive profile this restriction is intentionally disabled. Hashcat, bcc/bpftrace, and perf require hardware counter access for legitimate pentesting workflows.

### Unprivileged user namespaces

Both profiles keep `kernel.unprivileged_userns_clone=1` (the kernel default). This is required for Flatpak, Bubblewrap, rootless Podman/Docker, and browser sandboxing. On hardened deployments where these are not needed, the value can be set to `0` at runtime:

```bash
sudo sysctl kernel.unprivileged_userns_clone=0
```

---

## License

GPL-2.0-only

Patches retain their original authorship and licenses as noted in each patch file header.

---

## Credits

- [CachyOS](https://cachyos.org/) - linux-cachyos-bore base, BORE integration, ADIOS scheduler
- [Anthraxx](https://github.com/anthraxx/linux-hardened) - linux-hardened patch
- [Masahito Suzuki](https://github.com/firelzrd/bore-scheduler) - BORE scheduler
- [Kali Linux](https://www.kali.org/) - WiFi injection patches, platform keyring patch
- [Athena OS](https://athenaos.org/) - linux-aegis maintenance and profile system
