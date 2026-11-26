# Battery and charging development

## Status — 2026-11-26

Native Nura Linux was successfully installed on an IN2023. The experimental kernel
`7.2.0-oneplus-server-power-diag` booted. Battery voltage, state of charge, current and
temperature were readable. Battery charging was **not** demonstrated: charging remained
inhibited, including with a USB power source connected.

| Test | Observed |
| --- | --- |
| On battery | 42%, 3.806 V, -458 mA, battery temperature 24.5 °C |
| USB disconnected | Power path status 2; USB present 0; input valid 0 |
| USB connected | Power path status 149 (0x95); USB present 1; input valid 1 |
| Charger state | 7 |
| Charging enabled | 0 |
| Reported faults | 0 |
| After over six hours connected | 35% battery, current reported as 0 µA, charging enabled 0 |

The raw state values are recorded for comparison, not decoded as proof that any
particular hardware path is safe or active. A valid USB input is not proof of battery
charging. A zero current reading is not proof of a fully functional power-path/UPS mode.

Do not leave an experimental charging kernel running unattended. Stop testing on
unexpected voltage, temperature, swelling or unstable behavior. Do not bypass the
battery protection hardware, force charger register writes or disable thermal protection.

## Source baseline

- Upstream port: [darknagel/oneplus-instantnoodlep-nura-unofficial v2026.11.24](https://github.com/darknagel/oneplus-instantnoodlep-nura-unofficial/tree/v2026.11.24).
- Linux source: [Nura Qualcomm SM8250 kernel](https://gitlab.postmarketos.org/soc/qualcomm-sm8250/linux), tag `sm8250-7.2.0_git20261010`.
- Apply the port's complete kernel patch series before any local changes.
- See upstream [charging notes](https://github.com/darknagel/oneplus-instantnoodlep-nura-unofficial/blob/v2026.11.24/docs/charging.md) and [patch documentation](https://github.com/darknagel/oneplus-instantnoodlep-nura-unofficial/blob/v2026.11.24/docs/patches.md).

The upstream port has its own charging implementation. Our work is an experiment
on top of that port, not a replacement for its full driver stack.

## Local modifications recorded during development

| File | Purpose |
| --- | --- |
| `drivers/power/supply/qcom_pm8150b_charger.c` | Diagnostic charger policy and observability |
| `arch/arm64/boot/dts/qcom/sm8250-oneplus-instantnoodlep.dts` | Fuel-gauge compatible changed from `ti,bq27411` to `ti,bq27541`; existing charger-to-gauge reference retained |
| ADC device-tree configuration | Investigated battery voltage measurement through `channel@84` / `vbat_sns`; exact final diff needs source verification |

The diagnostic patch was designed with a requested 4.10 V float target, 500 mA
current limit, 80% stop / 70% resume thresholds and 10–40 °C charging window.
The recorded policy inhibited charging at boot, with invalid sensors or programming
errors, retained the watchdog, and removed a forced restart path. These are
design intentions, not certified battery limits or verified register values.

Original artifacts recorded in the development session:

- `oneplus-charging-diagnostic.patch` (approximately 12.5 KiB; charger driver)
- `oneplus-ups-driver-diagnostic.patch` (later diagnostic iteration; test status unconfirmed)

**Source availability:** The exact artifacts are not included yet. They must be
recovered from the original development machine and reviewed before publication.
The DTS change alone does not reproduce the diagnostic kernel. No placeholder or
guessed driver implementation is provided.

## Reconstruct the baseline

On a Linux or WSL2 build host:

```sh
mkdir -p ~/oneplus-build
cd ~/oneplus-build
git clone --branch v2026.11.24 https://github.com/darknagel/oneplus-instantnoodlep-nura-unofficial.git instantnoodlep-patches
git clone --branch sm8250-7.2.0_git20261010 https://gitlab.postmarketos.org/soc/qualcomm-sm8250/linux.git linux-sm8250
cd linux-sm8250
git switch -c op8pro-diagnostics
git am ~/oneplus-build/instantnoodlep-patches/kernel-patches/*.patch
git rev-parse HEAD
```

If using an already patched source tree, do not apply the upstream patches again.
Record its exact source revision and preserve it as a baseline for a clean diff.

After the original local patches have been recovered, inspect them and check each
against the correct baseline with `git apply --check`. Do not blindly stack the two
diagnostic versions: they may overlap or be successive revisions. The source
history is needed to determine the exact sequence. Review all DTS changes too.

## Build notes

Capture the running configuration from the phone (replace the address if necessary):

```sh
ssh root@172.16.42.1 'cat /proc/config.gz' > original-kernel-config.gz
gzip -t original-kernel-config.gz
```

The original build used an Alpine build environment with LLVM 23. The following
shows the build commands used, with host-specific paths adapted to a source
checkout. Install the matching Clang/LLVM and kernel build dependencies first;
this is not a self-contained toolchain installer.

```sh
export PATH=/usr/lib/llvm23/bin:$PATH
export OUT="$HOME/oneplus-build/out"
mkdir -p "$OUT"
gzip -dc original-kernel-config.gz > "$OUT/.config"
make O="$OUT" ARCH=arm64 LLVM=1 olddefconfig
make -j"$(nproc)" O="$OUT" ARCH=arm64 LLVM=1 LOCALVERSION= Image Image.gz modules qcom/sm8250-oneplus-instantnoodlep.dtb
make -s O="$OUT" ARCH=arm64 LLVM=1 LOCALVERSION= kernelrelease
make O="$OUT" ARCH=arm64 LLVM=1 LOCALVERSION= INSTALL_MOD_PATH="$OUT/staging" INSTALL_MOD_STRIP=1 modules_install
```

Run these in the source tree, using the captured config at the indicated path.
The kernel release string must exactly match the installed module directory.

The original experiment repacked the Android boot image using header version 2
and 4096-byte pages, preserving the original command line, load addresses,
ramdisk and required boot image metadata while replacing the kernel/DTB as
appropriate. The exact repack tool and command line have not been recovered.
Do not create a boot image from these notes alone. Prefer the upstream
pmbootstrap packaging method and preserve the exact compatible kernel and
module package; consult [upstream build instructions](https://github.com/darknagel/oneplus-instantnoodlep-nura-unofficial/tree/v2026.11.24#build-it-yourself).

## Test and rollback

Only after creating and verifying a compatible boot image and arranging matching
modules, boot it temporarily where the bootloader supports it:

```sh
fastboot boot boot-oneplus-server-test.img
```

The original device supported temporary fastboot boot. Do not permanently flash
a new image before verifying boot, SSH, power telemetry and safety behavior.
Use the matching original release boot image and modules if reverting. If only
the boot partition was modified and the original modules are still installed,
the original boot image can be restored in fastboot:

```sh
fastboot getvar current-slot
fastboot flash boot_a boot.img
fastboot reboot
```

This assumes slot a and the verified original boot.img from the installed release.
Do not reflash super merely to roll back a kernel; it destroys the installed
root filesystem.

## Read-only diagnostic commands

```sh
uname -r
for p in /sys/class/power_supply/*; do
    [ -r "$p/uevent" ] || continue
    printf '\n--- %s ---\n' "$p"
    cat "$p/uevent"
done
dmesg | grep -Ei 'pm8150b|bq27|fuel.gauge|charg|vbat'
```

The standard power-supply sysfs attributes normally report voltage in µV,
current in µA and battery temperature in tenths of a degree Celsius.
Verify driver-specific sign conventions and custom attributes against actual
source before interpreting them.

## Next steps

1. Recover the original driver and DTS changes from the development machine.
2. Publish a reviewed patch series with exact base revision and preserved license notices.
3. Reproduce the kernel build and boot image using documented commands.
4. Validate input detection, fuel-gauge measurements and charging policy under
   controlled conditions; verify actual battery current and voltage over time.
5. Only claim charging or unattended operation after repeated safe hardware tests.

No private services, backups, device identifiers, credentials or service
configuration are required for this project.
