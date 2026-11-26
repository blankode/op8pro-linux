# OnePlus 8 Pro — native Linux

A device-specific migration guide and battery/charger development log for the OnePlus 8 Pro running native Nura Linux (formerly postmarketOS), without Android or Termux.

> **Development status (2026-11-26):** Native Linux boots and USB/Wi-Fi SSH work on the tested IN2023. The custom power diagnostic kernel boots and exposes battery measurements, but it does **not** yet charge the battery reliably; charging is deliberately inhibited. This is not a finished UPS or charging fix. Do not use the experimental kernel unattended or leave the device connected indefinitely.

## Tested hardware and software

| Component | Tested configuration |
| --- | --- |
| Phone | OnePlus 8 Pro IN2023 (EU), instantnoodlep |
| SoC | Qualcomm Snapdragon 865 / SM8250 (kona) |
| Memory/storage | 12 GB / 256 GB |
| Starting firmware | Android 13 |
| Linux | Nura console image, release v2026.11.24 |
| Base kernel | sm8250-7.2.0_git20261010 |
| Experimental kernel | 7.2.0-oneplus-server-power-diag |

This is an independent extension of [darknagel's unofficial OnePlus 8 Pro Nura port](https://github.com/darknagel/oneplus-instantnoodlep-nura-unofficial). All credit for the native Linux port and base kernel patches belongs to their respective upstream authors.

## 1. Preparation

**WARNING: bootloader unlocking and installation erase data.** The installer overwrites the Android super partition. This is not dual boot. Back up personal files and obtain the correct stock firmware/recovery instructions before continuing. Preserve device-specific firmware and calibration partitions; never publish them. Do not use these images on a different model or blindly relock the bootloader with Linux installed.

- Use current [Android SDK Platform-Tools](https://developer.android.com/tools/releases/platform-tools) and a reliable USB data cable.
- Enable OEM unlocking and USB debugging in Android developer options.
- Check the connection with `adb devices` and reboot into fastboot with `adb reboot bootloader`.
- Check `fastboot devices`, then unlock with `fastboot flashing unlock` if the bootloader is still locked. Confirm the wipe on the phone and return to fastboot.
- Verify the device/slot with `fastboot getvar product` and `fastboot getvar current-slot`. Do not proceed if the hardware does not match.

## 2. Download the Linux image

Use the pinned [v2026.11.24 release](https://github.com/darknagel/oneplus-instantnoodlep-nura-unofficial/releases/tag/v2026.11.24).

Download [oneplus-instantnoodlep-install.zip](https://github.com/darknagel/oneplus-instantnoodlep-nura-unofficial/releases/download/v2026.11.24/oneplus-instantnoodlep-install.zip).

Archive SHA-256:

```text
0b3b15a954301a8a260b5676766ae26326079e5083b26d9d45e2054913aa429d
```

On Linux: `sha256sum oneplus-instantnoodlep-install.zip`. On Windows PowerShell: `Get-FileHash .\oneplus-instantnoodlep-install.zip -Algorithm SHA256`. Compare with the value above before unpacking. Verify the extracted images with the archive's SHA256SUMS (`sha256sum -c SHA256SUMS`).

The archive contains boot.img, dtbo.img, oneplus-instantnoodlep-rootfs.img, and checksums. Do not substitute older images: the upstream maintainer warns that releases through v2026.11.10 had an unsafe battery overcharge bug.

## 3. Install native Linux

In a terminal opened in the extracted directory, with the phone in fastboot mode:

```sh
fastboot devices
fastboot set_active a
fastboot getvar current-slot
fastboot flash dtbo_a dtbo.img
fastboot flash boot_a boot.img
fastboot flash super oneplus-instantnoodlep-rootfs.img
fastboot reboot
```

Run commands individually, checking each result before continuing. The active slot must be a. If set_active is unsupported, verify the current slot instead; stop if it is not a. If a flash actually fails, do not continue or erase other partitions. Keep the original verified boot image available for recovery.

On first boot the root filesystem expands and the port imports firmware from the device. Do not interrupt the initialization.

## 4. First boot and SSH

Connect the phone to the computer over USB. The image exposes a USB network adapter.

```sh
ssh root@172.16.42.1
passwd
nmtui
```

The upstream image's initial password is documented in its [release notes](https://github.com/darknagel/oneplus-instantnoodlep-nura-unofficial/releases/tag/v2026.11.24). Change it immediately before enabling Wi-Fi or exposing the device to other networks. Configure your own network in nmtui and discover the address using `ip -br addr`. Use SSH keys and restrict SSH to trusted networks.

If Windows does not recognize USB networking, select Microsoft UsbNcm Host Device in Device Manager, following the upstream troubleshooting guide.

Verify the native system:

```sh
uname -a
cat /etc/os-release
lsblk
ip -br addr
systemctl --failed
ls /sys/class/power_supply
```

The image uses a console and systemd. No desktop is required. Firmware provisioning may need an Internet connection and a subsequent reboot. See [upstream installation and troubleshooting](https://github.com/darknagel/oneplus-instantnoodlep-nura-unofficial/tree/v2026.11.24).

Avoid a blind apk upgrade or mixing unrelated kernel images/modules; the port carries device-specific patches.

## 5. Battery driver and kernel

The upstream release reports ordinary charging support on its tested devices. Our experimental changes target a separate power-management problem on the tested phone.

See [battery driver development and kernel build notes](docs/battery.md) for the observed behavior, source baseline, local patch names, build commands, test and rollback guidance.

The exact custom driver patch is not included in this initial documentation commit: the original local source has not been recovered into this repository. Do not treat a description of a patch as its implementation.

## Scope

Only native Linux installation, device-specific kernel changes, and generic diagnostics belong here. No personal application data, service deployments, private configuration, credentials, device backups, firmware/calibration dumps or SSH keys.

Kernel changes retain the licenses and copyright notices of their source files. The original port and its packages are subject to their respective licenses. This project is experimental and comes with no warranty.
