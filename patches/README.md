# Local kernel patch inventory

Latest tested kernel: `7.2.0-oneplus-server-ups-diag` (2026-11-26).

The following patch names are recorded in the development history. **The actual
patch contents are not yet present in this repository.** The list is not a
verified application sequence.

| Artifact | Role |
| --- | --- |
| oneplus-charging-diagnostic.patch | Earlier charger diagnostic prototype |
| oneplus-ups-driver-diagnostic.patch | UPS diagnostic iteration |
| oneplus-ups-stock-batif.patch | Stock battery-interface status |
| oneplus-ups-charge-timeout.patch | Charge-session timeout |
| oneplus-aux-thermal-guard.patch | Auxiliary temperature/voltage sensor guard |
| oneplus-bq-internal-guard.patch | Separate BQ27541 internal-temperature guard |

The last four changes were compiled during development; the integrated kernel
booted with charging authorization and enable still disabled. See
[latest validation results](../docs/battery.md).

## Recover the current source

Prefer a reviewed diff of the latest integrated source against the exact clean
upstream-patched baseline. That avoids accidentally stacking superseded
diagnostic prototypes. Record the baseline SHA, changed source files, kernel
configuration, and reproducible build procedure. Preserve source licenses.

On the original WSL development host, search the known build/download locations:

```sh
find ~/oneplus-build /mnt/c/OnePlus-Linux/battery-fix \
  -type f -name 'oneplus-*.patch' -print 2>/dev/null
```

For a Git source tree, inspect the current changes:

```sh
git status --short
git diff --stat
git diff
git log -5 --oneline
```

These commands do not capture already committed changes by themselves; compare
the current revision and working tree with the verified upstream-patched base.
Review and export only relevant kernel driver, DTS and configuration changes.
Check the exported patch on a clean copy of that baseline before publishing.

Clone this repository, copy only the reviewed patch files into `patches/`,
stage those files by name, inspect the staged diff and commit/push. Do not
stage a whole phone backup or development directory.

The charging-enabled implementation remains unverified. Do not publish
guessed driver code or distribute the diagnostic build as a working UPS fix.
No personal services, credentials, device firmware, backups or application
data belong here.
