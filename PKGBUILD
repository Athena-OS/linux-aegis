# Maintainer: Athena OS Team <team@athenaos.org>
# Based on linux-cachyos-bore by:
#   Peter Jung ptr1337 <admin@ptr1337.dev>
#   Piotr Gorski <piotrgorski@cachyos.org>
#   Vasiliy Stelmachenok <ventureo@cachyos.org>

### ============================================================
### BUILD OPTIONS
### ============================================================

### Aegis profile
# 'offensive' — Performance-first. BORE + sched-ext, -O3, 1000Hz.
#               Full WiFi injection. No perf restrictions. Unrestricted dmesg.
# 'hardened'  — Security-first for professional engagements.
#               BORE only (no sched-ext), hardening patches, module signing,
#               perf restrictions, dmesg restrict, Yama scope 1.
: "${_aegis_profile:=offensive}"

### Enable CachyOS config sauce (CACHY, scheduler tweaks)
: "${_cachy_config:=yes}"

### Tweak kernel options interactively before build
: "${_makenconfig:=no}"
: "${_makexconfig:=no}"

# Compile only modules currently in use — vastly reduces build time.
# Requires modprobed-db: https://aur.archlinux.org/packages/modprobed-db
: "${_localmodcfg:=no}"
: "${_localmodcfg_path:="$HOME/.config/modprobed.db"}"

# Use the running kernel's .config as base.
# NOT recommended on new kernel releases.
: "${_use_current:=no}"

### Enable -O3
: "${_cc_harder:=yes}"

### Set performance governor as default cpufreq governor
: "${_per_gov:=no}"

### Enable TCP BBR congestion control
: "${_tcp_bbr3:=no}"

### Timer frequency: 1000Hz for best pentesting responsiveness
: "${_HZ_ticks:=1000}"

### Tick type: 'full' (tickless), 'idle', or 'periodic'
: "${_tickrate:=full}"

### Preempt type: 'full', 'lazy', 'voluntary', or 'none'
: "${_preempt:=full}"

### Transparent Hugepages: 'always' or 'madvise'
: "${_hugepage:=always}"

# CPU compiler optimization:
# ""        = native (default, best for personal builds)
# "generic" = portable (use for distro/repo packages)
# "zen4"    = AMD Zen4/Zen5 specific
: "${_processor_opt:=}"

# Clang LTO: "none" (GCC build), "thin", "thin-dist", or "full"
: "${_use_llvm_lto:=none}"
: "${_use_lto_suffix:=no}"
: "${_use_gcc_suffix:=no}"

# kCFI — forward-edge control-flow integrity (Clang + LTO only)
: "${_use_kcfi:=no}"

### ============================================================
### INTERNALS — do not modify below this line
### ============================================================

_is_lto_kernel() {
    [[ "$_use_llvm_lto" = "thin" || "$_use_llvm_lto" = "full" || "$_use_llvm_lto" = "thin-dist" ]]
}

_is_ci_build() {
    [[ -n "$CI" || -n "$GITHUB_RUN_ID" ]]
}

case "$_aegis_profile" in
    offensive|hardened) ;;
    *) error "Invalid _aegis_profile='$_aegis_profile'. Use 'offensive' or 'hardened'."; exit 1;;
esac

if _is_lto_kernel && [ "$_use_lto_suffix" = "yes" ]; then
    _pkgsuffix="aegis-${_aegis_profile}-lto"
elif ! _is_lto_kernel && [ "$_use_gcc_suffix" = "yes" ]; then
    _pkgsuffix="aegis-${_aegis_profile}-gcc"
else
    _pkgsuffix="aegis-${_aegis_profile}"
fi

pkgbase="linux-$_pkgsuffix"
pkgname=("$pkgbase" "$pkgbase-headers")

_major=6.19
_minor=8
_tagrel=1
pkgver=${_major}.${_minor}
pkgrel=1

_srctag=cachyos-${_major}.${_minor}-${_tagrel}
_srcname=${_srctag}

pkgdesc="Linux BORE kernel for Athena OS — ${_aegis_profile} pentesting profile"
_kernver="$pkgver-$pkgrel"
_kernuname="${pkgver}-${_pkgsuffix}"
arch=('x86_64')
url="https://github.com/Athena-OS/linux-aegis"
license=('GPL-2.0-only')
options=('!strip' '!debug' '!lto')

makedepends=(
    bc cpio gettext libelf pahole perl python
    rust rust-bindgen rust-src tar xz zstd
)

