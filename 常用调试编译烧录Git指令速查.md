# 常用调试编译烧录 Git 指令速查

本文档汇总工作区中零散记录的调试指令、登录密码、Git 操作、编译方法和烧录方法。命令执行前应先确认当前平台、存储介质、分区表和文件名，避免误烧或误删。

## 1. 登录密码和静态ip设置

当前文档中记录的常用登录密码：

```text
321456
RMSoft1107

samba：\\192.168.82.172\yqtan
192.168.82.172

samba：\\192.168.80.9\yqtan
192.168.80.9

yqtan
JyeU5%c9

yqtan@streamax.com
qwert890A?
```

codex网页下载码

```
9PLM9XGG6VKS
```

建议仅在内部调试环境使用，不要写入对外发布文档、代码仓库或升级包。

```
物理地址. . . . . . . . . . . . . : 20-53-8D-3A-6E-F1
IPv4 地址 . . . . . . . . . . . . : 10.20.123.5(首选)
子网掩码  . . . . . . . . . . . . : 255.255.255.0
默认网关. . . . . . . . . . . . . : 10.20.123.251
DHCPv6 IAID . . . . . . . . . . . : 119559053
DHCPv6 客户端 DUID  . . . . . . . : 00-01-00-01-30-DA-4C-3A-20-53-8D-3A-6E-F1
DNS 服务器  . . . . . . . . . . . : 192.168.6.100
                                    192.168.6.110
```



## 2. 常用调试指令

### 2.1 查看启动参数和分区

查看内核启动参数，重点确认 `root=`、`rootfstype=`、`blkdevparts=`、`MachineType`、`flashsize` 等字段：

```sh
cat /proc/cmdline
```

eMMC 平台查看块设备分区：

```sh
cat /proc/partitions
ls -l /dev/mmcblk*
df -h
```

确认某个 eMMC 分区名，例如确认 `/dev/mmcblk0p4` 是否为 `kernel`：

```sh
cat /sys/block/mmcblk0/mmcblk0p4/uevent | grep PARTNAME
```

NAND Flash 平台查看 MTD 分区：

```sh
cat /proc/mtd
```

注意：eMMC 属于块设备，不走 MTD 子系统；没有 `/proc/mtd` 是正常现象。

### 2.2 U-Boot 启动状态处理

清除 U-Boot 启动异常计数并继续启动：

```sh
status set cnt 0;nvt_boot;boot;
status set cnt 3;
status clr crc;
status set addr 0x12345678;
reset;
nand erase 0x12A00000 0x100000;
```

该命令常用于设备因异常计数进入恢复逻辑时的调试。

### 2.3 杀业务进程

登录系统后停止业务进程：

```sh
cd /usr/local/bin/;
killall -9 *;
```

该操作会强制杀掉当前目录相关业务进程，仅适合调试场景。

### 2.4 MCU 喂狗或关闭看门狗

先查看串口节点：

```sh
ls -l /dev/tty*
```

常见看门狗相关节点：

```text
/dev/ttySpi0
/dev/ttyACM0
/dev/ttyS5
```

向实际节点发送控制命令：

```sh
echo -n -e "\x7e\x80\x01\x36\x07\x01\x30\x7f" > /dev/ttySpi0;
echo -n -e "\x7e\x80\x01\x36\x07\x01\x30\x7f" > /dev/ttyACM0;
echo -n -e "\x7e\x80\x01\x36\x07\x01\x30\x7f" > /dev/ttyS5;
```

旧记录中也出现过以下写法：

```sh
echo -n -e "\x7e\x80\x01\x36\x07\x01\x30\x7f" > /dev/ttySpi0;
echo -n -e "\x7e\x80\x01\x36\x07\x01\x30\x7f" > /dev/ttySpi;
echo -n -e "\x7e\x80\x01\x36\x07\x01\x30\x7f" > /dev/tty;
```

建议不要直接写 `/dev/tty*` 通配符，先确认真实节点再执行。

### 2.5 U 盘挂载

将 FAT 格式 U 盘挂载到 `/tmp/`：

```sh
mount -t vfat /dev/sda1 /tmp/

dd if=/tmp/Firmware_X3N30_M0010_TEST_B3.5.9_RC24072371 of=/dev/mmcblk02 bs=1M
sync
```

执行前可先用 `ls /dev/sd*` 或 `dmesg` 确认 U 盘节点，避免挂错设备。

### 2.6 IP设置

- 设置ip地址

```
ifconfig eth1 10.20.123.242 netmask 255.255.255.0 up
ifconfig eth0 10.20.123.242 netmask 255.255.255.0 up
```

- 查看eth的对应的哪一个gmac

```
ls -l /sys/class/net/eth1
```

### 2.7 查看内存emmc大小

```
#查看已使用的ddr大小
awk '/MemTotal/ {printf "DDR usable: %.2f GiB\n",$2/1048576}' /proc/meminfo

#查看完整的emmc大小
awk '{printf "eMMC total: %.2f GiB\n",$1*512/1073741824}' /sys/class/block/mmcblk0/size
```

```
# 1. 查看完整 DDR 地址和大小
xxd -g4 /sys/firmware/devicetree/base/memory@40000000/reg

# 2. 查看 CMM/媒体预留内存
xxd -g4 /sys/firmware/devicetree/base/reserved-memory/axera_memory_cmm@0/reg
```

### 2.8 查找文件所在的目录名

```
find . -type f -name "*.ko" -exec dirname {} \; | sort -
```

### 2.9 查找当前的ko文件是使用的什么类型的编译器

```
strings ax_audio.ko | grep -E -i "编译器名字"
示例：strings ax_audio.ko | grep -E -i "gcc|clang|toolchain"
```

### 2.10 查找直接的历史shell命令记录

```
history | grep "命令"
示例：history | grep "unsquashfs"
```

### 2.11 解包和打包

```
//解包主系统包
unsquashfs -d local STM_LOCAL_0x14_T22062000
//打包主系统
mksquashfs local/ STM_A5_AHD_LOCAL_T22062400 -b 128k -comp xz -noX -noappend
```

### 2.12 挂载emmc分区

```
umount /dev/mmcblk0p13;	
mkfs.ext2 /dev/mmcblk0p13; 
mount -t ext4 -o rw /dev/mmcblk0p13 /var/mnt/emmc/scare_rw;
df -h /var/mnt/emmc/scare_rw;

mkdir .upgrade
mv Z5_B359_T260722.02_M81031 .packet_raw
scp Z5_B359_T260722.02_M81031 root@10.20.123.242:/var/mnt/emmc/scare_rw/.upgrade
scp -r * root@10.20.123.242:/var/mnt/emmc/scare_rw
```

