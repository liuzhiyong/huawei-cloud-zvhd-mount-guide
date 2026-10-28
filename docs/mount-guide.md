# Kali Linux 下读取华为云导出的 ZVHD 磁盘备份

**作者：** 刘志勇  
**日期：** 2026年10月28日  
**操作环境：** Kali Linux x86_64  
**主要工具：** `qemu-img-hw`、GCC、zlib、QEMU NBD、LVM

## 一、背景与目标

在处理华为云上的 Linux 服务器备份时，有时需要将导出的磁盘文件下载到本地，在不启动原服务器的情况下读取其中的文件。

华为云支持多种磁盘镜像格式，包括 VHD、QCOW2、ZVHD 和 ZVHD2。文件扩展名不一定能准确反映实际格式，需要先检查文件。

华为云的 `qemu-img-hw` 是用于处理相关镜像格式的专用工具。官方文档说明，该工具用于镜像格式转换，可在 Linux 环境中使用。

官方参考资料：

- [华为云：使用 qemu-img-hw 转换镜像格式](https://support.huaweicloud.com/eu/bestpractice-ims/ims_bp_0052.html)
- [华为云：Linux 环境下快速导入镜像文件](https://support.huaweicloud.com/eu/usermanual-ims/ims_01_0340.html)

本次实践的目标是：

1. 在 Kali Linux 上安装必要的编译依赖。
2. 处理 `qemu-img-hw` 的动态库兼容性问题。
3. 编译兼容库并运行 `qemu-img-hw`。
4. 将识别出的 ZVHD 磁盘转换为 RAW 格式。
5. 使用 NBD 访问磁盘分区。
6. 识别 LVM 逻辑卷并读取目标文件。
7. 完成卸载和设备清理。

## 二、环境与注意事项

### 2.1 环境说明

本文记录的是一次实际操作成功的流程。已验证的操作包括：

- 编译 `libcrypto.so.10` 兼容库。
- 运行 `qemu-img-hw info`。
- 使用 `qemu-img-hw convert` 转换磁盘。
- 通过 NBD 访问磁盘分区。
- 挂载文件系统并读取数据。

不同版本的 Kali Linux、GCC、zlib、glibc 和 `qemu-img-hw` 可能存在差异，因此不能保证所有环境都能直接复现。

建议记录自己的环境信息：

```bash
uname -a
dpkg --print-architecture
gcc --version
ldd --version
zlib-flate -version 2>/dev/null || true
```

其中最后一条命令仅用于尝试获取相关工具信息，不是判断 zlib 版本的标准方法。也可以使用：

```bash
dpkg-query -W zlib1g
```

### 2.2 重要风险提示

磁盘镜像可能包含系统配置、账户信息、密钥和业务数据。

- 保留原始备份，不要直接修改它。
- 尽可能以只读方式访问磁盘。
- 转换前确认目标磁盘空间充足。
- 激活 LVM 前确认卷组名称，避免与本机已有卷组冲突。
- 不要将未经验证的动态库直接覆盖系统原有库文件。
- 处理生产数据时，应遵守组织的数据安全和访问授权要求。

## 三、安装依赖

更新软件包索引：

```bash
sudo apt update
```

安装编译工具及 zlib 开发文件：

```bash
sudo apt install -y gcc patchelf zlib1g-dev
```

检查软件包是否安装成功：

```bash
gcc --version
dpkg-query -W zlib1g-dev
```

### 3.1 OpenSSL 兼容性问题

在本次环境中，`qemu-img-hw` 存在对 `libcrypto.so.10` 的依赖，而较新的系统通常使用不同版本的 OpenSSL 动态库。

**不同版本的 OpenSSL 动态库不能仅凭文件名进行替换。**

本次实践采用了一个兼容库方案，将工具需要的部分压缩接口转发到系统 zlib，以解决特定环境下的运行问题。

这是一种针对特定工具和环境的兼容性处理方法，不是完整的 OpenSSL 1.0.x 实现。

如果目标系统提供了可用的原版兼容库，优先考虑使用工具提供方认可的依赖环境。

## 四、编译 libcrypto.so.10 兼容库

### 4.1 创建工作目录

建议在独立目录中操作：

```bash
mkdir -p ~/zvhd-work
cd ~/zvhd-work
```

创建两个文件：

- `fake_crypto.c`
- `fake_crypto.map`

### 4.2 编写 fake_crypto.c

将以下内容保存为 `fake_crypto.c`：

```c
#define _GNU_SOURCE

#include <zlib.h>
#include <stdlib.h>
#include <string.h>

__asm__(".symver COMP_compress_block,COMP_compress_block@libcrypto.so.10");
__asm__(".symver COMP_expand_block,COMP_expand_block@libcrypto.so.10");
__asm__(".symver COMP_CTX_new,COMP_CTX_new@libcrypto.so.10");
__asm__(".symver COMP_zlib,COMP_zlib@libcrypto.so.10");
__asm__(".symver ERR_load_COMP_strings,ERR_load_COMP_strings@libcrypto.so.10");

#define COMP_ZLIB 1

typedef struct comp_ctx_st {
    int method;
    z_stream *dstrm;
    z_stream *cstrm;
    int dinit;
    int cinit;
} COMP_CTX;

int COMP_zlib(void) {
    return COMP_ZLIB;
}

void *COMP_CTX_new(int method) {
    if (method != COMP_ZLIB)
        return NULL;

    COMP_CTX *ctx = calloc(1, sizeof(COMP_CTX));
    if (!ctx)
        return NULL;

    ctx->method = method;
    ctx->dstrm = calloc(1, sizeof(z_stream));
    ctx->cstrm = calloc(1, sizeof(z_stream));

    return ctx;
}

int COMP_compress_block(void *ctx_v, unsigned char *out, int olen,
                        unsigned char *in, int ilen) {
    COMP_CTX *ctx = (COMP_CTX *)ctx_v;

    if (!ctx || !ctx->cstrm)
        return -1;

    if (!ctx->cinit) {
        deflateInit(ctx->cstrm, Z_DEFAULT_COMPRESSION);
        ctx->cinit = 1;
    }

    ctx->cstrm->next_in = in;
    ctx->cstrm->avail_in = ilen;
    ctx->cstrm->next_out = out;
    ctx->cstrm->avail_out = olen;

    int ret = deflate(ctx->cstrm, Z_FULL_FLUSH);

    return (ret == Z_OK) ? ctx->cstrm->total_out : -1;
}

int COMP_expand_block(void *ctx_v, unsigned char *out, int olen,
                      unsigned char *in, int ilen) {
    COMP_CTX *ctx = (COMP_CTX *)ctx_v;

    if (!ctx || !ctx->dstrm)
        return -1;

    if (!ctx->dinit) {
        inflateInit(ctx->dstrm);
        ctx->dinit = 1;
    }

    uLong prev = ctx->dstrm->total_out;

    ctx->dstrm->next_in = in;
    ctx->dstrm->avail_in = ilen;
    ctx->dstrm->next_out = out;
    ctx->dstrm->avail_out = olen;

    int ret = inflate(ctx->dstrm, Z_FULL_FLUSH);

    if (ret != Z_OK && ret != Z_STREAM_END)
        return -1;

    return ctx->dstrm->total_out - prev;
}

void ERR_load_COMP_strings(void) {
}
```

### 4.3 编写 fake_crypto.map

将以下内容保存为 `fake_crypto.map`：

```text
libcrypto.so.10 {
    global:
        COMP_compress_block;
        COMP_expand_block;
        COMP_CTX_new;
        COMP_zlib;
        ERR_load_COMP_strings;
    local:
        *;
};
```

### 4.4 编译兼容库

执行：

```bash
gcc -shared -fPIC \
    -Wl,--version-script=fake_crypto.map \
    -o libcrypto.so.10 \
    fake_crypto.c -lz
```

检查生成的文件：

```bash
file libcrypto.so.10
readelf -d libcrypto.so.10
objdump -T libcrypto.so.10 | grep COMP
```

**注意：** 以上代码是本次实践使用的实验性兼容实现。它并未实现完整的 OpenSSL ABI，也没有覆盖所有压缩接口。

尤其需要注意：

- `COMP_CTX_new()` 中的内存分配和 zlib 流初始化需要配套的错误处理及资源释放逻辑。
- 压缩函数返回的长度必须符合调用方预期，累计输出量不一定等于本次调用产生的字节数。
- `inflate()` 的输入、输出及刷新方式需要与原始调用约定一致。
- 符号版本声明和导出符号名称一致，并不能证明函数语义完全兼容。

因此，应当将其视为特定环境下的实验性实现，而不是可直接用于其他程序的通用替代库。

### 4.5 加载兼容库

为避免覆盖系统原有动态库，建议优先使用独立目录加载：

```bash
mkdir -p ~/zvhd-work/lib
cp libcrypto.so.10 ~/zvhd-work/lib/
```

先检查 `qemu-img-hw` 的动态库依赖：

```bash
ldd /path/to/qemu-img-hw
```

将 `/path/to/qemu-img-hw` 替换为实际程序路径。

在确认程序确实需要该兼容库，并且其余依赖满足要求后，可以尝试：

```bash
LD_LIBRARY_PATH="$HOME/zvhd-work/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}" \
    /path/to/qemu-img-hw --help
```

如果工具能够正常启动，再进一步检查目标镜像。

这种方式可以避免为了测试单个工具而直接替换系统公共库。

如果出现 `version not found`、未定义符号或段错误，应先检查实际依赖和符号要求，不要仅凭错误信息就反复覆盖系统库。

## 五、安装 qemu-img-hw

从华为云官方文档提供的渠道获取工具包：

[华为云 qemu-img-hw 官方说明及下载入口](https://support.huaweicloud.com/eu/bestpractice-ims/ims_bp_0052.html)

官方工具包名称为 `qemu-img-hw.zip`。

下载后，进入文件所在目录并解压：

```bash
unzip qemu-img-hw.zip
```

检查目录结构：

```bash
find qemu-img-hw -type f -name 'qemu-img-hw'
```

如果解压后的路径与示例一致：

```bash
chmod +x qemu-img-hw/qemu-img-hw/qemu-img-hw
```

建议先保留工具在独立目录中运行，以便明确控制其动态库搜索路径。

如果确实需要安装到 `/usr/local/bin/`，可以执行：

```bash
sudo install -m 0755 \
    qemu-img-hw/qemu-img-hw/qemu-img-hw \
    /usr/local/bin/qemu-img-hw
```

之后可以通过以下命令检查：

```bash
qemu-img-hw --help
```

如果使用独立兼容库目录，应在运行工具时按上一节所述设置 `LD_LIBRARY_PATH`。

## 六、识别并转换 ZVHD 磁盘

假设下载的文件名为：

`10.213.170.140-vda.vhd`

### 6.1 检查文件

先检查文件大小：

```bash
ls -lh 10.213.170.140-vda.vhd
file 10.213.170.140-vda.vhd
```

然后尝试读取镜像信息：

```bash
qemu-img-hw info 10.213.170.140-vda.vhd
```

如果需要设置兼容库搜索路径，应使用前面验证成功的运行方式。

本次示例输出：

```text
image: 10.213.170.140-vda.vhd
file format: zvhd
virtual size: 100G (107374182400 bytes)
disk size: 15G
```

以上是示例输出。实际结果取决于备份文件。

`virtual size` 表示虚拟磁盘容量，`disk size` 表示文件实际占用空间，两者可能相差较大。

### 6.2 转换为 RAW

确认工具能够识别源文件后，执行：

```bash
qemu-img-hw convert -p -O raw \
    10.213.170.140-vda.vhd \
    10.213.170.140-vda.raw
```

如果使用独立兼容库目录，应将命令放在对应的 `LD_LIBRARY_PATH` 环境中执行。

转换进度会随工具的处理过程变化。耗时取决于磁盘容量、实际数据量、压缩情况及存储性能。

转换前应确认目标目录有足够的可用空间：

```bash
df -h .
```

转换完成后检查文件：

```bash
ls -lh 10.213.170.140-vda.raw
file 10.213.170.140-vda.raw
```

## 七、通过 NBD 访问 RAW 镜像

### 7.1 安装相关工具

安装 QEMU 工具和 LVM 工具：

```bash
sudo apt update
sudo apt install -y qemu-utils lvm2
```

加载 NBD 内核模块：

```bash
sudo modprobe nbd max_part=8
```

检查设备：

```bash
ls -l /dev/nbd*
```

### 7.2 连接磁盘镜像

先确认 `/dev/nbd0` 没有被其他任务占用。

连接镜像：

```bash
sudo qemu-nbd --read-only \
    --connect=/dev/nbd0 \
    10.213.170.140-vda.raw
```

使用只读模式可以降低意外修改磁盘镜像的风险。

检查分区表：

```bash
sudo fdisk -l /dev/nbd0
```

也可以执行：

```bash
lsblk -f /dev/nbd0
```

### 7.3 分区表示例

假设输出如下：

```text
Disk /dev/nbd0: 100 GiB, ... sectors
Disklabel type: gpt

Device        Start       End   Sectors  Size Type
/dev/nbd0p1    2048   1230847   1228800  600M EFI System
/dev/nbd0p2 1230848   3327999   2097152    1G Linux filesystem
/dev/nbd0p3 3328000 209713151 206385152 98.4G Linux LVM
```

以上仅为示例，实际分区布局应以命令输出为准。

如果系统没有自动识别分区，可以尝试：

```bash
sudo partprobe /dev/nbd0
sudo udevadm settle
lsblk -f /dev/nbd0
```

## 八、挂载分区和 LVM 逻辑卷

### 8.1 普通分区

先检查文件系统：

```bash
sudo blkid /dev/nbd0p2
```

确认分区类型和用途后，创建挂载目录：

```bash
sudo mkdir -p /mnt/guest_boot
```

如果文件系统适合当前内核读取，可以尝试只读挂载：

```bash
sudo mount -o ro /dev/nbd0p2 /mnt/guest_boot
```

检查文件：

```bash
ls -la /mnt/guest_boot
```

注意，分区编号不一定与文件系统用途一一对应，不要未经检查就认定 `nbd0p2` 一定是 `/boot`。

### 8.2 扫描 LVM

如果磁盘包含 LVM 分区，先检查本机当前的 LVM 状态：

```bash
sudo pvs
sudo vgs
sudo lvs
```

扫描卷组：

```bash
sudo pvscan
sudo vgscan
```

假设扫描结果显示目标卷组为 `klas`。

**如果本机已经存在同名卷组，不能直接执行激活命令。** 应先确认卷组归属、UUID 及设备映射，避免误操作本机已有存储。

确认没有名称冲突并明确目标设备后，才可以尝试：

```bash
sudo vgchange -ay klas
sudo lvs klas
```

`klas` 是示例名称，应替换为实际识别出的卷组名称。

### 8.3 挂载根逻辑卷

假设检查结果确认根逻辑卷为：

`/dev/mapper/klas-root`

创建挂载目录：

```bash
sudo mkdir -p /mnt/guest_root
```

如果底层设备只读、文件系统状态允许只读访问，可以尝试：

```bash
sudo mount -o ro /dev/mapper/klas-root /mnt/guest_root
```

查看文件：

```bash
ls -la /mnt/guest_root
```

如果挂载失败，应先检查逻辑卷路径、文件系统类型和错误信息，不要直接执行文件系统修复操作。

### 8.4 提取文件

例如，需要复制某个配置文件，可以使用：

```bash
sudo cp /mnt/guest_root/etc/os-release ~/os-release.backup
```

这里仅以 `/etc/os-release` 为例，实际文件路径应根据目标系统的目录结构确定。

## 九、ARM64 系统的读取问题

如果原始服务器使用 ARM64（aarch64）架构，而本地 Kali Linux 运行在 x86_64 主机上，只要磁盘格式和文件系统受支持，通常仍然可以读取并复制文件。

但需要区分以下两种操作：

- **读取文件系统：** 主要取决于镜像格式、分区结构、文件系统及内核支持。
- **运行原系统程序：** 还涉及 CPU 架构、运行时环境和系统依赖。

因此，读取 ARM64 系统中的文件不等于能够直接在 x86_64 主机上运行其中的程序。

## 十、卸载和清理

完成文件读取后，先卸载实际挂载的文件系统：

```bash
sudo umount /mnt/guest_root
sudo umount /mnt/guest_boot
```

如果只挂载了其中一个目录，就只卸载实际挂载的目录。

确认相关文件系统均已卸载后，再停用本次使用的卷组：

```bash
sudo vgchange -an klas
```

执行前必须确认 `klas` 是本次操作使用的目标卷组，不能影响本机其他业务。

最后断开 NBD：

```bash
sudo qemu-nbd --disconnect /dev/nbd0
```

检查设备状态：

```bash
lsblk
```

如果设备仍被占用，应先排查未卸载的文件系统、打开的文件或仍在使用的逻辑卷。

## 十一、常见问题

### 11.1 `qemu-img-hw` 报告动态库版本错误

例如：

```text
libcrypto.so.10: version not found
```

建议依次检查：

```bash
ldd /path/to/qemu-img-hw
readelf -d ~/zvhd-work/lib/libcrypto.so.10
objdump -T ~/zvhd-work/lib/libcrypto.so.10 | grep COMP
```

将 `/path/to/qemu-img-hw` 替换为实际路径。

重点确认：

- 工具实际需要哪些动态库。
- 兼容库是否导出了目标符号。
- 符号版本是否符合工具要求。
- 运行时是否从预期目录加载动态库。

不能仅凭 `SONAME` 正确就认定兼容库完全兼容。

### 11.2 出现 Segmentation fault

段错误可能由多种原因引起，包括：

- 动态库 ABI 不匹配。
- 函数参数或返回值不符合调用约定。
- 内存分配或释放存在问题。
- zlib 流的初始化、刷新或释放方式不正确。
- 输入文件损坏或工具自身存在问题。

应结合错误日志和实际依赖逐步排查，不要直接断定一定是某个编译参数导致。

### 11.3 `qemu-img-hw` 找不到命令

检查程序路径和执行权限：

```bash
command -v qemu-img-hw
ls -l /usr/local/bin/qemu-img-hw
```

如果程序没有安装到 PATH 中，可以使用绝对路径执行。

### 11.4 找不到分区

检查：

```bash
sudo fdisk -l /dev/nbd0
lsblk -f
sudo blkid
```

如果仍然无法识别，应进一步检查镜像格式、分区表、LVM 或加密结构。

### 11.5 LVM 卷组无法激活

检查物理卷、卷组及逻辑卷信息：

```bash
sudo pvs
sudo vgs
sudo lvs
```

如果出现卷组名称或 UUID 冲突，应先制定隔离方案，不要直接批量激活卷组。

## 十二、总结

本次实践验证了在 Kali Linux 环境下处理特定华为云 ZVHD 磁盘备份的一条可行路径：

**兼容性处理 → qemu-img-hw 识别 → 转换为 RAW → NBD 访问 → 分区及 LVM 检查 → 只读挂载 → 文件提取 → 清理设备。**

需要注意的是，实验性兼容库的成功运行只证明它在本次测试条件下满足了工具的实际需求，不代表它实现了完整的 OpenSSL 兼容性，也不代表它适用于所有版本的工具或所有镜像文件。

本文旨在记录实际技术实践，为遇到类似问题的 Linux 用户提供排查思路。

欢迎通过 GitHub Issues 反馈经过验证的改进方法。
