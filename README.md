# QEMU安装测试Arch Linux (aarch64)经验：
测试环境：macOS Tahoe 26.0.1
测试设备：MacBook Air (M3)

## 需要下载的文件：
- ArchLinuxARM-aarch64-latest.tar.gz镜像（不能直接启动）
- alpine-standard-3.23.0-aarch64.iso（可启动）
- QEMU_EFI.fd（如果curl下载下来文件不是2MB左右大小，可手动访问指令中的地址下载，或者使用我仓库中的文件）

- macOS系统中：终端输入：
```zsh
curl -o ~/Downloads/ArchLinuxARM-aarch64-latest.tar.gz https://mirrors.tuna.tsinghua.edu.cn/archlinuxarm/os/ArchLinuxARM-aarch64-latest.tar.gz
curl -o ~/Downloads/alpine-standard-3.23.0-aarch64.iso https://mirrors.tuna.tsinghua.edu.cn/alpine/v3.23/releases/aarch64/alpine-standard-3.23.0-aarch64.iso
curl -o ~/QEMU_EFI.fd https://releases.linaro.org/components/kernel/uefi-linaro/16.02/release/qemu64/QEMU_EFI.fd
```

## 准备QEMU虚拟机环境：
- 安装QEMU本体：
> 如果你还没有安装homebrew：终端输入：
> `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`
```zsh
brew install qemu
```
- 创建磁盘文件：（目录是你的当前用户文件夹，要记住，否则找不到o(^▽^)o）
```zsh
qemu-img create test.raw 20G
```

- 写入分区表：
```zsh
gdisk test.raw << 'EOF'
o
Y
n
1

+512M
EF00
n
2


8300
w
Y
EOF
```
- 检查分区表：
```zsh
gdisk test.raw <<'EOF'
p
EOF
```
> 理想情况显示：
> ```
> Number  Start (sector)    End (sector)  Size       Code  Name
>      1            2048         1050623   512.0 MiB   EF00  EFI system partition
>      2         1050624        41940991   19.5 GiB    8300  Linux filesystem）
> ```

- 转换为qcow2文件（省空间）
```zsh
qemu-img convert -f raw -O qcow2 test.raw test.qcow2
```
得到约524KB的test.qcow2

- 配置适当启动参数：（提示：如果你的设备内存≤16GB，不建议分配8GB内存给虚拟机，实际上2GB完全可以运行，记得修改）
```zsh
qemu-system-aarch64 \
  -accel hvf -cpu host -smp 4 \
  -M virt \
  -m 8G \
  -nographic \
  -device virtio-net-pci,netdev=net \
  -netdev user,id=net,hostfwd=tcp::2222-:22 \
  -cdrom ~/Downloads/alpine-standard-3.23.0-aarch64.iso \
  -bios QEMU_EFI.fd \
  -drive file=test.qcow2,if=virtio 
```

## 进入Alpine Linux
启动后，输入root回车，登录;
- 启用ext4文件系统相关操作：
```bash
apk add e2fsprogs
```
- 查看磁盘：
```bash
fdisk -l
```
> 从输出中记下要安装到哪个磁盘：369M的vda是Alpine的CDROM，不是目标磁盘，应该选择另一个：vdb（这和下面大部分命令都有关，如果你不是vdb，建议依次command+F搜索并替换：vdb1->[你的磁盘名]1 vdb2->[你的磁盘名]2。还挺精准，不会改乱。）（数字一般不变，1是EFI引导分区，2是安装目录）
> 示例输出：
> ```
> Found valid GPT with protective MBR; using GPT
> 
> Disk /dev/vda: 756096 sectors,  369M
> Logical sector size: 512
> Disk identifier (GUID): 35323032-3231-4330-b130-303534383230
> Partition table holds up to 248 entries
> First usable sector is 64, last usable sector is 756032
> 
> Number  Start (sector)    End (sector)  Size Name
>      1              64             811  374K Gap0
>      2             812            3691 1440K EFI boot partition
>      3            3692          756031  367M Gap1
> Found valid GPT with protective MBR; using GPT
> 
> Disk /dev/vdb: 41943040 sectors,     0
> Logical sector size: 512
> Disk identifier (GUID): c3685d8f-206c-4491-98c8-61ed4f05c173
> Partition table holds up to 128 entries
> First usable sector is 34, last usable sector is 41943006
> 
> Number  Start (sector)    End (sector)  Size Name
>      1            2048         1050623  512M EFI system partition
>      2         1050624        41940991 19.4G Linux filesystem
> ```