_patchsource="https://raw.githubusercontent.com/cachyos/kernel-patches/master/${_major}"

source=(
    "https://github.com/CachyOS/linux/releases/download/${_srctag}/${_srctag}.tar.gz"
    "config"
    # CachyOS BORE + Cachy sauce scheduler patch
    "${_patchsource}/sched/0001-bore-cachy.patch"
    # Kali WiFi injection patches
    "https://gitlab.com/kalilinux/packages/linux/-/raw/kali/master/debian/patches/features/all/kali-wifi-injection.patch"
    "carl9170-promisc.patch::https://gitlab.com/kalilinux/packages/linux/-/raw/kali/master/debian/patches/features/all/wireless-carl9170-Enable-sniffer-mode-promisc-flag-t.patch"
    # Intel IOMMU: exclude only iGPU, keep discrete GPU under IOMMU protection
    "intel-iommu-intgpu-off.patch::https://gitlab.com/kalilinux/packages/linux/-/raw/kali/master/debian/patches/features/x86/intel-iommu-add-option-to-exclude-integrated-gpu-only.patch"
)

if _is_lto_kernel; then
    makedepends+=(clang llvm lld)
    source+=("${_patchsource}/misc/dkms-clang.patch")
    BUILD_FLAGS=(CC=clang LD=ld.lld LLVM=1 LLVM_IAS=1)
fi

# Hardened profile: additional security patches
if [ "$_aegis_profile" = "hardened" ]; then
    source+=(
        "${_patchsource}/misc/0001-hardened.patch"
        #"hardened.patch::https://github.com/anthraxx/linux-hardened/releases/download/v${_major}.${_minor}-hardened1/linux-hardened-v${_major}.${_minor}-hardened1.patch"
        "platform-keyring-module-sig.patch::https://gitlab.com/kalilinux/packages/linux/-/raw/kali/master/debian/patches/features/all/db-mok-keyring/KEYS-Make-use-of-platform-keyring-for-module-signature.patch"
    )
fi

export KBUILD_BUILD_HOST=athenaos
export KBUILD_BUILD_USER="$pkgbase"
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

_die() { error "$@"; exit 1; }