`p14` 就是 `data2`。建议在主系统中查看，避免备份系统的 `BKUpgradeMain` 反复挂载、卸载它。

先确认是否已经挂载：

```
grep mmcblk0p14 /proc/mounts
```

如果没有输出，按只读方式挂载：

```
mkdir -p /mnt/data2_view
mount -t ext4 -o ro,noload /dev/mmcblk0p14 /mnt/data2_view
```

查看副本：

```
ls -lah /mnt/data2_view
ls -lah /mnt/data2_view/.upgrade
find /mnt/data2_view/.upgrade -maxdepth 2 -type f -ls
```

校验升级包：

```
sha256sum /mnt/data2_view/.upgrade/.packet_raw
ls -l /mnt/data2_view/.upgrade/.packet_raw
```

查看完成后卸载，卸载需要在主系统中进行卸载：

```
umount /mnt/data2_view
```

如果 `grep mmcblk0p14 /proc/mounts` 已显示挂载点，例如：

```
/dev/mmcblk0p14 /某个目录 ext4 ...
```

则直接进入该目录查看，无需重复挂载。不要在 `BKUpgradeMain` 正在扫描时操作 `/mnt/backup_upgrade`，因为该目录会在 p13、p14 之间切换。

可以删除，但建议在主系统中操作，避免 `BKUpgradeMain` 同时扫描 p14。

先单独以读写方式挂载 p14：

```
mkdir -p /mnt/data2_edit
mount -t ext4 -o rw /dev/mmcblk0p14 /mnt/data2_edit
grep ' /mnt/data2_edit ' /proc/mounts
```

删除前确认文件：

```
find /mnt/data2_edit/.upgrade -maxdepth 1 -type f -ls
sha256sum /mnt/data2_edit/.upgrade/.packet_raw
```

确认无误后删除 p14 上的副本：

```
rm -i /mnt/data2_edit/.upgrade/.packet_raw
sync
```

验证并卸载：

```
test ! -e /mnt/data2_edit/.upgrade/.packet_raw && echo "packet deleted"
umount /mnt/data2_edit
```

如果还要删除已经变空的 `.upgrade` 目录，可使用：

```
rmdir /mnt/data2_edit/.upgrade
```

不要直接使用 `rm -rf /mnt/data2_edit/.upgrade`，以免该目录还有其他需要保留的文件。另请注意：p13/data1 还有一份 `.packet_raw`，删除 p14 的副本不会删除 p13 上的升级包。

### 2.13 查看当前目录下各个子目录的大小

```
du -h -d 1 | sort -hr
```

### 2.14 清除分区数据并回读

```
# 若无 flash_erase，用 dd 覆零使全部状态副本失效
dd if=/dev/zero of=/dev/mtdblock10 bs=128k count=8
hexdump -C -n 64 /dev/mtdblock10
```

### 2.15 查看各个小镜像的版本信息

```
strings /dev/mtd2 | grep -E 'U-Boot 20|U_BOOT:'
strings /dev/mtd3 | grep -E 'KERNEL'
```

## 3. Git 常用指令

### 3.1 生成最近一次提交的 patch

```sh
git format-patch -1 HEAD
```

用途：把当前最新提交导出为 `.patch` 文件，便于跨仓库或邮件传递。

### 3.2 使用文件内容作为提交信息

```sh
git commit --no-verify -F ~/git_commit.txt
```

含义：

- `--no-verify`：跳过本地 hook 检查。
- `-F ~/git_commit.txt`：使用指定文件内容作为 commit message。

注意：`--no-verify` 会跳过本地自动检查，建议只在明确知道风险时使用。

### 3.3 克隆驱动仓库

旧记录中的驱动仓库：

```sh
git clone http://192.168.80.128/N9M/devicedriver.git
```

分支：

```text
develop
```

如果网络或权限不可用，需要先确认内网、账号和仓库地址。

### 3.4 追加更改到commit提交

```
git commit --amend -F ~/git_eq
```

### 3.5 查找所有关键词的log提交

```
git log --all --grep="新机型" --grep="新增.*机型" --grep="增加.*机型" --grep="添加.*机型" --
oneline
```

## 4. 编译方法

### 4.1 NOVA SDK 编译

进入 SDK 根目录：

```sh
cd nova/na51103_linux_sdk/
source build/envsetup.sh
lunch
```

常用目标：

```sh
make all
make uboot
make linux
make fdt
make linux_config
```

说明：

- `make all`：编译全部模块。
- `make uboot`：单独编译 U-Boot。
- `make linux`：单独编译内核。
- `make fdt`：单独编译设备树。
- `make linux_config`：进入内核配置。
- 产物通常输出到 `output/` 目录。

### 4.2 U-Boot 单独编译与配置

推荐通过 SDK 顶层环境编译：

```sh
cd na51103_linux_sdk
source build/envsetup.sh
make uboot
```

清理 U-Boot 编译缓存：

```sh
make uboot_clean
```

进入 U-Boot 配置界面：

```sh
make uboot_config
```

如果在 U-Boot 目录裸跑 `make`，出现：

```text
Please set STREAMAX_MACHINE_TYPE
```

需要先设置机型变量，例如：

```sh
export STREAMAX_MACHINE_TYPE=M1N20
make -j16
```

### 4.3 U-Boot 软链接编译错误

编译时如果出现大量软链接失败，例如：

```text
ln: failed to create symbolic link 'drivers/encrypt/Makefile': No such file or directory
create symlic for drivers/encrypt/Makefile fail
create symlic for upgrade/Makefile fail
```

需要检查 U-Boot Makefile 中的 `FILE_LINK_ROOT`：

```text
FILE_LINK_ROOT=~/nfs/git/bksystem/u-boot/common
```

处理方法：确保 `bksystem` 代码已经放到 `~/nfs/git/bksystem/u-boot/common` 对应路径。

### 4.4 AX 平台内核编译

```sh
./build.sh linux -j32
```

### 4.5 驱动编译

准备软链接：

```sh
cd ~/nfs/git/
ln -s ./devicedriver/script/ script
ln -s ./bksystem/recovery/common/ common
ln -s ./devicedriver/Makefile_main Makefile
ln -s ~/nfs/git/devicedriver/DeviceConfig DeviceConfig
```

需要确认驱动配置：

```text
_CONFIG_PLATFORM_NT9852X_=y
_KERNEL_ROOT_DIR_NT9852X_:=/path/to/BSP/linux-kernel
```

编译前先在 SDK 目录初始化环境并编译内核：

```sh
cd /path/to/nova/na_merge/
source build/envsetup.sh
make linux
```

再回到驱动目录编译，例如：

```sh
cd ~/nfs/git/
make gpio _PRODUCT_=ADPLUS
make spimcu _PRODUCT_=M1N2.0
```

