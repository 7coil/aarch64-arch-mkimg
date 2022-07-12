# AArch64 Arch Linux Bootable Disk Image Utility

Utility to create a bootable and easily customizable AArch64 Arch Linux disk image.
Supports running on x86 hosts via `binfmt_misc` and `qemu-user-static`.


## Usage

The `aarch64-arch-mkimage` utility allows for multiple profiles, which, in essence, shape the generated disk image.
See notes below for running on an x86 host system.

Running the script directly requires an Arch Linux host system.
To build a disk image or file system trees, run
```
sudo ./aarch64-arch-mkimg <profile>
```
where `<profile>` is the desired profile.
Outputs and intermediate build files are stored in `./build`.
Disk images are written to `./build/disk.img` and final file system trees to `./build/disk/root` for the root partition and `./build/disk/efi` for the EFI/EFS partition.


### Running on an x86 Host

To run this script on an x86 host, emulation for AArch64 is needed.
This can be set up via `binfmt_misc` and `qemu-user-static`.
See e.g. https://github.com/multiarch/qemu-user-static for details on how to set this up.
In short, running
```
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```
should set everything up for you.


### Details

The `aarch64-arch-mkimage` utility works in three major steps:
1. The basic root file system (rootfs) is prepared using the official Arch Linux ARM root archive and base files specified in the profile (`profiles/<name>/base`, these are simply copied to the rootfs tree).
2. The utility switches into the prepared root tree via `arch-chroot`.
   Here, requested packages (`profiles/name/packages`) are installed or uninstalled and customizations to the rootfs specified via the respective profile modules (`profiles/<name>/modules`) are applied.
3. Depending on the profile, a disk image and/or root and EFI file system trees are generated from the final rootfs.
   These are provided in `build/disk.img` and `build/disk/efi` as well as `build/disk/root`.


## Profiles

- `default`: This profile builds an initramfs from the rootfs and packs this up inside an otherwise small boot partition.
  The resulting image boots fully in ram, however, boot times will be slow and the system might seem stuck during boot at times.
  This also means that any changes you make will not persist across reboots.
  This is mostly intended as a basic installation media.

  To use this profile, simply flash the created `build/disk.img` to a USB stick, e.g. via
  ```
  dd if=build/disk.img of=/dev/sdX bs=1m && sync
  ```

- `persistent`: This profile generates EFI and root partition file systems intended for use in a persistent scenario.
  Unfortunately, this means that a full disk image cannot be created automatically (the file system sizes are unknown).
  Therefore, this profile outputs only file system trees at `build/disk/efi` and `build/disk/root`.

  To use this profile, format the USB stick, e.g. via `gdisk` and create an EFI (`gdisk` type `ef00`) partition with 128Mib and a root partition (`gdisk` type `8300` or `8304`).
  You can adapt the partition sizes as needed.
  Format the EFI partition via
  ```
  mkfs.fat -F 32 /dev/sdXY
  ```

  Finally, mount EFI and root partitions and copy over the respective file system trees, e.g. via
  ```
  cp -a build/disk/efi /mnt/efi
  cp -a build/disk/root /mnt/root
  ```


## Customization

You can create or adapt custom profiles.
Each profile provides:

- A `profile.sh` script detailing the build steps.

- A `base` directory, containing additional or replacement files copied directly to the rootfs.
  Files provided here are copied over the Arch Linux ARM rootfs.

- A `packages` directory specifying the packages to install or uninstall
  The `packages/install` file contains a list of packages to be installed.
  The `packages/uninstall` file contains a list of packages to be uninstalled.
  The `packages/custom` directory can contain optional pre-built packages.
  Packages will first be uninstalled, then the chroot system will be updated, new packages will be installed, and finally any custom pre-built packages will be installed.

- A `modules` directory containing module files for rootfs customization.
  Each module can provide a `setup()` and an `install()` function.
  Both will be called in the chroot environment.
  The `setup()` function will be called before any packages are installed and can e.g. be used to add custom repositories.
  The `install()` function will be called after the requested packages have been installed.

See the provided profiles for examples.