prepare() {
    cd "$_srcname"

    echo "Setting version..."
    echo "-$pkgrel" > localversion.10-pkgrel
    echo "${pkgbase#linux}" > localversion.20-pkgname

    # Apply all patches in source array order
    local src
    for src in "${source[@]}"; do
        src="${src%%::*}"
        src="${src##*/}"
        src="${src%.zst}"
        [[ $src = *.patch ]] || continue
        echo "Applying patch $src..."
        patch -Np1 < "../$src"
    done

    echo "Setting config..."
    cp ../config .config

    ### --------------------------------------------------------
    ### CPU optimization
    ### NOTE: CachyOS config ships GENERIC_CPU=y as a safe default.
    ### We override to native for personal builds. Set
    ### _processor_opt=generic for distro/repo packages.
    ### --------------------------------------------------------
    if [ -n "$_processor_opt" ]; then
        MARCH="${_processor_opt^^}"
        case "$MARCH" in
            GENERIC_V[1-4])
                scripts/config -e GENERIC_CPU -d MZEN4 -d X86_NATIVE_CPU \
                    --set-val X86_64_VERSION "${MARCH//GENERIC_V}";;
            ZEN4)
                scripts/config -d GENERIC_CPU -e MZEN4 -d X86_NATIVE_CPU;;
            NATIVE)
                scripts/config -d GENERIC_CPU -d MZEN4 -e X86_NATIVE_CPU;;
        esac
    else
        scripts/config -d GENERIC_CPU -d MZEN4 -e X86_NATIVE_CPU
    fi

    ### --------------------------------------------------------
    ### CachyOS config sauce
    ### NOTE: CONFIG_CACHY is NOT set in the config file itself —
    ### the PKGBUILD enables it here. Same is true for SCHED_BORE.
    ### --------------------------------------------------------
    if [ "$_cachy_config" = "yes" ]; then
        echo "Enabling CachyOS config sauce..."
        scripts/config -e CACHY
    fi

    ### --------------------------------------------------------
    ### Scheduler
    ### BORE always enabled. sched-ext enabled on offensive only:
    ### CachyOS hardened patchset is incompatible with sched-ext.
    ### --------------------------------------------------------
    scripts/config -e SCHED_BORE
    echo "BORE scheduler: enabled"

    if [ "$_aegis_profile" = "offensive" ]; then
        # CONFIG_SCHED_CLASS_EXT=y is already in the CachyOS base config.
        # BORE is dormant when an scx scheduler is active, and resumes
        # automatically when it is stopped or unloaded.
        echo "sched-ext (SCX): enabled"
    else
        # The CachyOS hardened patchset conflicts with BPF-based schedulers.
        scripts/config -d SCHED_EXT
        echo "sched-ext (SCX): disabled (incompatible with hardened patchset)"
    fi

    ### --------------------------------------------------------
    ### kCFI (Clang + LTO only)
    ### --------------------------------------------------------
    if [ "$_use_kcfi" = "yes" ]; then
        scripts/config -e ARCH_SUPPORTS_CFI_CLANG -e CFI_CLANG -e CFI_AUTO_DEFAULT
        echo "kCFI: enabled"
    fi

    ### --------------------------------------------------------
    ### LTO
    ### --------------------------------------------------------
    case "$_use_llvm_lto" in
        thin)      scripts/config -e LTO_CLANG_THIN;;
        thin-dist) scripts/config -e LTO_CLANG_THIN_DIST;;
        full)      scripts/config -e LTO_CLANG_FULL;;
        none)      scripts/config -e LTO_NONE;;
        *) _die "Invalid _use_llvm_lto='$_use_llvm_lto'";;
    esac

    if ! _is_lto_kernel; then
        echo "Enabling QR Code Panic for GCC Kernels"
        scripts/config \
            --set-str DRM_PANIC_SCREEN qr_code
            # When SCHED_EXT is enabled, config is restarted and
            # it will ask for explicit prompt for QR_CODE.
            # Moved to config file
            #-e DRM_PANIC_SCREEN_QR_CODE \
            #--set-str DRM_PANIC_SCREEN_QR_CODE_URL \
            #    "https://github.com/Athena-OS/linux-aegis/issues" \
            #--set-val DRM_PANIC_SCREEN_QR_VERSION 40
    fi

    ### --------------------------------------------------------
    ### Timer frequency
    ### NOTE: CachyOS config ships HZ_300. We override to 1000Hz.
    ### --------------------------------------------------------
    case "$_HZ_ticks" in
        100|250|500|600|750|1000)
            scripts/config -d HZ_300 -e "HZ_${_HZ_ticks}" --set-val HZ "${_HZ_ticks}";;
        300)
            scripts/config -e HZ_300 --set-val HZ 300;;
        *) _die "Invalid _HZ_ticks='$_HZ_ticks'";;
    esac
    echo "Timer: ${_HZ_ticks}Hz"

    ### --------------------------------------------------------
    ### CPU frequency governor
    ### --------------------------------------------------------
    if [ "$_per_gov" = "yes" ]; then
        scripts/config -d CPU_FREQ_DEFAULT_GOV_SCHEDUTIL \
            -e CPU_FREQ_DEFAULT_GOV_PERFORMANCE
    fi

    ### --------------------------------------------------------
    ### Tick type
    ### --------------------------------------------------------
    case "$_tickrate" in
        periodic)
            scripts/config -d NO_HZ_IDLE -d NO_HZ_FULL -d NO_HZ -d NO_HZ_COMMON \
                -e HZ_PERIODIC;;
        idle)
            scripts/config -d HZ_PERIODIC -d NO_HZ_FULL \
                -e NO_HZ_IDLE -e NO_HZ -e NO_HZ_COMMON;;
        full)
            scripts/config -d HZ_PERIODIC -d NO_HZ_IDLE \
                -d CONTEXT_TRACKING_FORCE -e NO_HZ_FULL_NODEF \
                -e NO_HZ_FULL -e NO_HZ -e NO_HZ_COMMON -e CONTEXT_TRACKING;;
        *) _die "Invalid _tickrate='$_tickrate'";;
    esac
    echo "Tick type: $_tickrate"

    ### --------------------------------------------------------
    ### Preempt type
    ### --------------------------------------------------------
    case "$_preempt" in
        full)
            scripts/config -e PREEMPT_DYNAMIC -e PREEMPT \
                -d PREEMPT_VOLUNTARY -d PREEMPT_LAZY -d PREEMPT_NONE;;
        lazy)
            scripts/config -e PREEMPT_DYNAMIC -d PREEMPT \
                -d PREEMPT_VOLUNTARY -e PREEMPT_LAZY -d PREEMPT_NONE;;
        voluntary)
            scripts/config -d PREEMPT_DYNAMIC -d PREEMPT \
                -e PREEMPT_VOLUNTARY -d PREEMPT_LAZY -d PREEMPT_NONE;;
        none)
            scripts/config -d PREEMPT_DYNAMIC -d PREEMPT \
                -d PREEMPT_VOLUNTARY -d PREEMPT_LAZY -e PREEMPT_NONE;;
        *) _die "Invalid _preempt='$_preempt'";;
    esac
    echo "Preempt: $_preempt"

    ### --------------------------------------------------------
    ### Compiler optimization
    ### --------------------------------------------------------
    if [ "$_cc_harder" = "yes" ]; then
        scripts/config -d CC_OPTIMIZE_FOR_PERFORMANCE \
            -e CC_OPTIMIZE_FOR_PERFORMANCE_O3
        echo "Optimization: -O3"
    fi

    if _is_ci_build; then
        scripts/config \
            -d CC_OPTIMIZE_FOR_PERFORMANCE_O3 \
            -e CC_OPTIMIZE_FOR_SIZE \
            -d DEBUG_KERNEL \
            -e DEBUG_INFO_REDUCED
    fi

    ### --------------------------------------------------------
    ### TCP congestion control
    ### --------------------------------------------------------
    if [ "$_tcp_bbr3" = "yes" ]; then
        scripts/config \
            -m TCP_CONG_CUBIC -d DEFAULT_CUBIC \
            -e TCP_CONG_BBR -e DEFAULT_BBR \
            --set-str DEFAULT_TCP_CONG bbr \
            -m NET_SCH_FQ_CODEL -e NET_SCH_FQ \
            -d DEFAULT_FQ_CODEL -e DEFAULT_FQ
    fi

    ### --------------------------------------------------------
    ### Transparent Hugepages
    ### --------------------------------------------------------
    case "$_hugepage" in
        always)  scripts/config -d TRANSPARENT_HUGEPAGE_MADVISE \
                     -e TRANSPARENT_HUGEPAGE_ALWAYS;;
        madvise) scripts/config -d TRANSPARENT_HUGEPAGE_ALWAYS \
                     -e TRANSPARENT_HUGEPAGE_MADVISE;;
        *) _die "Invalid _hugepage='$_hugepage'";;
    esac

    ### --------------------------------------------------------
    ### Pentesting networking
    ### Set explicitly — CachyOS config cannot be relied upon to
    ### have these (AppArmor was silently dropped in recent builds).
    ### --------------------------------------------------------
    scripts/config \
        -e PACKET \
        -m TUN \
        -m MAC80211_HWSIM \
        -e CFG80211_CRDA_SUPPORT \
        -e TCP_AO

    ### --------------------------------------------------------
    ### User namespaces
    ### unprivileged-userns-sysctl.patch adds the sysctl knob
    ### kernel.unprivileged_userns_clone (default=1).
    ### Offensive: stays 1. Hardened: athena-settings sets it to 0.
    ### --------------------------------------------------------
    scripts/config -e USER_NS

    ### --------------------------------------------------------
    ### Security — common to both profiles
    ### Set explicitly: CachyOS recently removed AppArmor (#574).
    ### But still used in Athena OS.
    ### --------------------------------------------------------
    scripts/config \
        -e SECURITY_LOCKDOWN_LSM \
        -e LOCK_DOWN_IN_EFI_SECURE_BOOT \
        -d LOCK_DOWN_KERNEL_FORCE_NONE \
        -e SECURITY_APPARMOR \
        -e SECURITY_APPARMOR_HASH \
        -e SECURITY_APPARMOR_HASH_DEFAULT \
        -e STACKPROTECTOR \
        -e STACKPROTECTOR_STRONG \
        -e STRICT_DEVMEM \
        -e STRICT_KERNEL_RWX \
        -e STRICT_MODULE_RWX \
        -e FORTIFY_SOURCE \
        -e HARDENED_USERCOPY \
        -e BPF_UNPRIV_DEFAULT_OFF \
        --set-str LSM "landlock,lockdown,yama,integrity,apparmor,bpf"

    ### --------------------------------------------------------
    ### Profile-specific security
    ### --------------------------------------------------------
    if [ "$_aegis_profile" = "hardened" ]; then
        echo "Applying hardened profile security config..."

        #scripts/config \
        #    -e SECURITY_PERF_EVENTS_RESTRICT \
        #    -e SECURITY_YAMA \
        #    -e SECURITY_DMESG_RESTRICT \
        #    -e MODVERSIONS \
        #    -e MODULE_SIG \
        #    -e MODULE_SIG_ALL \
        #    -e MODULE_SIG_SHA512 \
        #    -e INTEGRITY \
        #    -e INTEGRITY_SIGNATURE \
        #    -e INTEGRITY_ASYMMETRIC_KEYS \
        #    -e INTEGRITY_PLATFORM_KEYRING \
        #    -e IMA \
        #    -e IMA_APPRAISE \
        #    -e IMA_ARCH_POLICY \
        #    -e IMA_KEYRINGS_PERMIT_SIGNED_BY_BUILTIN_OR_SECONDARY

    else
        echo "Applying offensive profile security config..."

        scripts/config \
            -d SECURITY_PERF_EVENTS_RESTRICT \
            -e SECURITY_YAMA \
            -d SECURITY_DMESG_RESTRICT \
            -d MODVERSIONS
    fi

    ### --------------------------------------------------------
    ### Optional: use running kernel's config as base
    ### --------------------------------------------------------
    if [ "$_use_current" = "yes" ]; then
        if [[ -s /proc/config.gz ]]; then
            echo "Extracting config from /proc/config.gz..."
            zcat /proc/config.gz > ./.config
        else
            warning "Kernel not compiled with IKPROC. Aborting."
            exit 1
        fi
    fi

    ### --------------------------------------------------------
    ### Optional: localmodconfig
    ### --------------------------------------------------------
    if [ "$_localmodcfg" = "yes" ]; then
        if [ -e "$_localmodcfg_path" ]; then
            echo "Running localmodconfig..."
            make "${BUILD_FLAGS[@]}" LSMOD="${_localmodcfg_path}" localmodconfig
        else
            _die "modprobed.db not found at $_localmodcfg_path"
        fi
    fi

    ### --------------------------------------------------------
    ### Finalize
    ### --------------------------------------------------------
    echo "Finalizing config..."
    make "${BUILD_FLAGS[@]}" prepare
    yes "" | make "${BUILD_FLAGS[@]}" config >/dev/null
    diff -u ../config .config || :

    make -s kernelrelease > version
    echo "Prepared $pkgbase version $(<version)"

    [ "$_makenconfig" = "yes" ] && make "${BUILD_FLAGS[@]}" nconfig
    [ "$_makexconfig" = "yes" ] && make "${BUILD_FLAGS[@]}" xconfig

    # Save config for later reuse
    local basedir
    basedir="$(dirname "$(readlink "${srcdir}/config")")"
    cat .config > "${basedir}/config-${pkgver}-${pkgrel}${pkgbase#linux}"
}