注意：旧记录强调编译驱动时应在 `~/nfs/git` 执行，不是在 `~/nfs/git/devicedriver` 执行。

### 4.6 iptables 编译

内核需要打开 netfilter/iptables 相关配置，文档中记录使用 `iptables-1.4.21`。

编译命令：

```sh
./configure --host=arm-linux CC=${arm_xm}-gcc CXX=${arm_xm}-g++ LD=${arm_xm}-ld RANLIB=${arm_xm}-ranlib --disable-nftables --prefix=$PWD/bin
make clean
make install
```

产物目录：

```text
bin/sbin/
```

常见产物包括：

```text
iptables
iptables-restore
iptables-save
ip6tables
xtables-multi
```

### 4.7编译FW、LD和loader文件

```
cd /data2/yqtan/nova/na51103_linux_sdk
source build/envsetup.sh
lunch \
    Linux \
    cortex-a53 \
    cfg_98332_4GX2_EMMC_EVB_X3N30 \
    arm-ca53-linux-gnueabihf-8.4
```

- 编译FW文件

```
cd /data2/yqtan/nova/na51103_linux_sdk
make
￥生成位置：
/data2/yqtan/nova/na51103_linux_sdk/output/packed/FW98332A.bin
```

- 编译LD文件

```
cd /data2/yqtan/nova/na51103_linux_sdk/BSP/na51103_ldr/MakeCommon

make clean
make dep
make release

#生成位置：
/data2/yqtan/nova/na51103_linux_sdk/BSP/na51103_ldr/Project/Model/Loader332_Data/Release/LD98332A.bin
```

- 编译loader文件

```
cd /data2/yqtan/nova/na51103_linux_sdk/BSP/na51103_ldr/MakeCommon

make clean STORAGEEXT=Eth
make dep STORAGEEXT=Eth
make release STORAGEEXT=Eth
#生成位置：
/data2/yqtan/nova/na51103_linux_sdk/BSP/na51103_ldr/Project/Model/Loader332_Data/Release/loader.bin
```

## 5. 烧录方法

烧录前统一建议先确认分区：

```sh
cat /proc/cmdline
cat /proc/partitions
cat /proc/mtd
```

eMMC 使用 `/proc/cmdline` 和 `/proc/partitions`；NAND 使用 `/proc/mtd`。

### 5.1 Linux 下 dd 烧录 NAND 分区

先查看 MTD 分区：

```sh
cat /proc/mtd
```

示例：烧录 U-Boot 到 `mtdblock2`：

```sh
dd if=u-boot.bin of=/dev/mtdblock2 bs=1M conv=fsync
```

示例：烧录扩展应用分区和本地应用分区：

```sh
dd if=STM_EXTEND_0x15_T25112097.ubi of=/dev/mtdblock8 bs=1M conv=fsync
dd if=STM_LOCAL_0x14_T25112097.ubi of=/dev/mtdblock7 bs=1M conv=fsync
```

`conv=fsync` 会在写入完成后同步到物理设备。

### 5.2 Linux 下 dd 烧录 eMMC 分区

先查看启动参数和分区：

```sh
cat /proc/cmdline
cat /proc/partitions
```

确认分区名：

```sh
cat /sys/block/mmcblk0/mmcblk0p4/uevent | grep PARTNAME
```

示例：烧录 kernel 到第 4 分区：

```sh
dd if=STM_973_VA_KERNEL_T26051200 of=/dev/mmcblk0p4 bs=1M conv=fsync
```

注意：不同平台分区顺序不一致，不要只凭经验判断 `p4` 一定是 kernel。

### 5.3 eMMC U-Boot 绝对偏移烧录

M1N20/MIN210 相关文档中，`uboot+3Logo` 分区固定为：

```text
大小: 0x100000 = 1MB
偏移: 0xc0000 = 768KB
```

推荐写整盘绝对偏移：

```sh
dd if=/tmp/uboot.bin of=/dev/mmcblk0 bs=1K seek=768 conv=notrunc
```

等价写法：

```sh
dd if=/tmp/uboot.bin of=/dev/mmcblk0 bs=512 seek=1536 conv=notrunc
```

普通系统下分区节点通常是：

```sh
dd if=/tmp/u-boot.bin of=/dev/mmcblk0p2 conv=notrunc
dd if=/var/mnt/emmc/scare_rw/Z5_1G8G_partition_images/STM_Z5_ROOTFS_T25101000 of=/dev/mmcblk0p7 conv=notrunc

dd if=/var/mnt/emmc/scare_rw/STM_Z5_LOADER_T26072900 of=/dev/mmcblk0p1 conv=notrunc
dd if=/var/mnt/emmc/scare_rw/STM_Z5_UBOOT_T26072900 of=/dev/mmcblk0p3 conv=notrunc
dd if=/var/mnt/emmc/scare_rw/STM_Z5_FDT_T26072900 of=/dev/mmcblk0p4 conv=notrunc
dd if=/var/mnt/emmc/scare_rw/STM_Z5_KERNEL_T26072900 of=/dev/mmcblk0p5 conv=notrunc
dd if=/var/mnt/emmc/scare_rw/STM_Z5_ROOTFS_T26072900 of=/dev/mmcblk0p7 conv=notrunc
sync
```

BAK 系统下分区节点可能是：

```sh
dd if=/tmp/u-boot.bin of=/dev/mmcblk0p3 conv=notrunc
```

关键注意事项：

- 写整盘 `/dev/mmcblk0` 时必须使用 `seek=768` 或 `seek=1536`。
- 不要把 `seek` 写成 `skip`。
- 不要省略偏移，否则可能覆盖 eMMC 起始区域导致无法启动。

### 5.4 U-Boot 下 status tftp 烧录

常见命令：

```sh
status tftp spl spl_AX620_demo_signed.bin
status tftp uboot u-boot_signed.bin
status tftp dtb AX620_demo.dtb
status tftp kernel boot.img
status tftp rootfs rootfs.squashfs
status tftp BKSystem bk.img
status tftp AppLocal AppLocal.squashfs
status tftp AppExt AppExt.squashfs
status tftp data1 data1.ext4
status tftp data2 data2.ext4
status tftp uboot STM_Z3_UBOOT_T23042000
```

### 5.5 U-Boot 下 NAND tftp 手动烧录

配置网络：

```sh
setenv ipaddr 10.20.123.129
setenv serverip 10.20.123.14
setenv gatewayip 10.20.123.251
```

下载镜像到内存：

```sh
tftp 0x00200000 u-boot.bin
```

擦写 U-Boot：

```sh
nand erase 0x80000 0x100000
nand write 0x200000 0x80000 0x100000
```

