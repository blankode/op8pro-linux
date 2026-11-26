# Local kernel patches

This directory is reserved for the reviewed, original kernel patch files.
No custom charger implementation is included yet.

The development session produced:
- oneplus-charging-diagnostic.patch
- oneplus-ups-driver-diagnostic.patch

Recover the originals from the development host. The first patch was saved
under `C:\OnePlus-Linux\battery-fix\` during development. The second was
generated later; its local download location has not been confirmed.

On WSL, assuming the original files are in that Windows directory:

```sh
cd ~
git clone https://github.com/blankode/op8pro-linux.git
cd op8pro-linux
src=/mnt/c/OnePlus-Linux/battery-fix
find "$src" -maxdepth 1 -type f -name '*.patch' -print
```

Inspect the full patch contents and verify the source baseline before copying.
Only include reviewed driver/DTS changes. Do not include any unrelated files.

```sh
cp "$src/oneplus-charging-diagnostic.patch" patches/
git add -- patches/oneplus-charging-diagnostic.patch
git diff --cached
```

After review, commit and push:

```sh
git commit -m "Add original experimental OnePlus charger diagnostic patch"
git push origin main
```

Repeat for the later patch only after confirming its provenance and whether it
supersedes or applies on top of the first. Record the exact base commit,
application order and test results in [the battery notes](../docs/battery.md).
Preserve existing source licenses and copyright notices.

Do not upload private service migrations, boot/rootfs images, firmware dumps,
or a full development tree. The custom charging code remains experimental and
must not be presented as a verified charging solution.
