# pocketlab

Fedora Rawhide arm64 image experiments for booting Fedora's stock kernel and userspace on the OnePlus 6T (`oneplus-fajita`) through pocketboot.

The image is intentionally boring:

- GPT disk image.
- No ESP.
- XBOOTLDR mounted at `/boot`.
- ext4 root filesystem labeled `fedora`.
- Fedora `kernel-install` writes BLS Type #1 entries.
- Fedora `dracut` generates the initramfs.
- pocketboot reads BLS from XBOOTLDR and kexecs Fedora.

## Build

Use mkosi v26 from the local checkout:

```sh
~/src/mkosi/bin/mkosi summary
~/src/mkosi/bin/mkosi build
```

The main output is `mkosi.output/pocketlab.raw`.

## Boot Contract

pocketboot is required. It must be installed separately on the device and must be able to scan the disk image's XBOOTLDR partition.

The generated XBOOTLDR partition contains:

- `/loader/entries/*.conf`
- per-kernel `linux`
- per-kernel `initrd`
- per-kernel `sdm845-oneplus-fajita.dtb` copied by Fedora's `90-loaderentry.install`

## Fedora Alignment

This repo avoids mkosi's initrd and UKI paths. `/boot` is produced by Fedora mechanisms:

- `/etc/kernel/install.conf` selects `layout=bls` and `initrd_generator=dracut`.
- `/etc/kernel/cmdline` supplies the kernel command line consumed by Fedora's BLS generator.
- `/etc/kernel/devicetree` lets Fedora's `90-loaderentry.install` copy and reference the fajita DTB.
- `mkosi.finalize.chroot` reruns `kernel-install add` after generating a filtered SDM845 dracut config.

The Phosh package set follows Fedora Kiwi's Phosh intent by installing the `phosh-desktop-environment` environment plus `initial-setup-gui-wayland-generic`. `fedora-release-mobility` is intentionally excluded for this bring-up image.

`dracut-config-rescue` is also excluded because Fedora's rescue initrd path breaks this non-ESP/XBOOTLDR-only image and is not useful for this device workflow.

## SDM845 Dracut Drivers

The explicit candidate inventory lives in:

```text
mkosi.skeleton/usr/lib/pocketlab/dracut-force-drivers.list
```

During finalization, the list is filtered through the installed Rawhide kernel and written to:

```text
/etc/dracut.conf.d/10-sdm845-fajita-force-drivers.conf
```

Candidates missing from the installed kernel are recorded in:

```text
/var/lib/pocketlab/dracut-force-drivers.missing
```

This keeps the driver inventory explicit without making Rawhide module renames fatal.

## Development Defaults

The root password is currently `147147` so UART login remains possible while display/input/session bring-up is unstable. Remove this once initial setup is reliable enough for the device.
