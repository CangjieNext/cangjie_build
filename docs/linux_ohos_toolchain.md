# ohos-x86_64、ohos-aarch64工具链 (Ubuntu 22.04)
说明：本教程将构建出OHOS SDK中宿主平台无关部分的工具链，其将在 [Linux OHOS 构建指导](linux_ohos_zh.md)、[Linux ->Windows(OHOS) 交叉构建指导](linux_cross_windows_ohos_zh.md)、[macOS OHOS 构建指导](macos_ohos_zh.md) 上分别被打包进各自的完整SDK中。

## 1 环境准备

### 1.1 系统要求

- **操作系统**: Ubuntu 22.04 LTS

- **磁盘空间**: ≥200GB
- **内存**: ≥8GB (物理内存+交换空间)
- **用户权限**: 普通用户+sudo权限
- **芯片指令集**：x86_64

### 1.2 系统工具安装

- 使用华为镜像源(可选)

```bash
  # 1.备份配置文件：
  sudo cp -a /etc/apt/sources.list /etc/apt/sources.list.bak;
  # 2.修改sources.list文件，将http://archive.ubuntu.com和http://security.ubuntu.com替换成http://mirrors.huaweicloud.com，可以参考如下命令：
  sudo sed -i "s@http://.*archive.ubuntu.com@http://mirrors.huaweicloud.com@g" /etc/apt/sources.list;
  sudo sed -i "s@http://.*security.ubuntu.com@http://mirrors.huaweicloud.com@g" /etc/apt/sources.list;
  # 3.更新索引
  sudo apt-get update;
```

- 安装系统工具

```bash
sudo apt update && sudo apt install -y --no-install-recommends \
openssh-server openssh-client openjdk-8-jdk vim wget curl bzip2 zip unzip sudo \
tar build-essential ninja-build zlib1g-dev libncurses5-dev libxml2-dev libedit-dev \
dos2unix libstdc++6 binutils libc-dev libc++-dev flex bison lld python3-requests \
unrar git git-lfs make openssl libssl-dev libffi-dev iputils-ping maven less \
libcurl4 expat gettext nfs-common libtool sqlite3 libcurl4-openssl-dev autoconf \
libxext-dev libxtst-dev libxt-dev libcups2-dev libfreetype6-dev libasound2-dev  \
libfontconfig1-dev libxrender-dev libopenmpi-dev  gdb lldb ccache rpm patch  \
libtinfo5 cpio rpm2cpio libncurses5 strace net-tools python3-jinja2 python3-pexpect  \
rapidjson-dev  libelf-dev libdwarf-dev rsync sshpass ssh python3-markupsafe tar \
libgdbm-dev libnss3-dev  libreadline-dev  libbz2-dev libncurses-dev libtinfo-dev \
gnupg gperf libc6-dev-i386 x11proto-core-dev libx11-dev libgl1-mesa-dev libxml2-utils \
xsltproc bc ruby gcc-aarch64-linux-gnu libxcursor-dev libxrandr-dev libxinerama-dev \
gcc g++ clang libgmp-dev libmpfr-dev libmpc-dev;
```

## 2 编译ohos_aarch64、ohos_x86-64工具链

### 2.1 创建工作区

**<font color="red">\* 下方示例的`/path/to/workspace`需更换为您构建ohos的工作目录</font>**

```bash
export WORKSPACE=/path/to/workspace
mkdir -p $WORKSPACE;
cd $WORKSPACE;
# 在执行prebuilts_download.sh的时候会在上层创建存放ohos相关二进制目录，这里创建一个source目录用于存放ohos源码
mkdir source;
```

### 2.2 repo下载
```bash
cd $WORKSPACE;
curl https://raw.gitcode.com/gitcode-dev/repo/raw/main/repo-py3 -o repo;
chmod a+x repo;
pip3 install -i https://repo.huaweicloud.com/repository/pypi/simple requests;
```

### 2.3 OHOS源码下载

```bash
cd ${WORKSPACE}/source;
# 请在命令执行前配置git user.name和user.email
# 如果有报python找不到错的话将`#!/usr/bin/env python` 改成 `#!/usr/bin/env python3`
${WORKSPACE}/repo init -u https://gitcode.com/openharmony/manifest.git -b master --no-repo-verify;
${WORKSPACE}/repo sync -c;
```

### 2.4 交叉编译ohos musl

```bash
# vim third_party/musl/BUILD.gn, 在最后加上下面代码
group("all_musls") {
  deps = [
    ":soft_libc_musl_shared(//build/toolchain/ohos:ohos_clang_arm64)",
    ":soft_libc_musl_shared(//build/toolchain/ohos:ohos_clang_x86_64)",
  ]
}
```
```bash
# 执行musl构建
cd ${WORKSPACE}/source;
./build/prebuilts_download.sh;
./build.sh --product-name rk3568 --build-target all_musls;
```

### 2.5 交叉编译ohos openssl

```bash
# vim third_party/openssl/BUILD.gn, 在最后加上下面代码
group("all_ssl") {
  deps = [
    ":libcrypto_static(//build/toolchain/ohos:ohos_clang_arm64)",
    ":libssl_static(//build/toolchain/ohos:ohos_clang_arm64)",
    ":libcrypto_shared(//build/toolchain/ohos:ohos_clang_arm64)",
    ":libssl_shared(//build/toolchain/ohos:ohos_clang_arm64)",
    ":ssl_source(//build/toolchain/ohos:ohos_clang_arm64)",
    ":libcrypto_static(//build/toolchain/ohos:ohos_clang_x86_64)",
    ":libssl_static(//build/toolchain/ohos:ohos_clang_x86_64)",
    ":libcrypto_shared(//build/toolchain/ohos:ohos_clang_x86_64)",
    ":libssl_shared(//build/toolchain/ohos:ohos_clang_x86_64)",
    ":ssl_source(//build/toolchain/ohos:ohos_clang_x86_64)",
  ]
}
```
```bash
# 执行构建
cd ${WORKSPACE}/source;
./build.sh --product-name rk3568 --build-target all_ssl;
```

### 2.6 构建ohos-sdk所需工具链
```bash
cd ${WORKSPACE}/source;
./build.sh --product-name ohos-sdk --ccache;
```

### 2.7 拷贝工具链
```bash
# OHOS_ROOT环境变量指定为您用于保存 ohos 工具链的路径
export OHOS_ROOT=/opt/buildtools/ohos_root;
cd ${WORKSPACE}/source;