build() {
    cd "$_srcname"
    make "${BUILD_FLAGS[@]}" -j"$(nproc)" all

    if ! _is_ci_build; then
        make -C tools/bpf/bpftool vmlinux.h feature-clang-bpf-co-re=1
    fi
}

_package() {
    pkgdesc="The $pkgdesc kernel and modules"
    depends=('coreutils' 'kmod' 'initramfs')
    optdepends=(
        'wireless-regdb: set correct wireless channels for your country'
        'linux-firmware: firmware images needed for some devices'
        'modprobed-db: track used modules for faster localmodconfig builds'
        'scx-scheds: sched-ext userspace schedulers (offensive profile only)'
    )
    provides=(
        VIRTUALBOX-GUEST-MODULES WIREGUARD-MODULE KSMBD-MODULE
        V4L2LOOPBACK-MODULE NTSYNC-MODULE VHBA-MODULE ADIOS-MODULE
    )

    cd "$_srcname"
    local modulesdir="$pkgdir/usr/lib/modules/$(<version)"

    echo "Installing boot image..."
    install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"
    echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

    echo "Installing modules..."
    ZSTD_CLEVEL=19 make "${BUILD_FLAGS[@]}" \
        INSTALL_MOD_PATH="$pkgdir/usr" \
        INSTALL_MOD_STRIP=1 \
        DEPMOD=/doesnt/exist \
        modules_install

    rm "$modulesdir/build"
}