烧录 rootfs 示例：

```sh
tftp 0x00200000 STM_D6X_L04_ROOTFS_T26010800
nand erase 0x500000 0x1000000
nand write 0x200000 0x500000 0x1000000
```

注意：地址和大小必须来自当前设备 DTS 或分区表，不同机型不能直接套用。

### 5.6 U-Boot 下 eMMC 烧录

示例：

```sh
mmc erase 0x1000 0x100
mmc write 0x40000000 0x1000 0x100
```

说明：

- `mmc erase` 和 `mmc write` 的地址、长度通常以块为单位。
- 写入前要确认镜像已经加载到内存地址 `0x40000000`。
- 块偏移和长度必须与当前平台分区表对应。

### 5.7 BKUpgradeMain 升级

`BKUpgradeMain` 只适合处理打包工具生成的升级包。

升级包放置路径与存储介质有关：

| 存储介质 | 文件系统 | 升级包路径 |
| --- | --- | --- |
| eMMC | EXT4 | `/mnt/backup_upgrade/.upgrade/.packet_raw` |
| NAND Flash | UBIFS | `/mnt/backup_upgrade/.packet_raw` |
| U 盘 | FAT | `/mnt/backup_upgrade/upgrade/*.bin` |

NAND/UBIFS 示例：

```sh
mount
cat /sys/class/ubi/ubi1_0/name
mkdir -p /var/mnt/backup_upgrade/.upgrade/
mount -t ubifs /dev/ubi1_0 /var/mnt/backup_upgrade
./BKUpgradeMain
```

可通过 `get_blk_part_index` 查看拆包后镜像对应的分区。

### 5.8 瑞芯微adb切换loader设备

- 在uboot下敲以下命令后，再去tools页面下选择烧录

```
rockusb 0 mmc 0
```

### 5.9 M3N 空 eMMC：U-Boot 下 TFTP 分区烧录

适用场景：设备处于 U-Boot 命令行，PC 的 TFTP 根目录为：

```text
E:\M1N2.0空emmc烧录资料\Work
```

当前使用的 M3N 镜像：

| 分区 | TFTP 文件 | eMMC 起始块 | 写入块数 | 分区大小 |
| --- | --- | --- | --- | --- |
| Loader（eMMC Boot1） | `STM_M3N_LOADER_T24081900` | `0x0` | `0x200` | 256 KiB |
| FDT | `STM_M3N_FDT_T26050600` | `0x200` | `0x400` | 512 KiB |
| U-Boot | `STM_M3N_UBOOT_T26052500` | `0x600` | `0x800` | 1 MiB |
| Kernel | `STM_M3N_KERNEL_T26010900` | `0xE00` | `0x4000` | 8 MiB |
| RootFS | `STM_M3N_ROOTFS_T26060300` | `0x4E00` | `0x8000` | 16 MiB |
| BKSystem | `STM_98331_BACKUP_T25041600` | `0xCE00` | `0x2000` | 4 MiB |

上述块地址和块数均以 512 字节为单位。BKSystem 按要求继续使用原有镜像。

#### 5.9.1 配置网络并检查 TFTP

在 U-Boot 提示符下逐行执行：

```sh
setenv ipaddr 10.20.123.145
setenv serverip 10.20.123.17
setenv netmask 255.255.255.0
ping 10.20.123.17
```

必须看到：

```text
host 10.20.123.17 is alive
```

#### 5.9.2 检查 eMMC

```sh
mmc list
mmc dev 0
mmc rescan
```

正常情况下应能看到类似：

```text
NVT_MMC0: 0 (eMMC)
```

如果 `mmc dev 0` 或 `mmc rescan` 卡住，不要继续执行任何 `mmc write`，应先检查 eMMC 电源、时钟、pinmux 和 U-Boot MMC 驱动。

#### 5.9.3 首次烧录 Loader 到 eMMC Boot1

先选择 eMMC Boot1，并启用从 Boot1 启动：

```sh
mmc dev 0
mmc partconf 0 1 1 1
```

清理下载内存、下载 M3N Loader，并写满 256 KiB Boot1 区域：

```sh
mw.b 0x1000000 0xff 0x40000
tftpboot 0x1000000 STM_M3N_LOADER_T24081900
mmc write 0x1000000 0x0 0x200
```

确认出现 `write ok` 或 `written OK` 后执行：

```sh
reset
```

设备重启时持续按 `Ctrl+C`，重新进入 U-Boot 命令行。

#### 5.9.4 切回 eMMC User Area

重启进入命令行后，重新配置网络：

```sh
setenv ipaddr 10.20.123.145
setenv serverip 10.20.123.17
setenv netmask 255.255.255.0
ping 10.20.123.17
```

保持 Boot1 为启动分区，但把当前读写区域切换回 eMMC User Area：

```sh
mmc dev 0
mmc partconf 0 1 1 0
mmc dev 0 0
mmc rescan
```

#### 5.9.5 烧录 FDT

FDT 分区：字节偏移 `0x40000`，块偏移 `0x200`，大小 `0x400` 块。

```sh
mw.b 0x1000000 0x00 0x80000
tftpboot 0x1000000 STM_M3N_FDT_T26050600
mmc write 0x1000000 0x200 0x400
```

#### 5.9.6 烧录 U-Boot

U-Boot 分区：字节偏移 `0xC0000`，块偏移 `0x600`，大小 `0x800` 块。

```sh
mw.b 0x1000000 0x00 0x100000
tftpboot 0x1000000 STM_M3N_UBOOT_T26052500
mmc write 0x1000000 0x600 0x800
```

#### 5.9.7 烧录 Kernel

Kernel 分区：字节偏移 `0x1C0000`，块偏移 `0xE00`，大小 `0x4000` 块。

```sh
mw.b 0x1000000 0x00 0x800000
tftpboot 0x1000000 STM_M3N_KERNEL_T26010900
mmc write 0x1000000 0xE00 0x4000
```

#### 5.9.8 烧录 RootFS

RootFS 分区：字节偏移 `0x9C0000`，块偏移 `0x4E00`，大小 `0x8000` 块。

```sh
mw.b 0x800000 0x00 0x1000000
tftpboot 0x800000 STM_M3N_ROOTFS_T26060300
mmc write 0x800000 0x4E00 0x8000
```

#### 5.9.9 烧录 BKSystem

BKSystem 分区：字节偏移 `0x19C0000`，块偏移 `0xCE00`，大小 `0x2000` 块。

```sh
mw.b 0x1000000 0x00 0x400000
tftpboot 0x1000000 STM_98331_BACKUP_T25041600
mmc write 0x1000000 0xCE00 0x2000
```

#### 5.9.10 完成烧录

