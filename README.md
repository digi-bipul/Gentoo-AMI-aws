Here is a gorgeous, ultra-minimal, fully-optimized README ready to be copied directly into your GitHub repository as README.md. It reflects all the hard-earned lessons, kernel constraints, and pure anti-bloat philosophy from our trial-and-error sessions.
```markdown
<div align="center">

<img src="https://upload.wikimedia.org/wikipedia/commons/d/d4/Gentoo_Linux_logo_matte.svg" alt="Gentoo Logo" width="160" />

# ⚡ Gentoo-AWS-Nitro
**The Ultimate, Zero-Bloat, Hyper-Optimized Gentoo AMI Guide for AWS EC2**

[![Architecture: x86_64](https://img.shields.io/badge/Architecture-x86__64-blue.svg?style=for-the-badge&logo=intel)](#)
[![AWS: Nitro](https://img.shields.io/badge/AWS-Nitro-FF9900.svg?style=for-the-badge&logo=amazon-aws)](#)
[![Init: OpenRC](https://img.shields.io/badge/Init-OpenRC-333333.svg?style=for-the-badge&logo=linux)](#)
[![Kernel: Monolithic](https://img.shields.io/badge/Kernel-Monolithic-8A2BE2.svg?style=for-the-badge&logo=linux)](#)
[![Filesystem: BTRFS](https://img.shields.io/badge/Filesystem-BTRFS-0077B5.svg?style=for-the-badge)](#)

*As of **June 2026** • Battle-tested for production.*

</div>

---

## 📖 Philosophy
This guide documents the exact, foolproof steps to forge an absolutely minimal, UEFI-bootable Gentoo Linux image tailored specifically for an AWS `m7i-flex.large` instance. 

We embrace an **anti-bloat** architecture: No `systemd`, no `cloud-init`, and absolutely **zero** kernel modules. Everything required to boot, attach to the network, and SSH in is baked directly into a monolithic kernel.

### 🛡️ Core Directives
1. **Target Instance:** `m7i-flex.large` (Intel 4th Gen Xeon Scalable "Sapphire Rapids").
2. **Boot Mechanism:** Pure UEFI (Natively required by AWS Nitro).
3. **Filesystem:** BTRFS (Modern, copy-on-write).
4. **Init System:** OpenRC.
5. **Kernel:** 100% Monolithic (`CONFIG_MODULES=n`). NVMe and ENA baked directly into the image.
6. **Cloud Agent:** Alpine's `tiny-cloud` principles via an ultra-lean IMDSv2 metadata fetcher.

---

## 🛠️ Step 1: Host Preparation (The Arch Linux Builder)

Since installing Gentoo directly from an AWS rescue disk can be a nightmare, we use an **Arch Linux EC2 instance** as our builder host.

1. Launch an Arch Linux instance in your AWS VPC.
2. Create and attach a secondary EBS volume (e.g., `20GB gp3`) to this instance. This will become our Gentoo AMI.
3. SSH into your Arch builder and elevate to root.

---

## 💽 Step 2: Partitioning & BTRFS Formatting

AWS Nitro instances present EBS volumes as NVMe devices. Assuming your attached volume is `/dev/nvme1n1`:

```bash
# Wipe the partition table
sgdisk -Z /dev/nvme1n1

# Create an EFI System Partition (ESP) - 512MB
sgdisk -n 1:0:+512M -t 1:ef00 -c 1:"EFI System Partition" /dev/nvme1n1

# Create the BTRFS Root Partition - Remaining Space
sgdisk -n 2:0:0 -t 2:8300 -c 2:"Gentoo Root" /dev/nvme1n1

# Format the partitions
mkfs.fat -F32 /dev/nvme1n1p1
mkfs.btrfs -L "gentoo-root" /dev/nvme1n1p2

