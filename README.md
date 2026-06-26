# pocketlab

Fedora Rawhide arm64 image experiments for booting Fedora's stock kernel and userspace on phones through pocketboot.

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
~/src/mkosi/bin/mkosi summary --profile sdm845-oneplus-fajita
~/src/mkosi/bin/mkosi summary --profile msm8916-samsung-a5u-eur
~/src/mkosi/bin/mkosi build --profile sdm845-oneplus-fajita
~/src/mkosi/bin/mkosi build --profile msm8916-samsung-a5u-eur
```

The main outputs are `mkosi.output/fedora-sdm845-oneplus-fajita.raw` and `mkosi.output/fedora-msm8916-samsung-a5u-eur.raw`.

## Boot Contract

pocketboot is required. It must be installed separately on the device and must be able to scan the disk image's XBOOTLDR partition.

The generated XBOOTLDR partition contains:

- `/loader/entries/*.conf`
- per-kernel `linux`
- per-kernel `initrd`
- the per-device DTB selected by the active mkosi profile and copied by Fedora's `90-loaderentry.install`

## Fedora Alignment

This repo avoids mkosi's initrd and UKI paths. `/boot` is produced by Fedora mechanisms:

- `/etc/kernel/install.conf` selects `layout=bls` and `initrd_generator=dracut`.
- `/etc/kernel/cmdline` supplies the kernel command line consumed by Fedora's BLS generator.
- The active mkosi profile provides `/etc/kernel/devicetree`, letting Fedora's `90-loaderentry.install` copy and reference the selected device DTB.
- `mkosi.finalize.chroot` reruns `kernel-install add` after generating a filtered per-device dracut config.

The Phosh package set intentionally avoids Fedora's `phosh-desktop-environment` comps environment. That environment pulls broad desktop groups such as `hardware-support`, `base-x`, `standard`, `fonts`, and `input-methods`; this image keeps the Phosh shell and default app bundle as an explicit package allowlist instead.

Firmware is explicit and device-oriented: `qcom-firmware` for SDM845 platform support and `atheros-firmware` for Wi-Fi. Unrelated generic firmware packages are blocked through DNF `excludepkgs` so they are not downloaded during dependency resolution. `fedora-release-mobility` is intentionally excluded for this bring-up image.

`dracut-config-rescue` is also excluded because Fedora's rescue initrd path breaks this non-ESP/XBOOTLDR-only image and is not useful for this device workflow.

`blob-wrangler` is installed from the `samcday/blob-wrangler-nightly` COPR to load device firmware blobs on the target.

## Device Profiles

Pocketlab currently has explicit per-device mkosi profiles:

```text
sdm845-oneplus-fajita
msm8916-samsung-a5u-eur
```

Each profile provides:

```text
/etc/kernel/devicetree
/usr/lib/pocketlab/dracut-force-drivers.list
```

During finalization, the driver list is filtered through the installed Rawhide kernel and written to:

```text
/etc/dracut.conf.d/10-pocketlab-<profile>-force-drivers.conf
```

Candidates missing from the installed kernel are recorded in:

```text
/var/lib/pocketlab/dracut-force-drivers.<profile>.missing
```

This keeps the driver inventory explicit without making Rawhide module renames fatal.

## Device Services

`blob-wrangler.service`, `rmtfs.service`, and `tqftpserv.service` are enabled through the image's systemd preset policy and verified during finalization.

## Development Defaults

The root password is currently `147147` so UART login remains possible while display/input/session bring-up is unstable. Remove this once initial setup is reliable enough for the device.