- 创建文件系统：
```bash
mkfs.ext4 /dev/vdb2
```
（或者强制：`mkfs.ext4 -F /dev/vdb2`）

- 检查文件系统：
```bash
fsck.ext4 -f /dev/vdb2
apk add e2fsprogs e2fsprogs-extra
dumpe2fs /dev/vdb2 | head -20
blkid /dev/vdb2
```
（最后一个命令输出示例：`/dev/vdb2: UUID="b21052d5-7608-493e-9d52-ff7259f68f1f" TYPE="ext4"`）

- 格式化EFI分区：
```bash
mkfs.vfat -F32 /dev/vdb1
```
- 挂载2个分区：（这里很坑！需要用这个命令，否则`Invalid Argument`）
```bash
mount -t ext4 -o defaults,errors=remount-ro /dev/vdb2 /mnt
mkdir /mnt/boot
mount /dev/vdb1 /mnt/boot/
```
- 先启动网络：
```bash
ip link set eth0 up
udhcpc -i eth0
```
- 在Alpine中启动SSH，传Arch Linux镜像：
```bash
apk add openssh openssh-client
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
```
- 设置密码（要输两遍，输入完就回车，不会显示）（`Bad password`不用管，继续输）：
```bash
passwd root
/etc/init.d/sshd start
```
- 检查虚拟机ssh是否在端口22监听：（有ssh字样就是）
```bash
netstat -tlnp | grep :22
```
- 然后在macOS另一个终端：（按⌘+N新建终端）
```zsh
scp -P 2222 ~/Downloads/ArchLinuxARM-aarch64-latest.tar.gz root@localhost:/root/
```
> 如果有`WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`：
> 使用`ssh-keygen -R "[localhost]:2222"`清除密钥；

- 回到Alpine，检查文件是否传输成功：
```bash
ls -lh /root/ArchLinuxARM-aarch64-latest.tar.gz
cd /mnt
```
- 移动到当前工作目录：
```bash
mv /root/ArchLinuxARM-aarch64-latest.tar.gz ./
```
- 检查当前目录
```bash
pwd
ls -lh
```
- 解压：
```bash
tar -xzf ArchLinuxARM-aarch64-latest.tar.gz
```
- 挂载必要的虚拟文件系统：
```bash
mount -t proc /proc /mnt/proc
mount -t sysfs /sys /mnt/sys
mount --rbind /dev /mnt/dev
mount --rbind /run /mnt/run
```

- 复制 DNS 配置：
```bash
cp /etc/resolv.conf /mnt/etc/resolv.conf
```

- 设置主机名：（`arch-vm`字样可自由修改）
```bash
echo "arch-vm" > /mnt/etc/hostname
```

- 切换到新系统
```bash
chroot /mnt /bin/bash
```

- 创建fstab文件：
  1. 获取分区UUID：
  ```bash
  BOOT_UUID=$(blkid -s UUID -o value /dev/vdb1)
  ROOT_UUID=$(blkid -s UUID -o value /dev/vdb2)
  ```
  2. 创建fstab文件
  ```bash
  cat > /etc/fstab << EOF
  # Static information about the filesystems.
  # See fstab(5) for details.

  # <file system> <dir> <type> <options> <dump> <pass>
  UUID=${ROOT_UUID} / ext4 defaults,noatime 0 1
  UUID=${BOOT_UUID} /boot vfat defaults,noatime 0 2
  EOF
  ```

  3. 检查创建的fstab：（确认每个UUID后面都有值！没有的话看看EFI和主分区有没有挂载）
  ```bash
  cat /etc/fstab
  ```
  > 示例：
  > ```
  > # Static information about the filesystems.
  > # See fstab(5) for details.
  > 
  > # <file system> <dir> <type> <options> <dump> <pass>
  > UUID=38c55c13-8929-4f15-aa41-4091a976f413 / ext4 defaults,noatime 0 1
  > UUID=693A-FE3A /boot vfat defaults,noatime 0 2）
  > ```