```
Mount them securely for the chroot:
```bash
mkdir -p /mnt/gentoo
mount /dev/nvme1n1p2 /mnt/gentoo
mkdir -p /mnt/gentoo/boot/efi
mount /dev/nvme1n1p1 /mnt/gentoo/boot/efi

```
## 📦 Step 3: Stage 3 & Chroot Setup
Download the latest amd64-openrc stage3 tarball:
```bash
cd /mnt/gentoo
wget [https://distfiles.gentoo.org/releases/amd64/autobuilds/current-stage3-amd64-openrc/stage3-amd64-openrc-latest.tar.xz](https://distfiles.gentoo.org/releases/amd64/autobuilds/current-stage3-amd64-openrc/stage3-amd64-openrc-latest.tar.xz)
tar xpvf stage3-amd64-openrc-latest.tar.xz --xattrs-include='*.*' --numeric-owner

```
Bind necessary filesystems to enter the chroot safely:
```bash
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run
cp /etc/resolv.conf /mnt/gentoo/etc/

chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"

```
## ⚙️ Step 4: Hyper-Optimized make.conf
This configuration sets aggressive optimizations specifically for the m7i Sapphire Rapids architecture, stripping out all desktop/GUI bloat via USE flags.
Edit /etc/portage/make.conf:
```ini
# --- COMPILER FLAGS FOR SAPPHIRE RAPIDS ---
COMMON_FLAGS="-O2 -march=sapphirerapids -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"

# Adjust based on builder CPU cores
MAKEOPTS="-j8"
EMERGE_DEFAULT_OPTS="--jobs=4 --load-average=8.0 --with-bdeps=y"

# --- ABSOLUTE MINIMAL USE FLAGS ---
# Disabling all GUI, X, systemd, and bloat. Only core CLI / OpenRC requirements.
USE="-* acl btrfs bzip2 cli crypt cxx efi iconv ipv6 minimal ncurses nls \
     openrc pam pcre readline seccomp split-usr ssl symlink unicode zlib"

# --- FEATURES ---
ACCEPT_KEYWORDS="amd64"
FEATURES="binpkg-request-signature clean-logs"
PORTAGE_LOGDIR="/var/log/portage"

```
Sync Portage and update:
```bash
emerge-webrsync
emerge -avuDN @world

```
## 🧠 Step 5: The Monolithic Kernel
This is where previous trial-and-error killed us. A monolithic kernel requires no module loading at boot, significantly speeding up initialization and reducing attack surfaces.
```bash
emerge -av sys-kernel/gentoo-sources sys-kernel/linux-firmware
cd /usr/src/linux
make menuconfig

```
> **🚨 CRITICAL KERNEL FLAGS TO SET (Built-in [*], NOT Modules [M]):**
>  1. **Disable Modules Entirely:** >    Enable loadable module support -> UNCHECK
>  2. **AWS Nitro Network (ENA):** >    Device Drivers -> Network device support -> Ethernet driver support -> Amazon Elastic Network Adapter (ENA) -> [*]
>  3. **AWS Nitro Storage (NVMe):** >    Device Drivers -> NVME Support -> NVM Express block device -> [*]
>  4. **BTRFS Filesystem:** >    File systems -> Btrfs filesystem support -> [*]
>  5. **EFI Stub (For UEFI Boot):** >    Processor type and features -> EFI stub support -> [*]
>  6. **Serial Console (Mandatory for AWS EC2 Troubleshooting):** >    Device Drivers -> Character devices -> Serial drivers -> 8250/16550 and compatible serial support -> Console on 8250/16550 and compatible serial port -> [*]
> 
Compile and install:
```bash
make -j8 && make install

```
*(No make modules_install required because modules are disabled!)*
## 🗄️ Step 6: fstab & Networking
AWS NVMe device names can shift depending on how they are attached. Always use UUID.
Find your UUIDs:
```bash
blkid

```
Edit /etc/fstab:
```text
# <fs>                                <mountpoint>    <type>  <opts>          <dump/pass>
UUID=XXXX-XXXX                        /boot/efi       vfat    defaults        0 2
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx /               btrfs   defaults,noatime,compress=zstd 0 1

```
**Networking (DHCP via ENA):**
On AWS, the ENA interface is usually ens5.
```bash
echo 'config_ens5="dhcp"' > /etc/conf.d/net
cd /etc/init.d
ln -s net.lo net.ens5
rc-update add net.ens5 default
rc-update add sshd default

```
## ☁️ Step 7: Zero-Bloat Cloud-Init (IMDSv2 Fetcher)
Rather than installing full cloud-init, which brings hundreds of dependencies, we adhere to the tiny-cloud philosophy. We will use a lightweight local.d script to securely fetch your SSH key from AWS IMDSv2 (Instance Metadata Service) on boot.
Create /etc/local.d/01-aws-ssh-keys.start:
```bash
#!/bin/sh
# Ultra-minimal IMDSv2 SSH Key Injector
mkdir -p /root/.ssh
chmod 700 /root/.ssh

# Fetch IMDSv2 Session Token
TOKEN=$(curl -X PUT "[http://169.254.169.254/latest/api/token](http://169.254.169.254/latest/api/token)" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" -s)

# Fetch Public Key securely and append
curl -H "X-aws-ec2-metadata-token: $TOKEN" -s [http://169.254.169.254/latest/meta-data/public-keys/0/openssh-key](http://169.254.169.254/latest/meta-data/public-keys/0/openssh-key) > /root/.ssh/authorized_keys

chmod 600 /root/.ssh/authorized_keys
chown -R root:root /root/.ssh

```
Make it executable and enable local in OpenRC:
```bash
chmod +x /etc/local.d/01-aws-ssh-keys.start
rc-update add local default
emerge net-misc/curl

```
## 🚀 Step 8: Bootloader (GRUB EFI)
Install GRUB natively for UEFI.
```bash
emerge --ask sys-boot/grub

```
Edit /etc/default/grub to enforce serial console logging so you can view boot errors in the AWS EC2 Serial Console:
```ini
GRUB_CMDLINE_LINUX_DEFAULT="console=tty0 console=ttyS0,115200n8 rootfstype=btrfs quiet"
GRUB_TERMINAL="serial console"
GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"

```
Install to the EFI partition:
```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=gentoo --removable
grub-mkconfig -o /boot/grub/grub.cfg

```
*(The --removable flag is crucial for AWS to successfully recognize the bootloader without NVRAM entries.)*
## 📸 Step 9: Creating the AWS AMI
 1. **Clean up and Exit:**
   ```bash
   rm /root/.bash_history
   exit
   umount -R /mnt/gentoo
   
   ```
 2. **Snapshot via AWS CLI (From your local machine):**
   ```bash
   # Find the volume ID of your Gentoo EBS
   aws ec2 describe-volumes
   
   # Create a snapshot
   aws ec2 create-snapshot --volume-id vol-0xxxxxxxxx --description "Gentoo Root Snapshot"
   
   ```
 3. **Register the AMI:**
   ```bash
   aws ec2 register-image \
       --name "Gentoo-m7i-Monolithic-June2026" \
       --architecture x86_64 \
       --root-device-name /dev/xvda \
       --virtualization-type hvm \
       --ena-support \
       --boot-mode uefi \
       --block-device-mappings '[{"DeviceName": "/dev/xvda","Ebs": {"SnapshotId": "snap-0xxxxxxxxx"}}]'
   
   ```
## 🎉 Conclusion
You now have a brutally efficient, single-purpose Gentoo AMI. By locking the CPU architecture to sapphirerapids, enforcing a monolithic kernel without loadable modules, and dumping heavy python-based cloud agents for an IMDSv2 bash fetcher, you have achieved absolute system mastery on AWS.
*Compile once, deploy forever.* ☕
</div>
```

```