每条 `tftpboot` 都必须确认文件下载成功；每条 `mmc write` 都必须确认出现 `write ok` 或 `written OK`。全部成功后执行：

```sh
reset
```

关键注意事项：

- Loader 写入的是 eMMC Boot1；FDT、U-Boot、Kernel、RootFS 和 BKSystem 写入的是 eMMC User Area。
- Loader 写完重启后必须切回 User Area，再烧录其余分区。
- `mmc write` 的起始地址和长度以 512 字节块为单位，不能直接填写字节偏移。
- 每次下载前先用 `mw.b` 清理完整分区大小的内存，避免分区尾部写入旧内存数据。
- 本流程是分区烧录，不使用 `FW98332A.bin`。
- 任意一条命令报错或 eMMC 命令卡住时，立即停止后续写入。

### 5.10 Windows 通过 SSH/SCP 向开发板传输镜像

该方法适用于 Windows PC 与开发板网络互通，并且开发板已经启动 SSH 服务的场景。SSH/SCP 主要用于把镜像传到开发板的可写数据分区；烧录前仍需根据 `/proc/cmdline` 和 `/proc/partitions` 确认目标分区。

#### 5.10.1 Windows 检查网络和 SSH 端口

在 Windows PowerShell 中执行：

```powershell
ping 10.20.123.242
Test-NetConnection 10.20.123.242 -Port 22
```

只有看到以下结果后才能使用 `ssh` 和 `scp`：

```text
PingSucceeded    : True
TcpTestSucceeded : True
```

如果 Ping 成功但 `TcpTestSucceeded=False`，说明开发板网络正常，但板端没有进程监听 TCP 22 端口，需要通过串口登录开发板并启动 `sshd`。

#### 5.10.2 板端检查并启动 sshd

在开发板串口终端执行：

```sh
ifconfig eth0
command -v sshd
ps | grep '[s]shd'
mkdir -p /var/run/sshd
/usr/sbin/sshd -t
```

如果 `/usr/sbin/sshd -t` 没有报错，直接启动：

```sh
/usr/sbin/sshd
ps | grep '[s]shd'
netstat -lntp 2>/dev/null | grep ':22'
```

部分只读 SquashFS 系统的 `/etc/passwd` 中没有 `sshd` 用户，会出现：

```text
Privilege separation user sshd does not exist
```

较新版本 OpenSSH 已废弃 `UsePrivilegeSeparation=no`，使用该选项仍不能绕过缺少 `sshd` 用户的问题。可通过临时文件和 bind mount 增加运行期用户，不永久修改只读 rootfs：

```sh
mkdir -p /tmp/sshd-etc /var/empty/sshd /var/run/sshd

cp -L /etc/passwd /tmp/sshd-etc/passwd
cp -L /etc/group /tmp/sshd-etc/group

grep -q '^sshd:' /tmp/sshd-etc/passwd || \
echo 'sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/bin/false' >> /tmp/sshd-etc/passwd

grep -q '^sshd:' /tmp/sshd-etc/group || \
echo 'sshd:x:74:' >> /tmp/sshd-etc/group

mount --bind /tmp/sshd-etc/passwd /etc/passwd
mount --bind /tmp/sshd-etc/group /etc/group

chown 74:74 /var/empty/sshd
chmod 755 /var/empty/sshd

/usr/sbin/sshd -t
/usr/sbin/sshd
ps | grep '[s]shd'
netstat -lntp 2>/dev/null | grep ':22'

dropbearkey -t ed25519 -f /etc/dropbear/dropbear_ed25519_host_key
/etc/init.d/S50dropbear restart
```

配置X3N3.0ssh板端配置的脚本

```
#!/bin/sh
# X3N3.0 板端 SSH 配置脚本
# 在板端串口的 root shell 中执行：
#   sh /路径/配置板端SSH.sh

set -e

BOARD_IP="10.20.123.253/24"
SSH_DIR="/var/mnt/emmc/scare_rw/ssh"
ROOT_HOME="$SSH_DIR/root"
DROPBEAR="$SSH_DIR/dropbear"
HOST_KEY="$SSH_DIR/dropbear_ecdsa_host_key"
PID_FILE="/var/run/dropbear.pid"

# 与本机私钥 id_ecdsa 配套的公钥。公钥可以公开，但不要泄露私钥。
PUBKEY='ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBNFWd50f0L+BYUZ9ewBWflTKkezKzIszHMd6wNS5c9khVM1NdFNBXSH8b4IUrwuiFtjQd7WyDUjVYsdbZf4KHs8= streamax@DESKTOP-IMVELR6'

if [ ! -x "$DROPBEAR" ]; then
    echo "错误：找不到可执行文件 $DROPBEAR"
    echo "请先把 ARMv7 Dropbear 文件复制到该位置，并执行 chmod 700 $DROPBEAR"
    exit 1
fi

# 设置板端 IP；保留 eth0 上固件原有的 10.100.100.1 地址。
ip link set eth0 up
ip addr replace "$BOARD_IP" dev eth0

# 准备持久化的 SSH 目录和授权公钥。
mkdir -p "$ROOT_HOME/.ssh"
printf '%s\n' "$PUBKEY" > "$ROOT_HOME/.ssh/authorized_keys"
chmod 700 "$SSH_DIR" "$DROPBEAR" "$ROOT_HOME" "$ROOT_HOME/.ssh"
chmod 600 "$ROOT_HOME/.ssh/authorized_keys"

# Dropbear 和 dropbearkey 是同一个多调用程序。
ln -sf dropbear "$SSH_DIR/dropbearkey"
if [ ! -s "$HOST_KEY" ]; then
    "$SSH_DIR/dropbearkey" -t ecdsa -s 256 -f "$HOST_KEY"
fi
chmod 600 "$HOST_KEY"

# /etc/passwd 链接到可写的 /var/run/passwd。
# 保留固件现有的 root 密码哈希，只将 root HOME 改到持久化目录。
ROOT_PASS="$(busybox awk -F: '$1 == "root" { print $2; exit }' /var/run/passwd)"
if [ -z "$ROOT_PASS" ]; then
    echo "错误：无法读取原有 root 密码哈希，未修改 passwd"
    exit 1
fi

PASSWD_TMP="$SSH_DIR/passwd.runtime"
printf 'root:%s:0:0::%s:/bin/sh\n' "$ROOT_PASS" "$ROOT_HOME" > "$PASSWD_TMP"
printf 'sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/bin/false\n' >> "$PASSWD_TMP"
cat "$PASSWD_TMP" > /var/run/passwd
chmod 600 /var/run/passwd

# -s：禁用密码登录，只允许上面的密钥登录。
# -E：日志输出到控制台；默认后台运行。
killall dropbear 2>/dev/null || true
"$DROPBEAR" -E -s -p 22 -P "$PID_FILE" -r "$HOST_KEY"

echo
echo "SSH 配置完成"
echo "板端地址：10.20.123.253"
echo "SSH PID：$(cat "$PID_FILE")"
echo "监听端口：22"
ip -4 addr show dev eth0
```

