# pocketlab

Fedora Rawhide arm64 userspace experiments for hacking on mainline kernel support for pocket computers.

The default build is kernel-less. It produces a Fedora userspace directory tree and an EROFS image suitable for fastboop-style workflows where the kernel, initramfs, and rootfs transport are supplied externally.

Device profiles are optional. When a device profile is selected, mkosi also builds a `device` subimage that layers Fedora's kernel, dracut, BLS, DTB, and repart machinery on top of the same rootfs so the result can still be flashed and booted as a full device image.

## Build

Use mkosi v26 from the local checkout.

Default fastboop-oriented build:

```sh
~/src/mkosi/bin/mkosi summary
~/src/mkosi/bin/mkosi build
```

Default outputs:

```text
mkosi.output/rootfs/
mkosi.output/image.ero
```

The `image.ero` postprocessing step requires `mkfs.erofs` from `erofs-utils` on the host.

Device-image builds:

```sh
~/src/mkosi/bin/mkosi summary --profile sdm845-oneplus-fajita
~/src/mkosi/bin/mkosi summary --profile msm8916-samsung-a5u-eur
~/src/mkosi/bin/mkosi build --profile sdm845-oneplus-fajita
~/src/mkosi/bin/mkosi build --profile msm8916-samsung-a5u-eur
```

Device outputs:

```text
mkosi.output/fedora-sdm845-oneplus-fajita.raw
mkosi.output/fedora-msm8916-samsung-a5u-eur.raw
```

When a device profile is active, the rootfs is still built as the base layer, but `mkosi.output/image.ero` is skipped because the selected artifact is the device disk image.

## Image Layout

`mkosi.conf` is only the shared selector image. A normal build selects `rootfs`.

`mkosi.images/rootfs/` contains the kernel-less Fedora userspace package set, systemd preset policy, and the EROFS post-output hook.

`mkosi.images/device/` depends on `rootfs`, adds Fedora kernel and dracut packages, uses `mkosi.images/device/mkosi.repart/`, and runs the device finalization script.

Top-level profiles in `mkosi.profiles/` only select the `device` image. Device-specific profile payloads live under `mkosi.images/device/mkosi.profiles/` so the rootfs image stays device-agnostic.

## Rootfs Contract

The rootfs image intentionally contains Fedora userspace and no kernel or dracut packages.

Plymouth remains in the rootfs because an external fastboop/initramfs workflow can still drive it.

The Phosh package set intentionally avoids Fedora's `phosh-desktop-environment` comps environment. That environment pulls broad desktop groups such as `hardware-support`, `base-x`, `standard`, `fonts`, and `input-methods`; this image keeps the Phosh shell and default app bundle as an explicit package allowlist instead.

Firmware is explicit and device-oriented: `qcom-firmware` for Qualcomm platform support and `atheros-firmware` for Wi-Fi. Unrelated generic firmware packages are blocked through DNF `excludepkgs` so they are not downloaded during dependency resolution. `fedora-release-mobility` is intentionally excluded for this bring-up image.

`blob-wrangler` is installed from the `samcday/blob-wrangler-nightly` COPR to load device firmware blobs on the target.

## Device Contract

The generated device image is intentionally boring:

- GPT disk image.
- No ESP.
- XBOOTLDR mounted at `/boot`.
- ext4 root filesystem labeled `fedora`.
- Fedora `kernel-install` writes BLS Type #1 entries.
- Fedora `dracut` generates the initramfs.
- pocketboot reads BLS from XBOOTLDR and kexecs Fedora.

The generated XBOOTLDR partition contains:

- `/loader/entries/*.conf`.
- per-kernel `linux`.
- per-kernel `initrd`.
- the per-device DTB selected by the active mkosi profile and copied by Fedora's `90-loaderentry.install`.

This repo avoids mkosi's initrd and UKI paths for device images. `/boot` is produced by Fedora mechanisms:

- `/etc/kernel/install.conf` selects `layout=bls` and `initrd_generator=dracut`.
- `/etc/kernel/cmdline` supplies the kernel command line consumed by Fedora's BLS generator.
- The active device profile provides `/etc/kernel/devicetree`, letting Fedora's `90-loaderentry.install` copy and reference the selected device DTB.
- `mkosi.images/device/mkosi.finalize.chroot` reruns `kernel-install add` after generating a filtered per-device dracut config.

`dracut-config-rescue` is excluded because Fedora's rescue initrd path breaks this non-ESP/XBOOTLDR-only image and is not useful for this device workflow.

## Device Profiles

Pocketlab currently has explicit per-device mkosi profiles:

```text
sdm845-oneplus-fajita
msm8916-samsung-a5u-eur
```

Each device profile provides:

```text
/etc/kernel/devicetree
/usr/lib/pocketlab/dracut-force-drivers.list
```

During device finalization, the driver list is filtered through the installed Rawhide kernel and written to:

```text
/etc/dracut.conf.d/10-pocketlab-<profile>-force-drivers.conf
```

Candidates missing from the installed kernel are recorded in:

```text
/var/lib/pocketlab/dracut-force-drivers.<profile>.missing
```

This keeps the driver inventory explicit without making Rawhide module renames fatal.

## Device Services

`blob-wrangler.service`, `rmtfs.service`, and `tqftpserv.service` are enabled through the image's systemd preset policy and verified during device finalization.

## Development Defaults

The root password is currently `147147` so UART login remains possible while display/input/session bring-up is unstable. Remove this once initial setup is reliable enough for the device.
