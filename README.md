<!-- PROJECT LOGO -->
<br/>
<p align="center">
    <a href="https://github.com/nicolas-mark/deniable-initcpio">
        <img src="images/logo.png" alt="Logo" width="80" height="80">
    </a>
    <h3 align="center">Deniable Encryption Initcpio</h3>
    <p align="center">
        An initcpio hook to implement full disk encryption with plausible deniability.
        <br/>
        <a href="https://github.com/nicolas-mark/deniable-initcpio"><strong>Explore the docs &#8227</strong></a>
        <br/>
        <a href="https://github.com/nicolas-mark/deniable-initcpio/demo">View Demo</a>
        &#8226
        <a href="https://github.com/nicolas-mark/deniable-initcpio/issues">Report Bug</a>
        &#8226
        <a href="https://github.com/nicolas-mark/deniable-initcpio/issues">Request Features</a>
    </p>
</p>

<!-- TABLE OF CONTENTS -->
<details open="open">
    <summary>Table of Contents</summary>
    <ol>
        <li>
            <a href="#about">About The Project</a>
            <ul>
                <li><a href="#built-with">Built With</a></li>
            </ul>
        </li>
        <li>
        <a href="#getting-started">Getting Started</a>
        <ul>
            <li><a href="#prerequisites">Prerequisites</a></li>
            <li><a href="#installation">Installation</a></li>
        </ul>
        </li>
        <li><a href="#usage">Usage</a></li>
        <!-- <li><a href="#roadmap">Roadmap</a></li> -->
        <li>
            <a href="#contributing">Contributing</a>
            <ul>
                <li><a href="#security-policy">Security Policy</a></li>
            </ul>
        </li>
        <li><a href="#license">License</a></li>
        <li><a href="#contact">Contact</a></li>
        <!-- <li><a href="#acknowledgments">Acknowledgments</a></li> -->
    </ol>
</details>

<!-- ABOUT THE PROJECT -->
## About The Project

Although there exists an `encrypt` initcpio hook, this didn't suit the requirements for achieving [full-disk encryption](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system) with [plausible deniability](https://en.wikipedia.org/wiki/Plausible_deniability). The purpose of this hook is to achieve plausible deniability using a LUKS-encrypted loop device, stored on an encrypted boot partition, containing:
* Detached headers
* Keyfile

The hook will then use these to mount the block devices.

### Built With