上述 passwd/group 修改仅在当前运行期间有效，开发板重启后自动恢复。

#### 5.10.3 Windows 使用 SCP 复制镜像

PowerShell 通用示例：

```powershell
$Image = 'Z:\path\to\image.bin'
$Board = 'root@10.20.123.242'
$RemoteDir = '/var/mnt/emmc/scare_rw/partition_images'

ssh $Board "mkdir -p $RemoteDir"
scp "$Image" "${Board}:${RemoteDir}/image.bin"
```

`${Board}` 必须加花括号，否则 PowerShell 可能把远端路径前的冒号误解析为变量名的一部分。

Z5 rootfs 传输示例：

```powershell
$Image = 'Z:\axera\AX620E_SDK_V3.0.0_20241120230136_NO1951\Z5_1G8G_partition_images\STM_Z5_ROOTFS_T25101000'
$Board = 'root@10.20.123.242'
$RemoteDir = '/var/mnt/emmc/scare_rw/Z5_1G8G_partition_images'

ssh $Board "mkdir -p $RemoteDir"
scp "$Image" "${Board}:${RemoteDir}/STM_Z5_ROOTFS_T25101000"
```

#### 5.10.4 Windows 和板端分别校验 SHA256

Windows 端：

```powershell
Get-FileHash -Algorithm SHA256 -LiteralPath $Image
```

开发板端：

```powershell
ssh $Board "ls -l $RemoteDir/STM_Z5_ROOTFS_T25101000 && sha256sum $RemoteDir/STM_Z5_ROOTFS_T25101000"
```

两端的文件大小和 SHA256 必须完全一致，才能继续烧录。

#### 5.10.5 烧录前备份当前分区

以 Z5 的 rootfs 分区 `/dev/mmcblk0p7` 为例：

```powershell
ssh $Board "mkdir -p /var/mnt/emmc/scare_rw/Z5_ROOTFS_backup && dd if=/dev/mmcblk0p7 of=/var/mnt/emmc/scare_rw/Z5_ROOTFS_backup/mmcblk0p7_rootfs_before.bin bs=65536 && sync && sha256sum /var/mnt/emmc/scare_rw/Z5_ROOTFS_backup/mmcblk0p7_rootfs_before.bin"
```

重要注意事项：

- 必须先通过 `/proc/cmdline` 和 `/proc/partitions` 核对分区，不能直接套用 `mmcblk0p7`。
- 不建议在当前系统正从 `/dev/mmcblk0p7` 挂载 rootfs 时，通过 SSH 在线 `dd` 覆盖该分区。SquashFS 仍可能按需读取底层块设备，在线覆盖可能造成系统崩溃或镜像写入不完整。
- rootfs 建议先通过 SCP 保存到独立的可写数据分区，再进入 U-Boot、恢复系统或 initramfs 环境完成烧写。
- U-Boot、DTB 等未挂载分区也必须先备份、确认镜像大小不超过分区容量，并在 `dd` 后执行 `sync` 和回读校验。

### 5.11串口烧录空片emmc

[uboot_ymodem_send.ps1](D:/笔记/X3N空片烧录/uboot_ymodem_send.ps1) 是一个 Windows 串口 YMODEM 自动发送脚本，用来把 PC 上的镜像通过 COM3 传到 U-Boot 的 RAM。

```
D:\笔记\X3N空片烧录\uboot_ymodem_send.ps1
```

它主要自动完成：

1. 打开串口：默认 `COM3、115200、8N1`
2. 可选执行 `mw.b` 清理 RAM 缓冲区
3. 执行 U-Boot 命令：

```
loady 0x01000000 115200
```

1. 在 PC 端实现 YMODEM 协议并发送指定文件
2. 显示发送进度和重试情况
3. 计算本地文件 CRC32
4. 可选调用 U-Boot `crc32` 检查 RAM 数据

它本身不会执行以下命令，因此不会直接烧写 eMMC：

```
mmc write
mmc erase
nvt_update_all
reset
```

这些写盘命令是我在文件传输和 CRC 校验完成后另外执行的。

例如传输 FW：

```
& 'D:\笔记\X3N空片烧录\uboot_ymodem_send.ps1' `
  -Port COM3 `
  -File 'E:\X3N3.0空片烧录\work\FW98332A.bin' `
  -Address 0x01000000 `
  -FillLength 0x00800000 `
  -FillValue 0 `
  -SkipRamCrc
```

参数含义：

- `-Port COM3`：使用 COM3
- `-File`：要发送的镜像
- `-Address`：U-Boot RAM 接收地址
- `-FillLength`：传输前清理的 RAM 长度
- `-FillValue`：用 `0x00` 或 `0xFF` 填充
- `-SkipRamCrc`：跳过脚本自动解析板端 CRC

之所以使用 `-SkipRamCrc`，是因为当前厂商 U-Boot 在 YMODEM 完成后偶尔输出两个 `na51103:` 提示符，脚本可能错误截断 CRC 文本。文件传输不受影响，但 CRC 建议随后手动执行，例如：

```
crc32 0x01000000 0x3D79E4
```

需要注意：脚本要求板卡已经进入带有 `loady` 命令的 U-Boot。当前板卡复位后没有进入可识别的 U-Boot，因此在重新通过 PCTOOL 启动临时 U-Boot之前，该脚本暂时无法使用。

下面是我实际执行的串口烧录顺序与命令。所有文件先通过 YMODEM 传到 RAM，校验 CRC 后才执行 eMMC 写入。

重要说明：我没有直接烧录目录中的旧版 `STM_X3N3.0_UBOOT_T24101800` 和 `STM_X3N3.0_KERNEL_T25080700`；U-Boot 和 Kernel 来自新生成的 `FW98332A.bin`。

#### 5.11.1. 串口传输通用流程

U-Boot 端：

```
mw.b 0x01000000 0x00 <缓冲区长度>
loady 0x01000000 115200
```

随后 PC 端用 YMODEM 发送镜像。传输完成后：

```
echo ${filesize}
crc32 0x01000000 <文件实际字节数>
```

我使用了自动发送脚本：

[uboot_ymodem_send.ps1](D:/笔记/X3N空片烧录/uboot_ymodem_send.ps1)

也可以用 Tera Term：

```
File -> Transfer -> YMODEM -> Send
```

#### 5.11.2. 烧录 Loader 到 eMMC Boot1

文件：

