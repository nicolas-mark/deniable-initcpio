<!-- PROJECT LOGO -->
<br/>
<p align="center">
    <a href="https://github.com/nicolas-mark/deniable-initcpio">
        <img src="docs/logo.png" alt="Logo" width="80" height="80">
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
    </ol>
</details>

<!-- ABOUT THE PROJECT -->
## About The Project

Although there exists an `encrypt` hook, this doesn't suit the requirements for achieving [full-disk encryption](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system) with [plausible deniability](https://en.wikipedia.org/wiki/Plausible_deniability) as the headers must be separated from the device. The purpose of this hook is to achieve plausible deniability using a LUKS-encrypted loop device stored on an encrypted and removable boot partition, containing:
* Detached headers
* Keyfile

The hook will then use the contents of this loop device to mount the LVM contained on the block devices specified in the kernel parameters.

### Built With

This initcpio hook is written in Bash.
* [GNU Bash](https://www.gnu.org/software/bash/)

<!-- GETTING STARTED -->
## Getting Started

Plausible deniability is achieved using the `deniable` hook by storing detached headers and a keyfile in an encrypted loop device. This loop device is contained within the initramfs, which should be stored on an encrypted boot partition on a removable USB.

Below is an example setting up full-disk encryption with plausible deniability using the `deniable` hook. The following block devices will be used in this example:
* **nvme0n1** - m.2 SSD
* **sda** - SATA SSD
* **sdb** - USB (contains encrypted boot and efi)

### Prerequisites

You should have a removable drive containing the encrypted boot partition and EFI partition. An encrypted loop device should be created which will contain the headers and keyfile to be used.

<br/>
<details open="open">
<summary><b>Creating an encrypted loop device</b></summary>
<br/>
An encrypted loop device can be created to contain the block headers and keyfile quite easily.
<br/>
<br/>

1. Create the file to be encrypted and associated with loop.
```
dd if=/dev/urandom of=/tmp/crypto_loop.bin bs=20M count=1 iflag=fullblock
```

2. Using [`cryptsetup`][cryptsetup], create an encrypted loop.
```
cryptsetup luksFormat -y -h sha512 -s 512 --type luks1 /tmp/crypto_loop.bin
```

3. Open the encrypted loop and build the filesystem.
```
cryptsetup open /tmp/crypto_loop.bin crypto_loop
mkfs.ext2 /dev/mapper/crypto_loop
```

4. Mount the filesystem, into which headers and keyfile will be stored.
```
mount /dev/mapper/crypto_loop /mnt
```

<br/>
<details open="open">
<summary><b>Encrypting block devices</b></summary>
<br/>

If you are doing a fresh install with an [Arch Linux Installer USB][installer], block devices can be encrypted prior to configuring the LVM. Optionally, wipe the contents of the block devices before continuing.
<br/>
<br/>
For example, assuming the following configuration:
* **nvme0n1** - m.2 SSD
* **sda** - SATA SSD
* **sdb** - USB (to contain encrypted boot and efi)
<br/>

one might do as follows
<br/>
<br/>

```bash
dd if=/dev/urandom of=/dev/nvme0n1 bs=16M
dd if=/dev/urandom of=/dev/sda bs=16M
```
<br/>

With the encrypted loop device mounted to /mnt headers and keyfile can now appropriately be stored,

1. Create detached headers and a keyfile
```bash
dd if=/dev/zero of=/mnt/nvme0n1_header.bin bs=4M count=1
dd if=/dev/zero of=/mnt/sda_header.bin bs=4M count=1
dd if=/dev/urandom of=/mnt/crypto_keyfile.bin bs=1024 count=1
```

2. Format the devices using [`cryptsetup`][cryptsetup] with detached header and keyfile.
```bash
cryptsetup luksFormat --header=/mnt/nvme0n1_header.bin --key-file=/mnt/crypto_keyfile.bin --keyfile-offset=0 --keyfile-size=512 /dev/nvme0n1

cryptsetup luksFormat --header=/mnt/sda_header.bin --key-file=/mnt/crypto_keyfile.bin --keyfile-offset=512 --keyfile-size=512 /dev/sda
```

3. Open the crypts, then set up [LVM](https://wiki.archlinux.org/index.php/LVM).
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

4. Build the filesystems of [logical volumes](https://wiki.archlinux.org/index.php/LVM#Logical_volumes) and make swap.

```bash
mkfs.ext4 /dev/arch/root /dev/arch/var /dev/arch/home
mkswap /dev/arch/swap
```

5. Mount the filesystems and continue to follow the [Arch Linux Installation Guide][guide].
```
mount /dev/arch/root /mnt
mkdir /mnt/var && mount /dev/arch/var /mnt/var
mkdir /mnt/home && mount /dev/arch/home /mnt/home
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
MODULES=( loop ... )
...
FILES=( /crypto_loop.bin )
...
HOOKS=( base udev autodetect modconf block deniable lvm2 filesystems keyboard fsck )
```

4. Configure the boot loader.
```
GRUB_ENABLE_CRYPTODISK=y
```

<!-- USAGE EXAMPLES -->
## Usage

This hook can be used with the encrypted loop stored in the initramfs
```
cryptloop=rootfs:/crypto_loop
```

or on a separate drive altogether
```
cryptloop=UUID=device-uuid:/crypto_loop 
```

Specify which devices to decrypt in the following format, device:name:header:offset:keysize:options
e.g.
```
cryptdevices=sda:crypt_backup:sda_header.bin:0:512:allow-discards,readonly;nvme0n1:crypt_root:nvme_header:512:512:allow-discards
```

<!-- CONTRIBUTING -->
## Contributing

Contributions are most welcome and make the open source community a great place to learn, create, and get inspired. Any contributions are most welcome, but please sign your commits and follow these steps

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/noice-feature`)
3. Commit your Changes (`git commit -m 'Added some very cool stuff here'`)
4. Push the Changes (`git push origin feature/noice-feature`)
5. Open a Pull Request

Please sign your commits and check out [CONTRIBUTING](docs/CONTRIBUTING.md) for more detailed information.

### Code of Conduct

In the interest of fostering an open and welcoming environment, we pledge to make participation in our project and our community a harassment-free experience for everyone. For more detailed information on what that means, please read [CONTRIBUTING](docs/CONTRIBUTING.md).

<!-- SECURITY POLICY -->
### Security Policy

To report vulnerabilities or security concerns, please read [SECURITY](docs/SECURITY.md).

<!-- LICENSE -->
## License

Distributed under the GPL3 License. See [LICENSE](LICENSE.md) for more information.

<!-- CONTACT -->
## Contact

Nicolas Mark - nicolas.mark@ghostroad.studio 

Project Link: [deniable-initcpio](https://github.com/nicolas-mark/deniable-initcpio)


[installer]: https://wiki.archlinux.org/index.php/USB_flash_installation_medium
[guide]: https://wiki.archlinux.org/index.php/installation_guide
[cryptsetup]: https://linux.die.net/man/8/cryptsetup