- 设置时区：
```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

- 设置硬件时钟：
```bash
hwclock --systohc
```

- 设置root密码（要记住！）：
```bash
passwd
```

- （可选）创建普通用户（`archuser`字样可自由修改）：
```bash
useradd -m -G wheel -s /bin/bash archuser
passwd archuser
```

- 配置sudo：
```bash
echo "wheel ALL=(ALL) ALL" >> /etc/sudoers
```

- 启用`systemd-networkd`（网络相关服务）：
```bash
systemctl enable systemd-networkd
systemctl enable systemd-resolved
```

- 配置网络接口
```bash
cat > /etc/systemd/network/20-eth0.network << EOF
[Match]
Name=eth0

[Network]
DHCP=yes
EOF
```
- 添加**清 华 大 学**开源软件镜像站
```bash
echo 'Server = http://mirrors.tuna.tsinghua.edu.cn/archlinuxarm/$arch/$repo' > /etc/pacman.d/mirrorlist
```

- 更新包数据库：
```bash
pacman -Sy
```

- 初始化密钥环：
```bash
pacman-key --init
```

- 填充 Arch Linux ARM 密钥：
```bash
pacman-key --populate archlinuxarm
```

- 安装 GRUB：
```bash
pacman -S grub efibootmgr
```

- 安装 GRUB 为可移动媒体（正常安装老失败，这样效果一样）：
```bash
grub-install --target=arm64-efi --efi-directory=/boot --bootloader-id=GRUB --removable
```

- 生成配置：
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

- 检查：
```bash
head -20 /boot/grub/grub.cfg
```
> 示例：
> ```
> #
> # DO NOT EDIT THIS FILE
> #
> # It is automatically generated by grub-mkconfig using templates
> # from /etc/grub.d and settings from /etc/default/grub
> #
> 
> ### BEGIN /etc/grub.d/00_header ###
> insmod part_gpt
> insmod part_msdos
> if [ -s $prefix/grubenv ]; then
>   load_env
> fi
> 
> if [ "${env_block}" ] ; then
>   set env_block="(${root})${env_block}"
>   export env_block
>   load_env -f "${env_block}"
> fi
> ```

- 安装内核：
```bash
pacman -S linux-aarch64
```

- 查找内核文件：
```bash
find /boot -name "*initramfs-linux.img*" -o -name "*Image*"
```

- 创建 GRUB 内核配置：
```bash
ROOT_UUID=$(blkid -s UUID -o value /dev/vdb2)
cat > /boot/grub/grub.cfg << EOF
set default=0
set timeout=5

insmod part_gpt
insmod ext2
insmod fat

menuentry "Arch Linux ARM" {
    linux /Image root=UUID=$ROOT_UUID rw quiet
    initrd /initramfs-linux.img
}

menuentry "Arch Linux ARM (fallback)" {
    linux /Image root=UUID=$ROOT_UUID rw quiet
    initrd /initramfs-linux-fallback.img
}
EOF
```

- 检查：要确保UUID有值！
```bash
cat /boot/grub/grub.cfg
```
> 示例：
> ```
> set default=0
> set timeout=5
> 
> insmod part_gpt
> insmod ext2
> insmod fat
> 
> menuentry "Arch Linux ARM" {
>     linux /Image root=UUID=38c55c13-8929-4f15-aa41-4091a976f413 rw quiet
>     initrd /initramfs-linux.img
> }
> 
> menuentry "Arch Linux ARM (fallback)" {
>     linux /Image root=UUID=38c55c13-8929-4f15-aa41-4091a976f413 rw quiet
>     initrd /initramfs-linux-fallback.img
> }
> ```

- 安装基本工具（可选）
```bash
pacman -S sudo vim git wget curl
```

- 完成：
```bash
exit
```
- 把窗口关掉。

## 启动Arch Linux（使用新指令）
- macOS终端输入：
```zsh
qemu-system-aarch64 \
  -accel hvf -cpu host -smp 4 \
  -M virt \
  -m 8G \
  -nographic \
  -device virtio-net-pci,netdev=net \
  -netdev user,id=net,hostfwd=tcp::2222-:22 \
  -bios QEMU_EFI.fd \
  -drive file=test.qcow2,if=virtio 