```
E:\X3N3.0空片烧录\work\STM_X3N3.0_LOADER_T24031200
```

文件大小：

```
0x10000
```

RAM 清理与传输：

```
mw.b 0x01000000 0xFF 0x40000
loady 0x01000000 115200
```

RAM 校验：

```
crc32 0x01000000 0x10000
```

得到：

```
DCC0D649
```

选择 eMMC Boot1 并写入：

```
mmc partconf 0 1 1 1
mmc write 0x01000000 0x0000 0x0200
```

这里写入了完整的 256 KB Loader 区域，文件后的空间为 `0xFF`。

回读有效的 64 KB：

```
mmc read 0x03000000 0x0000 0x0080
crc32 0x03000000 0x10000
```

回读 CRC：

```
DCC0D649
```

随后立即切回 User Area：

```
mmc partconf 0 1 1 0
mmc partconf 0
```

确认：

```
PARTITION_ACCESS: 0x0
```

#### 5.11.3. 烧录 RootFS

文件：

```
E:\X3N3.0空片烧录\work\STM_X3N3.0_ROOTFS_T25081200
```

文件大小：

```
8990720
0x893000
```

目标分区：

```
User Area，起始块 0x4E00
分区长度 0x8000 块，即 16 MB
```

传输：

```
mw.b 0x01000000 0x00 0x1000000
loady 0x01000000 115200
```

RAM 校验：

```
crc32 0x01000000 0x893000
```

结果：

```
5DDCF010
```

写入：

```
mmc partconf 0 1 1 0
mmc write 0x01000000 0x4E00 0x8000
```

回读有效数据部分：

```
mmc read 0x03000000 0x4E00 0x4498
crc32 0x03000000 0x893000
```

回读 CRC：

```
5DDCF010
```

#### 5.11.4. 烧录 Backup

文件：

```
E:\X3N3.0空片烧录\work\STM_98331_BACKUP_T25041600
```

文件大小：

```
2343360
0x23C1C0
```

目标：

```
User Area，起始块 0xCE00
分区长度 0x2000 块，即 4 MB
```

传输：

```
mw.b 0x01000000 0x00 0x400000
loady 0x01000000 115200
```

RAM 校验：

```
crc32 0x01000000 0x23C1C0
```

结果：

```
C88CC233
```

写入：

```
mmc partconf 0 1 1 0
mmc write 0x01000000 0xCE00 0x2000
```

回读有效数据：

```
mmc read 0x03000000 0xCE00 0x11E1
crc32 0x03000000 0x23C1C0
```

回读 CRC：

```
C88CC233
```

#### 5.11.5. 烧录kernel

文件：

```
E:\X3N3.0空片烧录\work\STM_X3N3.0_KERNEL_T25080700
```

文件大小：

```
4028900
0x3D79E4
```

传输：

```
mw.b 0x01000000 0x00 0x800000
loady 0x01000000 115200
```

写入：

```
mmc partconf 0 1 1 0
mmc write 0x01000000 0xE00 0x4000
```

#### 5.11.6. 烧录fdt

文件：

```
E:\X3N3.0空片烧录\work\STM_X3N3.0_FDT_T26013000
```

文件大小：

```
4028900
0x3D79E4
```

传输

```
mw.b 0x01000000 0x00 0x800000
loady 0x01000000 115200

```

写入

```
mmc partconf 0 1 1 0
mmc write 0x1000000 0x200 0x400
```

#### 5.11.7. 烧录uboot

文件：

```
E:\X3N3.0空片烧录\work\STM_X3N3.0_UBOOT_T24101800
```

文件大小：

```
4028900
0x3D79E4
```

传输

```
mw.b 0x01000000 0x00 0x100000
loady 0x01000000 115200
```

写入

```
mmc partconf 0 1 1 0
mmc write 0x1000000 0x600 0x800
```

#### 5.11.8. 最终全分区回读校验

FDT：

```
mmc read 0x03000000 0x0200 0x0084
crc32 0x03000000 0x107F5
```

结果：

```
44DA7335
```

新 U-Boot：

```
mmc read 0x03000000 0x0600 0x05FA
crc32 0x03000000 0xBF364
```

结果：

```
A6484D59
```

Kernel：

```
mmc read 0x03000000 0x0E00 0x183F
crc32 0x03000000 0x307D90
```

结果：

```
40BA9917
```

RootFS：

```
mmc read 0x03000000 0x4E00 0x4498
crc32 0x03000000 0x893000
```

结果：

```
5DDCF010
```

Backup：

```
mmc read 0x03000000 0xCE00 0x11E1
crc32 0x03000000 0x23C1C0
```

结果：

```
C88CC233
```

Loader：

```
mmc partconf 0 1 1 1
mmc read 0x03000000 0x0000 0x0080
crc32 0x03000000 0x10000
```

结果：

```
DCC0D649
```

最后恢复 User Area：

```
mmc partconf 0 1 1 0
mmc partconf 0
```

最终状态：

```
BOOT_ACK: 0x1
BOOT_PARTITION_ENABLE: 0x1
PARTITION_ACCESS: 0x0
```

最后我执行了：

```
reset
```

复位后串口输出乱码，没有进入可识别的 U-Boot，因此写入和回读 CRC 是成功的，但上述 Loader/启动分区方案仍需重新确认。

## 6. 根文件系统制作

常见流程是基于已有 SquashFS 镜像解包、修改、重新打包。

查看文件系统类型：

```sh
file STM_X1SG1_ROOTFS_T26050700
```

解包：

```sh
unsquashfs STM_X1SG1_ROOTFS_T26050700
```

查看压缩算法：

```sh
unsquashfs -s STM_X1SG1_ROOTFS_T26050700 | grep Compression
```

重新打包，例如原镜像使用 `lz4`：

```sh
mksquashfs squashfs-root new_rootfs.squashfs -comp lz4 -noappend
```

注意：压缩算法必须与内核支持能力匹配。若压缩算法不支持，可能出现根文件系统挂载失败和 kernel panic。

## 7. 操作前检查清单

1. 确认平台：NOVA、AX、瑞芯微或其它平台。
2. 确认存储介质：eMMC、NAND、SPI NOR/NAND 或 U 盘。
3. 确认当前分区表：`cat /proc/cmdline`、`cat /proc/partitions`、`cat /proc/mtd`。
4. 确认目标分区名：不要只按分区号判断。
5. 确认镜像文件和设备匹配。
6. 写 eMMC 整盘时确认 `seek` 偏移。
7. 写 NAND 前确认擦除起始地址和长度。
8. 量产或交付前避免使用 `--no-verify`、硬编码密码和临时调试命令。

## 8. Git 常见问题处理