mkdir -p ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot
cp -r out/sdk/obj/third_party/musl/usr ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/ && \
cp -r out/sdk/sdk-native/os-irrelevant/sysroot/usr ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/ && \
cp -r out/rk3568/ohos_clang_arm64/obj/third_party/musl/usr ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/ && \
cp -r out/rk3568/ohos_clang_x86_64/obj/third_party/musl/usr ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/ && \
cp -r out/rk3568/ohos_clang_x86_64/obj/third_party/openssl/build_all_generated/include/openssl ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/include/x86_64-linux-ohos && \
cp -r out/rk3568/ohos_clang_arm64/obj/third_party/openssl/build_all_generated/include/openssl ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/include/aarch64-linux-ohos && \
cp -r third_party/openssl/include/openssl/macros.h ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/include/aarch64-linux-ohos/openssl/ && \
cp -r third_party/openssl/include/openssl/* ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/include/aarch64-linux-ohos/openssl/ && \
cp -r out/rk3568/ohos_clang_arm64/thirdparty/openssl/* ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/aarch64-linux-ohos/ && \
cp -r out/sdk/sdk-native/os-specific/ohos/llvm/lib/aarch64-linux-ohos/lib* ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/aarch64-linux-ohos/ && \
cp -r third_party/openssl/include/openssl/macros.h ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/include/x86_64-linux-ohos/openssl/ && \
cp -r third_party/openssl/include/openssl/* ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/include/x86_64-linux-ohos/openssl/ && \
cp -r out/rk3568/ohos_clang_x86_64/thirdparty/openssl/* ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/x86_64-linux-ohos/ && \
cp -r out/sdk/sdk-native/os-specific/ohos/llvm/lib/x86_64-linux-ohos/lib* ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/x86_64-linux-ohos/ && \
cp -r out/sdk/obj/third_party/openssl/build_all_generated/include/openssl ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/include && \
cp -r third_party/openssl/include/openssl/* ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/include/openssl/ && \
cp -r out/rk3568/ohos_clang_x86_64/thirdparty/openssl/* ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/x86_64-linux-ohos/ && \
cp -r out/rk3568/ohos_clang_arm64/thirdparty/openssl/* ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/aarch64-linux-ohos/ && \
cp -r out/rk3568/ohos_clang_arm64/thirdparty/openssl/* ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/arm-linux-ohos/ && \
cp -r out/sdk/sdk-native/os-specific/linux/llvm/lib/aarch64-linux-ohos/* ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/aarch64-linux-ohos/ && \
cp -r out/sdk/sdk-native/os-specific/linux/llvm/lib/x86_64-linux-ohos/* ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/x86_64-linux-ohos/ && \
cp -r out/sdk/sdk-native/os-specific/ohos/llvm/lib/clang/15.0.4/lib/aarch64-linux-ohos/* ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/aarch64-linux-ohos/ && \
cp -r out/sdk/sdk-native/os-specific/ohos/llvm/lib/clang/15.0.4/lib/x86_64-linux-ohos/* ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/x86_64-linux-ohos/ && \
cp -r prebuilts/clang/ohos/linux-x86_64/llvm/lib/x86_64-linux-ohos/libc++.* ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/x86_64-linux-ohos/ && \
cp -r prebuilts/clang/ohos/linux-x86_64/llvm/lib/aarch64-linux-ohos/libc++.* ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/aarch64-linux-ohos/;

mkdir -p ${OHOS_ROOT}/prebuilts/clang/ohos;
# 如果 mac arm 平台，拷贝如下
cp -r prebuilts/clang/ohos/darwin-aarch64 ${OHOS_ROOT}/prebuilts/clang/ohos;
# 如果 mac x86_64平台，拷贝如下
cp -r prebuilts/clang/ohos/darwin-x86_64 ${OHOS_ROOT}/prebuilts/clang/ohos;
# 如果 linux 平台， 拷贝如下
cp -r prebuilts/clang/ohos/linux-x86_64 ${OHOS_ROOT}/prebuilts/clang/ohos;
```