```

> 显示示例：
> ```
> GNU GRUB  version 2:2.14rc1.r54.g29f3131a-2
> 
>  /----------------------------------------------------------------------------\
>  |*Arch Linux ARM                                                             | 
>  | Arch Linux ARM (fallback)                                                  |
>  |                                                                            |
>  |                                                                            |
>  |                                                                            |
>  |                                                                            |
>  |                                                                            |
>  |                                                                            |
>  |                                                                            |
>  |                                                                            |
>  |                                                                            |
>  |                                                                            |
>  |                                                                            | 
>  \----------------------------------------------------------------------------/
> 
>       Use the ^ and v keys to select which entry is highlighted.          
>       Press enter to boot the selected OS, `e' to edit the commands       
>       before booting or `c' for a command-line. 
> ```       

- 会自动引导系统；                  
  输入root回车；
  输入密码进入系统；
  此时系统已联网，可以通过pacman自由安装软件。

## 附：
- 磁盘扩容：macOS终端输入：
  ```zsh
  qemu-img resize test.qcow2 +10G
  ```
  可以增加10GB存储空间。
- 静默启动：
  启动参数中`-nographic`换为
  ```zsh
  -daemonize \
  -display none
  ```
  注意每个`\`后面不能有空格。
- 完整启动脚本示例：`启动Arch Linux_静默.sh`
  - 提示：
    - 如果你的设备内存≤16GB，不建议分配8GB内存给虚拟机，实际上2GB完全可以运行，记得修改；
    - `echo`字样的指令为提示信息，无实际作用；
  ```bash
  #!/bin/bash
  echo "================================"
  echo "  Arch Linux ARM 虚拟机（静默启动）"
  echo "================================"
  echo ""
  echo "连接信息："
  echo "  SSH地址: localhost:1123"
  echo "  用户名: root"
  echo "  密码: 0000"
  echo "  用户名: archuser"
  echo "  密码: 0000"
  echo ""
  echo "================================"
  echo "          SSH使用提示：           "
  echo "================================"
  echo "# 启动 SSH 服务"
  echo "systemctl start sshd"
  echo "systemctl enable sshd"
  echo ""
  echo "# 检查服务状态"
  echo "systemctl status sshd"
  echo ""
  echo "# 允许 root 登录（如果需要）"
  echo "echo \"PermitRootLogin yes\" >> /etc/ssh/sshd_config"
  echo ""
  echo "# 重启 SSH 服务"
  echo "systemctl restart sshd"
  echo ""
  echo "# 重新生成 systemd 配置"
  echo "systemctl daemon-reload"
  echo ""
  echo "直接访问Arch Linux：macOS终端输入："
  echo "ssh -p 1123 root@localhost"
  echo ""
  echo "# 文件传输（macOS终端）"
  echo "scp -P 1123 [本地文件] root@localhost:[远程路径]"
  echo "scp -P 1123 root@localhost:[远程文件] [本地路径]"
  echo "目录请使用：scp -P 1123 -r [本地目录] root@localhost:[远程路径]"
  echo "一键同步文件夹：rsync -rlptvz -e \"ssh -p 1123\" --no-owner --no-group ~/arch_share/ root@localhost:/hostshare/"
  echo ""
  echo "重置连接密钥："
  echo "ssh-keygen -R \"[localhost]:1123\""
  echo ""
  echo "在macOS终端使用"
  echo "socat TCP6-LISTEN:1124,reuseaddr,fork TCP4:127.0.0.1:1123 &"
  echo "socat TCP6-LISTEN:1123,reuseaddr,fork TCP4:127.0.0.1:1124 &"
  echo "来使得虚拟机服务器可被外部IPv6访问"
  echo "Arch Linux启动中..."
  echo "等待3–5秒，系统引导完成后，即可使用Visual Studio Code进行SSH访问。"
  echo "要停止虚拟机运行，可以使用远程终端输入shutdown now，或者直接在活动监视器中结束qemu-system-aarch64进程。"
  qemu-system-aarch64 \
    -accel hvf \
    -cpu host \
    -smp 4 \
    -M virt \
    -m 8G \
    -device virtio-net-pci,netdev=net,mac=52:54:00:12:34:56 \
    -netdev user,id=net,hostfwd=tcp::1123-:22,hostfwd=tcp::1124-:8080 \
    -bios QEMU_EFI.fd \
    -drive "file=arch.qcow2,if=virtio,media=disk" \
    -daemonize \
    -display none
  ```
  在文本编辑器中完成编辑后，保存到`test.qcow2`和`QEMU_EFI.fd`的相同目录，再使用`chmod +x 启动Arch Linux_静默.sh`使文件可执行。
- DeepSeek小提示：
  1. **安全性**：文档中启用了root SSH登录，生产环境应禁用；
  2. **密码强度**：建议使用更强的密码；
  3. **磁盘空间**：20G对于开发环境可能足够，但长期使用建议更大；