### 8.1 `index.lock` 锁文件导致 Git 命令失败

现象：执行 `git add`、`git reset --hard`、`git status` 等命令时报错：

```text
fatal: Unable to create '/data2/yqtan/RK3588/.git/index.lock': File exists.
Another git process seems to be running in this repository...
```

原因：Git 在修改索引时会创建 `.git/index.lock`。如果上一次 Git 命令异常退出，锁文件可能残留，后续 Git 命令就无法写入索引。

处理步骤：

```sh
cd /data2/yqtan/RK3588

ps -ef | grep '[g]it'
```

如果没有当前用户正在操作 `/data2/yqtan/RK3588` 仓库的 Git 进程，可以删除锁文件：

```sh
rm -f /data2/yqtan/RK3588/.git/index.lock
```

确认锁文件已经删除：

```sh
ls -l /data2/yqtan/RK3588/.git/index.lock
```

如果输出类似下面内容，表示锁文件已经不存在：

```text
No such file or directory
```

然后重新执行 Git 命令：

```sh
git status
```

注意：如果当前目录在 `/data2/yqtan/RK3588/device`，直接执行 `rm -f .git/index.lock` 是删不到的，因为 `.git` 在仓库根目录 `/data2/yqtan/RK3588/.git`。

### 8.2 恢复被误删的已跟踪文件

现象：`git status` 显示：

```text
deleted:    kernel-5.10/drivers/net/wireless/rockchip_wlan/rtl8821cu/8821cu.ko
deleted:    kernel-5.10/drivers/net/wireless/rockchip_wlan/rtl8821cu/8821cu.mod
```

如果这些修改不需要，使用 `git restore` 恢复：

```sh
cd /data2/yqtan/RK3588

git restore kernel-5.10/drivers/net/wireless/rockchip_wlan/rtl8821cu/8821cu.ko \
            kernel-5.10/drivers/net/wireless/rockchip_wlan/rtl8821cu/8821cu.mod
```

也可以恢复整个仓库中所有已跟踪文件的修改：

```sh
git reset --hard HEAD
```

注意：`git reset --hard HEAD` 会丢弃所有已跟踪文件的本地修改，执行前要确认没有需要保留的源码改动。

### 8.3 清理 kernel 编译产生的未跟踪文件

现象：`git status` 中出现大量 kernel 编译产物，例如：

```text
kernel-5.10/.config
kernel-5.10/.config.old
kernel-5.10/.version
kernel-5.10/Module.symvers
kernel-5.10/include/generated/
kernel-5.10/scripts/mod/modpost
kernel-5.10/scripts/dtc/dtc
```

先预览将要删除的文件：

```sh
cd /data2/yqtan/RK3588

git clean -fdn kernel-5.10 tmp
```

确认预览内容都是不要的编译产物后，再真正删除：

```sh
git clean -fd kernel-5.10 tmp
```

参数说明：

| 参数 | 含义 |
| --- | --- |
| `-f` | 强制删除未跟踪文件 |
| `-d` | 同时删除未跟踪目录 |
| `-n` | 只预览，不真正删除 |

注意：不要在仓库根目录直接执行 `git clean -fd`，否则会删除所有未跟踪文件，包括你可能想保留的 `device/DriverIpSwitch.ko`、临时补丁、日志等。

### 8.4 添加外部编译好的 ko 文件

如果要把编译好的 ko 文件添加到 Git，例如：

```text
/data2/yqtan/RK3588/device/DriverIpSwitch.ko
```

推荐从仓库根目录执行：

```sh
cd /data2/yqtan/RK3588

git add device/DriverIpSwitch.ko

git status
```

注意路径是 `device/DriverIpSwitch.ko`，不是 `devices/DriverIpSwitch.ko`。

如果当前已经在 `device` 目录，也可以执行：

```sh
git add DriverIpSwitch.ko
```

但为了避免路径写错，推荐统一回到仓库根目录执行 `git add device/DriverIpSwitch.ko`。

### 8.5 清理和提交前推荐流程

当仓库中同时存在需要保留的 ko 文件和不需要的 kernel 编译产物时，推荐流程如下：

```sh
cd /data2/yqtan/RK3588

# 1. 如果存在历史锁文件，先确认并删除
ps -ef | grep '[g]it'
rm -f /data2/yqtan/RK3588/.git/index.lock

# 2. 恢复误删的已跟踪文件
git restore kernel-5.10/drivers/net/wireless/rockchip_wlan/rtl8821cu/8821cu.ko \
            kernel-5.10/drivers/net/wireless/rockchip_wlan/rtl8821cu/8821cu.mod

# 3. 预览清理 kernel 编译产物
git clean -fdn kernel-5.10 tmp

# 4. 确认无误后执行真正清理
git clean -fd kernel-5.10 tmp

# 5. 添加需要保留的 ko 文件
git add device/DriverIpSwitch.ko

# 6. 查看最终状态
git status
```

最终 `git status` 应只显示你明确需要提交的文件。若还有 `.config`、`Module.symvers`、`include/generated/` 等内容，说明 kernel 编译产物还没有清理干净。

## 9. 工程管理系统

### 9.1 创建项目并开始

```
创建并开始开发项目：
axera_Z5_2GB-DDR适配。硬件资料在……；硬件具体更改内容
```

### 9.2 提取临时会话并总结

```
请归档当前会话到所属项目，但不要把项目标记为 completed。

提取并回写：
1. 本次需求和关键决策
2. 已修改或计划修改的源码路径
3. 编译、测试和实测指标
4. 硬件修改及相关资料
5. 已验证的可复用经验
6. 未完成事项和下一步

更新对应的 .project 结构化记录和 SUMMARY.md，
然后刷新项目索引。
不要保存密码，不要记录未经验证的推测。
```

### 9.3  汇报当前的项目的进度

```
打开当前已有项目。

先读取：
1. .project/SUMMARY.md
2. .project/project.json
3. 未完成事项和结构化工程记录

不要批量加载其他项目，也不要立即访问远程 SDK。
先向我汇报项目状态、已有成果、待完成事项和建议的下一步。
```

### 9.4 其他平台的agent介入该系统

```
接管当前嵌入式项目。

请先读取：
1. AGENTS.md
2. .project/project.json
3. .project/SUMMARY.md
4. 相关结构化工程记录

然后使用 $check-engineering-workspace 检查当前状态。
不要批量加载其他项目，不要立即修改远程 SDK；
先汇报项目现状、已有成果、风险和未完成事项。
```

### 9.5 项目目录文件整理

- 直接输入：

```
整理当前项目目录，先只生成目录整理预览，不移动任何文件。
```

- 确认预览后，再输入：

```
执行已经确认的目录整理计划，只处理预览中列出的文件。
```

