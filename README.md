Here is my dotfiles and configuration files.



## Arch Linux

Installation guide see [HERE](https://wiki.archlinux.org/index.php/installation_guide).

### Pre-installation.安装之前

提示： 用 lsblk 找到U盘并确保没有挂载
用U盘替换 /dev/sdx，如 /dev/sdb。（不要加上数字，也就是说，不要键入 /dev/sdb1 之类的东西)

```console
 # dd bs=4M if=/path/to/archlinux.iso of=/dev/sdx status=progress && sync
```

作用：即准备一块Arch安装盘（其他方式包括且不仅限于[rufus](https://rufus.ie/)）。


### Verify the boot mode.验证启动方式
```bash
ls /sys/firmware/efi/efivars(存在即UEFI，不存在是BIOS或者CSM)
```

### Update the system clock.更新系统时钟

```bash
timedatectl set-ntp true(timedatectl status)
```

### [LVM](https://wiki.archlinux.org/index.php/LVM) on [LUKS](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system).

`lvm2` package.

0. 准备LUKS加密设备.

```bash
cryptsetup --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 10000 luksFormat /root/like.vol
```

1. 打开加密设备.

```bash
cryptsetup luksOpen /root/like.vol cryptlvm
```

2. 创建**pv**.

```bash
pvcreate /dev/mapper/cryptlvm
```

3. 创建**vg**.

```bash
vgcreate MyVolGroup /dev/mapper/cryptlvm
```

4. 创建**lv**

```bash
lvcreate -L 8G MyVolGroup -n swap
lvcreate -L 32G MyVolGroup -n root
lvcreate -l 100%FREE MyVolGroup -n home
```

5. Format the partitions.格式化分区

```bash
mkfs.vfat -F32 /dev/sdxY　# 双系统下不需要，windows已经有了esp分区，挂载即可
mkfs.ext4 /dev/root_partition
mkswap /dev/swap_partition
```

6. Mount the file systems. 挂载分区

```bash
mount /dev/root_partition /mnt
mkdir /mnt/boot
mount /dev/boot_partition /mnt/boot
swapon /dev/swap_parttion
```

7. Select the mirrors.选择镜像

```bash
##
## Arch Linux repository mirrorlist
## Generated on 2020-11-01
##

## China
Server = http://mirrors.163.com/archlinux/$repo/os/$arch
Server = http://mirrors.bfsu.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.bfsu.edu.cn/archlinux/$repo/os/$arch
Server = http://mirrors.cqu.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.cqu.edu.cn/archlinux/$repo/os/$arch
Server = http://mirrors.dgut.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.dgut.edu.cn/archlinux/$repo/os/$arch
Server = http://mirror.lzu.edu.cn/archlinux/$repo/os/$arch
Server = http://mirrors.neusoft.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.neusoft.edu.cn/archlinux/$repo/os/$arch
Server = http://mirrors.nju.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.nju.edu.cn/archlinux/$repo/os/$arch
Server = http://mirror.redrock.team/archlinux/$repo/os/$arch
Server = https://mirror.redrock.team/archlinux/$repo/os/$arch
Server = https://mirrors.sjtug.sjtu.edu.cn/archlinux/$repo/os/$arch
Server = http://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
Server = http://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.xjtu.edu.cn/archlinux/$repo/os/$arch
Server = http://mirrors.zju.edu.cn/archlinux/$repo/os/$arch
```

8. Install essential packages.

```bash
pacstrap /mnt base base-devel linux linux-firmware
```

9. Fstab.

```bash
genfstab -U /mnt >> /mnt/etc/fstab # check:`cat /mnt/etc/fstab`
```

10. Chroot.

```bash
arch-chroot /mnt
```

11. Time zone.

```bash
ln -sf /usr/share/zoneinfo/<Asia>/<Shanghai> /etc/localtime
hwclock --systohc
```

12. Localization.

Edit `/etc/locale.gen` and uncomment `en_US.UTF-8 UTF-8` and other needed [locales](https://wiki.archlinux.org/index.php/Locale). Generate the locales by running:

```
# locale-gen
```

Create the [locale.conf(5)](https://jlk.fjfi.cvut.cz/arch/manpages/man/locale.conf.5) file, and [set the LANG variable](https://wiki.archlinux.org/index.php/Locale#Setting_the_system_locale) accordingly:

```
/etc/locale.conf
LANG=en_US.UTF-8
```

13. Network configuration. [`/etc/systemd/network/20-wired.network`](./etc/systemd/network/20-wired.network)

Create the [hostname](https://wiki.archlinux.org/index.php/Hostname) file:

```
/etc/hostname
myhostname
```

Add matching entries to [hosts(5)](https://jlk.fjfi.cvut.cz/arch/manpages/man/hosts.5):

```
/etc/hosts
127.0.0.1	localhost
::1		localhost
127.0.1.1	myhostname.localdomain	myhostname
```

14. Configuring mkinitcpio.`/etc/mkinitcpio.conf`

```bash
HOOKS=(base systemd autodetect keyboard sd-vconsole modconf block sd-encrypt sd-lvm2 filesystems fsck) # sd-encrypt
```

15. /etc/vconsole.conf`.

```bash
KEYMAP=us
```

16. Configuring the boot loader.` /etc/crypttab.initramfs`,注意是需要解密的设备UUID.

```bash
# <name>       <device>                                     <password>              <options>
externaldrive         UUID=2f9a8428-ac69-478a-88a2-4aa458565431        none    luks,timeout=16
```


17. GRUB.`dosfstools`, `grub`, `efibootmgr`, `os-prober` packages

```bash
grub-install --target=x86_64-efi --efi-directory=<esp_mountpoint> --bootloader-id=Arch_GRUB --recheck # 添加开机启动项目
grub-mkconfig -o /boot/grub/grub.cfg
```

18. Root password.

```bash
passwd
```

19. Add User.

```bash
useradd -m -g <wheel> -s /bin/bash <username>
passwd <username>
visudo(uncomment %wheel ALL=(ALL) ALL)
```

20. Reboot.

```bash
Ctrl^D
umount -R /mnt
reboot
```

### Font. 字体

```
wqy-zenhei \
TODO
```

### archlinuxcn.

  `/etc/pacman.conf`. `archlinuxcn-keyring` package.

```sh
[archlinuxcn]
SigLevel = Optional TrustedOnly
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch
```

### Packages

```
xorg \
cinnamon \
openssh \
firefox \
visual-studio-code-bin \
telegram-desktop \
typora \
p7zip \
fcitx fcitx-rime fcitx-qt5 \
ntfs-3g \
TODO
```

### Backup

```bash
pacman -Qqe > pkglist.txt
```

### Restore

```bash
pacman -S --needed - < pkglist.txt
pacman -S --needed $(comm -12 <(pacman -Slq | sort) <(sort pkglist.txt)) # To filter out from the list the foreign packages
```

### ENV.环境变量
.bashrc: 每次终端时读取并运用里面的设置

.profile：每次启动系统的读取并运用里面的配置

.xinitrc: 每次startx启动X界面时读取并运用里面的设置

.xprofile: 每次使用lightdm等图形登录管理器时读取并运用里面的设置
