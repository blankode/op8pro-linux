# Battery and charging development

## Current status — 2026-11-26

The latest integrated diagnostic kernel, `7.2.0-oneplus-server-ups-diag`, compiled and booted temporarily on the IN2023. The build produced 614 modules and an approximately 168 MB module archive. USB input detection, battery telemetry and the auxiliary thermal/voltage sensor reporting passed the initial checks. Physical battery charging and automatic UPS failover have **not** been validated.

### Latest observed measurements

| Measurement | Observed |
| --- | --- |
| Kernel | 7.2.0-oneplus-server-ups-diag |
| USB input | Present, valid |
| Battery voltage | 3,784,000 µV (3.784 V) |
| Battery current | 54,000 µA (54 mA reported; sign convention requires verification) |
| Battery pack temperature | 24.5 °C; subsequent observations 24.5–24.6 °C |
| BQ27541 internal temperature | 30.6 °C |
| Battery health | Good (driver report) |
| Auxiliary sample age | 2 s |
| Auxiliary valid mask | 0x1f |
| Auxiliary fault mask | 0x00 |
| PM8150 skin sensor | 34.558 °C |
| PM8150B skin sensor | 33.333 °C |
| MSM skin sensor | 36.406 °C |
| USB connector sensor | 25.275 °C |
| VBAT sense ADC | 3,793,077 µV |
| Charge requested | 1 |
| Requested charging current | 1,000,000 µA |
| Charge authorized | 0 |
| Charging enabled | 0 |

The voltage readings differ by approximately 9.1 mV. These are observations from a single diagnostic run, not evidence of sustained charging or safe unattended operation. The capacity reading was not sufficiently clear in the captured output and is omitted.

### Target UPS behavior (not yet validated)

- USB power is primary; the battery acts as backup when input power disappears.
- Request charging below 90% and stop at 95%, with a maximum requested current of 1 A.
- Target a controlled shutdown at 10% battery when external power is unavailable.
- Keep hardware protection, independent temperature checks and fail-closed authorization; validate actual charging and power transitions before deployment.

The older 500 mA / 70–80% experimental targets are superseded as policy goals. A requested current is not proof of physical current delivered to the pack. No charging authorization or enable was observed in the latest run.

Do not run the experimental charging kernel unattended. Stop testing on unexpected voltage, temperature, swelling or unstable behavior. Do not bypass battery protection, force charger register writes or disable thermal protection.

## Source baseline

- Upstream port: [darknagel/oneplus-instantnoodlep-nura-unofficial v2026.11.24](https://github.com/darknagel/oneplus-instantnoodlep-nura-unofficial/tree/v2026.11.24).
- Linux source: [Nura Qualcomm SM8250 kernel](https://gitlab.postmarketos.org/soc/qualcomm-sm8250/linux), tag `sm8250-7.2.0_git20261010`.
- Apply the port's complete kernel patch series before any local changes.
- See upstream [charging notes](https://github.com/darknagel/oneplus-instantnoodlep-nura-unofficial/blob/v2026.11.24/docs/charging.md) and [patch documentation](https://github.com/darknagel/oneplus-instantnoodlep-nura-unofficial/blob/v2026.11.24/docs/patches.md).

The upstream port has its own charging implementation. Our work is an experiment
on top of that port, not a replacement for its full driver stack.

## Local modifications recorded during development

| Component | Purpose |
| --- | --- |
| `drivers/power/supply/qcom_pm8150b_charger.c` | Diagnostic UPS policy, charger state and observability |
| BQ27541 fuel-gauge driver | Separate pack and internal temperature reporting; internal temperature register 0x28 |
| `arch/arm64/boot/dts/qcom/sm8250-oneplus-instantnoodlep.dts` | Fuel-gauge compatible changed from `ti,bq27411` to `ti,bq27541`; charger-to-gauge reference retained |
| ADC/device-tree configuration | Auxiliary thermal channels and battery-voltage sensing; exact final diff requires source verification |

### Recorded diagnostic patch progression

| Patch | Development status / purpose |
| --- | --- |
| `oneplus-charging-diagnostic.patch` | Early charger diagnostics; historical |
| `oneplus-ups-driver-diagnostic.patch` | UPS diagnostic iteration |
| `oneplus-ups-stock-batif.patch` | Stock BATIF status and battery terminal/thermistor-or-ID missing indicators; compiled |
| `oneplus-ups-charge-timeout.patch` | 10-hour session timeout using monotonic boot time; compiled |
| `oneplus-aux-thermal-guard.patch` | Five auxiliary channels: three skin sensors, USB connector and VBAT ADC; object and DTB compiled |
| `oneplus-bq-internal-guard.patch` | Internal gauge temperature guard; object compiled |

The recorded guard design includes periodic 5-second sampling, a battery-voltage consistency check with a 150 mV maximum difference, and separate pack/internal temperature observations. These are design and development notes, not certified limits or independently audited code. The integrated diagnostic build reported five valid auxiliary channels and no auxiliary faults.

The next phase is controlled implementation and validation of charging authorization. No later charging-enabled kernel test is available in this record.

**Source availability:** The original patch files and the exact final source diff have not been recovered into this repository. See [patch inventory](../patches/README.md). Do not infer application order from the table or recreate safety-critical register programming from these notes.

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
against the correct baseline with `git apply --check`. Do not blindly stack the diagnostic versions or later guards: patches may overlap or supersede earlier changes. The source
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

## Latest read-only UPS diagnostics

The development build exposed additional attributes. Enumerate the actual power-supply directories rather than assuming a charger device name:

```sh
uname -r
for p in /sys/class/power_supply/*; do
    for f in capacity voltage_now current_now temp temp_internal health \
        batif_status batif_terminal_missing batif_therm_or_id_missing \
        charge_session_timeout aux_sensor_report ups_charge_requested \
        ups_charge_authorized charging_enabled; do
        [ -r "$p/$f" ] || continue
        printf '%s: ' "$p/$f"
        cat "$p/$f"
    done
done
```

Attribute availability depends on the exact patch revision. This is a read-only check, not a command to activate charging.

## Next steps

1. Recover the exact integrated driver, DTS and configuration changes from the development machine.
2. Publish a reviewed patch series with exact base revision and preserved license notices.
3. Reproduce the latest kernel build, matching modules and boot image.
4. Validate charging authorization, actual battery current, voltage, temperature, timeout, hysteresis, power-loss behavior and low-battery shutdown in controlled tests.
5. Only claim charging or unattended UPS operation after repeated safe hardware tests.

No private services, backups, device identifiers, credentials or service
configuration are required for this project.