_package-headers() {
    pkgdesc="Headers and scripts for building modules for the $pkgdesc kernel"
    depends=('pahole' "${pkgbase}")
    provides=(LINUX-HEADERS)

    cd "${_srcname}"
    local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

    echo "Installing build files..."
    install -Dt "$builddir" -m644 \
        .config Makefile Module.symvers System.map \
        localversion.* version vmlinux

    if ! _is_ci_build; then
        install -Dt "$builddir" -m644 tools/bpf/bpftool/vmlinux.h
    fi

    install -Dt "$builddir/kernel" -m644 kernel/Makefile
    install -Dt "$builddir/arch/x86" -m644 arch/x86/Makefile
    cp -t "$builddir" -a scripts
    ln -srt "$builddir" "$builddir/scripts/gdb/vmlinux-gdb.py"
    install -Dt "$builddir/tools/objtool" tools/objtool/objtool

    if [ -f tools/bpf/resolve_btfids/resolve_btfids ]; then
        install -Dt "$builddir/tools/bpf/resolve_btfids" \
            tools/bpf/resolve_btfids/resolve_btfids
    fi

    echo "Installing headers..."
    cp -t "$builddir" -a include
    cp -t "$builddir/arch/x86" -a arch/x86/include
    install -Dt "$builddir/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s

    install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
    install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h
    install -Dt "$builddir/drivers/media/i2c" \
        -m644 drivers/media/i2c/msp3400-driver.h
    install -Dt "$builddir/drivers/media/usb/dvb-usb" \
        -m644 drivers/media/usb/dvb-usb/*.h
    install -Dt "$builddir/drivers/media/dvb-frontends" \
        -m644 drivers/media/dvb-frontends/*.h
    install -Dt "$builddir/drivers/media/tuners" \
        -m644 drivers/media/tuners/*.h
    install -Dt "$builddir/drivers/iio/common/hid-sensors" \
        -m644 drivers/iio/common/hid-sensors/*.h

    echo "Installing KConfig files..."
    find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

    compgen -G "rust/*.rmeta" 1>/dev/null && \
        install -Dt "$builddir/rust" -m644 rust/*.rmeta
    compgen -G "rust/*.so" 1>/dev/null && \
        install -Dt "$builddir/rust" rust/*.so

    echo "Installing unstripped VDSO..."
    make INSTALL_MOD_PATH="$pkgdir/usr" vdso_install link=

    echo "Removing unneeded architectures..."
    local arch
    for arch in "$builddir"/arch/*/; do
        [[ $arch = */x86/ ]] && continue
        rm -r "$arch"
    done

    rm -r "$builddir/Documentation"
    find -L "$builddir" -type l -printf 'Removing %P\n' -delete
    find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

    echo "Stripping build tools..."
    local file
    while read -rd '' file; do
        case "$(file -Sib "$file")" in
            application/x-sharedlib\;*)      strip -v $STRIP_SHARED "$file";;
            application/x-archive\;*)        strip -v $STRIP_STATIC "$file";;
            application/x-executable\;*)     strip -v $STRIP_BINARIES "$file";;
            application/x-pie-executable\;*) strip -v $STRIP_SHARED "$file";;
        esac
    done < <(find "$builddir" -type f -perm -u+x ! -name vmlinux -print0)
    strip -v $STRIP_STATIC "$builddir/vmlinux"

    mkdir -p "$pkgdir/usr/src"
    ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