This initcpio hook is written in Bash.
* [GNU Bash](https://www.gnu.org/software/bash/)

<!-- GETTING STARTED -->
## Getting Started

This is an example of how to set up full-disk encryption with plausible deniability using the `deniable` hook. The following block devices will be used in this example:
* **nvme0n1** - m.2 SSD
* **sda** - SATA SSD
* **sdb** - USB (contains encrypted boot and efi)

### Prerequisites

You should have a USB containing the encrypted boot partition and an EFI partition. The system drives should be encrypted with detached headers, which can be stored in a LUKS-encrypted loop device in the initramfs.

<br/>
<details open="open">
<summary><b>Encrypting block devices</b></summary>
<br/>

If you're just starting, you will need an [Arch Linux Installer USB](https://wiki.archlinux.org/index.php/USB_flash_installation_medium). Optionally, wipe the contents of the block devices.
<br/>
<br/>
```
dd if=/dev/urandom of=/dev/nvme0n1 bs=16M
...
```

1. Create and mount the encrypted loop device.
```
dd if=/dev/urandom of=/tmp/crypto_loop.bin bs=20M count=1 iflag=fullblock
cryptsetup luksFormat -y -h sha512 -s 512 --type luks /tmp/crypto_loop.bin
cryptsetup open /tmp/crypto_loop.bin crypto_loop
mount /dev/mapper/crypto_loop /mnt
```

2. Use `dd`(https://man7.org/linux/man-pages/man1/dd.1.html) to create detached headers and a keyfile.
```
dd if=/dev/zero of=/mnt/nvme0n1_header.bin bs=4M count=1
dd if=/dev/zero of=/mnt/sda_header.bin bs=4M count=1
dd if=/dev/urandom of=/mnt/crypto_keyfile.bin bs=1024 count=1
```

4. Format the drives using `cryptsetup` with detached header and keyfile.
```bash
cryptsetup luksFormat --header=/mnt/nvme0n1_header.bin --key-file=/mnt/crypto_keyfile.bin --keyfile-offset=0 --keyfile-size=512 /dev/nvme0n1

cryptsetup luksFormat --header=/mnt/sda_header.bin --key-file=/mnt/crypto_keyfile.bin --keyfile-offset=512 --keyfile-size=512 /dev/sda

# This is the encrypted boot partition containing initramfs.
cryptsetup luksFormat /dev/sdb2
```

5. Open the crypts, then set up [LVM](https://wiki.archlinux.org/index.php/LVM).
```bash
cryptsetup open --header=/mnt/nvme0n1_header.bin --key-file=/mnt/crypto_keyfile.bin --keyfile-offset=0 --keyfile-size=512 /dev/nvme0n1 crypt_lvm

cryptsetup open --header=/mnt/sda_header.bin --key-file=/mnt/crypto_keyfile.bin --keyfile-offset=512 --keyfile-size=512 /dev/sda crypt_ext

# Unmount the loop
umount /mnt

# Create physical volumes and volume group
pvcreate /dev/mapper/crypt_lvm
pvcreate /dev/mapper/crypt_ext
vgcreate arch /dev/mapper/crypt_lvm /dev/mapper/crypt_ext

# Partition into logical volumes
lvcreate -L 2G arch -n swap
lvcreate -l +20%FREE arch -n root
lvcreate -l +20%FREE arch -n var
lvcreate -l +100%FREE arch -n home
```

6. Format the [logical volumes](https://wiki.archlinux.org/index.php/LVM#Logical_volumes), mount them, and continue to follow the [Arch Linux Installation Guide](https://wiki.archlinux.org/index.php/installation_guide)

```bash
mkfs.ext4 /dev/arch/root
...
mkswap /dev/arch/swap

mount /dev/arch/root /mnt
...
swapon /dev/arch/swap

# Remember to mount the boot and efi partitions that are on the USB.
cryptsetup open /dev/sdb2 crypt_boot
mount /dev/mapper/crypt_boot /mnt/boot
mount /dev/sdb1 /mnt/efi

# Complete the installation, then move crypto_loop.bin to the root filesystem.
pacstrap /mnt base linux linux-firmware
genfstab -U /mnt >> /mnt/etc/fstab
mv /tmp/crypto_loop.bin /mnt
```
</details>
<br/>

### Installation

1. Clone the repo
```git
git clone git@github.com:nicolas-mark/deniable-initcpio
```

2. Install the hook
```bash
makepkg -Acs
```

3. Modify `mkinitcpio.conf` to include appropriate module, loop file, and initcpio hook.
```
MODULES=( loop )
...
FILES=( /crypto_loop.bin )
...
HOOKS=( base udev autodetect modconf block deniable lvm2 filesystems keyboard fsck )
```

4. Configure the boot loader.
```
```

<!-- USAGE EXAMPLES -->
## Usage

This hook can be used with the encrypted loop in the initramfs
```
cryptloop=rootfs:/crypto_loop ...
```

or on a separate drive altogether
```
cryptloop=UUID=device-uuid:/crypto_loop ...
```

<!-- CONTRIBUTING -->
## Contributing

Contributions are most welcome and make the open source community such a great place to learn, create, and get inspired. Any contributions are most welcome.

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/noice-feature`)
3. Commit your Changes (`git commit -m 'Added some very cool stuff here'`)
4. Push the Changes (`git push origin feature/noice-feature`)
5. Open a Pull Request

<!-- SECURITY POLICY -->
### Security Policy

To report vulnerabilities or security concerns, please read [SECURITY](docs/SECURITY.md).

<!-- LICENSE -->
## License

Distributed under the MIT License. See [LICENSE](docs/LICENSE.md) for more information.

<!-- CONTACT -->
## Contact

Nicolas Mark - nicolas.mark@ghostroad.studio 

Project Link: [deniable-initcpio](https://github.com/nicolas-mark/deniable-initcpio)