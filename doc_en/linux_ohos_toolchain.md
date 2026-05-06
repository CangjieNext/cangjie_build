# ohos-x86_64, ohos-aarch64 Toolchain

Note: This tutorial will build the host-platform-independent part of the OHOS SDK toolchain, which will be packaged into their respective complete SDKs in [Linux OHOS Build Guide](linux_ohos.md), [Linux -> Windows (OHOS) Cross Build Guide](linux_cross_windows_ohos.md), and [macOS OHOS Build Guide](macos_ohos.md).

## 1 Environment Preparation

### 1.1 System Requirements

- **Operating System**: Ubuntu 22.04 LTS as an example, other versions can also refer to this process

- **Disk Space**: ≥200GB
- **Memory**: ≥8GB (physical memory + swap space)
- **User Permissions**: Regular user + sudo privileges
- **CPU Architecture**: x86_64

### 1.2 System Tools Installation

- Using Huawei Mirror Source (Optional)

```bash
  # 1. Backup configuration file:
  sudo cp -a /etc/apt/sources.list /etc/apt/sources.list.bak;
  # 2. Modify sources.list file, replace http://archive.ubuntu.com and http://security.ubuntu.com with http://mirrors.huaweicloud.com, you can refer to the following command:
  sudo sed -i "s@http://.*archive.ubuntu.com@http://mirrors.huaweicloud.com@g" /etc/apt/sources.list;
  sudo sed -i "s@http://.*security.ubuntu.com@http://mirrors.huaweicloud.com@g" /etc/apt/sources.list;
  # 3. Update index
  sudo apt-get update;
```

- Install System Tools

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

## 2 Build ohos_aarch64, ohos_x86-64 Toolchain

### 2.1 Create Workspace

**<font color="red">\* Replace `/path/to/workspace` in the example below with your workspace directory for building OHOS</font>**

```bash
export WORKSPACE=/path/to/workspace
mkdir -p $WORKSPACE;
cd $WORKSPACE;
# When executing prebuilts_download.sh, it will create a directory for OHOS-related binaries in the parent directory, so create a source directory here to store OHOS source code
mkdir source;
```

### 2.2 Download repo
```bash
cd $WORKSPACE;
curl https://raw.gitcode.com/gitcode-dev/repo/raw/main/repo-py3 -o repo;
chmod a+x repo;
pip3 install -i https://repo.huaweicloud.com/repository/pypi/simple requests;
```

### 2.3 Download OHOS Source Code

```bash
cd ${WORKSPACE}/source;
# Please configure git user.name and user.email before executing the command
# If you get a python not found error, change `#!/usr/bin/env python` to `#!/usr/bin/env python3`
${WORKSPACE}/repo init -u https://gitcode.com/openharmony/manifest.git -b master --no-repo-verify;
${WORKSPACE}/repo sync -c;
```

### 2.4 Cross-compile OHOS musl

```bash
# vim third_party/musl/BUILD.gn, add the following code at the end
group("all_musls") {
  deps = [
    ":soft_libc_musl_shared(//build/toolchain/ohos:ohos_clang_arm64)",
    ":soft_libc_musl_shared(//build/toolchain/ohos:ohos_clang_x86_64)",
  ]
}
```
```bash
# Execute musl build
cd ${WORKSPACE}/source;
./build/prebuilts_download.sh;
./build.sh --product-name rk3568 --build-target all_musls;
```

### 2.5 Cross-compile OHOS openssl

```bash
# vim third_party/openssl/BUILD.gn, add the following code at the end
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
# Execute build
cd ${WORKSPACE}/source;
./build.sh --product-name rk3568 --build-target all_ssl;
```

### 2.6 Build Toolchain Required for ohos-sdk
```bash
cd ${WORKSPACE}/source;
./build.sh --product-name ohos-sdk --ccache;
```

### 2.7 Copy Toolchain
```bash
# OHOS_ROOT environment variable specifies the path you use to save the OHOS toolchain
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
# If mac arm platform, copy as follows
cp -r prebuilts/clang/ohos/darwin-aarch64 ${OHOS_ROOT}/prebuilts/clang/ohos;
# If mac x86_64 platform, copy as follows
cp -r prebuilts/clang/ohos/darwin-x86_64 ${OHOS_ROOT}/prebuilts/clang/ohos;
# If linux platform, copy as follows
cp -r prebuilts/clang/ohos/linux-x86_64 ${OHOS_ROOT}/prebuilts/clang/ohos;
```