for _p in "${pkgname[@]}"; do
    eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
    }"
done

b2sums=('c2aa84e0ab473d1b9a92fb42d90eac61f6ab600c6cd3b052978e349dfd66f5140c0ff8d04eadfda4b953d2999a057aa5deb4b4610e0f439b87a62e2edad28ce3'
        '235a4fa38926d575633dea787c2ceca18c251908bad77f15aae1897cf446bba35f87d090ead292221432298e6e55f8852e1bdfb74425570b7e56a0ad1f312b7c'
        '452d57751b27c3fee1c402dd4c6102cbbfd85fb054ce52a6e8fc8d1c7c2cc6f2292213dc9c93561552b4755933c04857ba6d8057ad10d2942f7c422dbeddde7a'
        'ba9d775e5c6b504083a1e97ad143facda064840e88df6328c9680ca4100e6c354ac821c320ee01b9896fb9f5e53cc808d4b251f6883d02dd58b4e151160f2730'
        'b227bde0ba8729fa9e50ecaf344d96f1521df120b74188bdef445eaa550bdbbe6313183f9ad0182f12c6079d17d055b38b23d0479f937360fdbd22df794ddbae'
        'c83386d9456fc27fb360f3b9c2775d36f91678d12c903fe91575927965a32a2202ab001379fdab1710b8152c506496bbdc0e2c5396530f27706cb964dae34c20')
