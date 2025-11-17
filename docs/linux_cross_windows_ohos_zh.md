# OHOS 仓颉SDK构建指导书 (Ubuntu 22.04)

## 1 构建概述

### 1.1 构建流程全景图

```mermaid
graph LR
    A[准备环境] --> B[编译核心<br>依赖]
    B --> C[创建<br>工作区]
    C --> D[获取<br>仓颉源码]
    D --> E[编译仓颉<br>编译器和调试器]
    E --> F[编译<br>仓颉运行时]
    F --> G[编译<br>仓颉标准库]
    G --> H[编译<br>STDX扩展库]
    H --> I[编译<br>仓颉工具集]
    I --> J[组织并<br>打包文件]
    J --> K[编译<br>Hello,Cangjie程序]
```

### 1.2 关键注意事项

1. **环境隔离**：所有自定义依赖安装在 `/opt/buildtools`，避免污染系统路径
2. **内存要求**：完整构建需要 ≥8GB 内存，建议添加 4GB 交换空间
3. **网络要求**：首次构建需下载约 3GB 数据，请确保稳定网络连接
4. **windows工具链支持**：需要编译MinGW-w64以及配套工具链
5. **芯片指令集**：x86_64

## 2 环境准备

### 2.1 系统要求

- **操作系统**: Ubuntu 22.04 LTS
- **磁盘空间**: ≥200GB
- **内存**: ≥8GB (物理内存+交换空间)
- **用户权限**: 普通用户 + sudo权限


> 您可以自己搭建相关构建环境，也可以基于我们提供的Docker环境进行仓颉构建，里面已内置所有构建仓颉所需的系统工具和构建工具：
>
> ```bash
> docker pull swr.cn-north-4.myhuaweicloud.com/cj-docker/cangjie_ubuntu18_x86_kernel:v2.8
> # 或
> docker pull swr.cn-north-4.myhuaweicloud.com/cj-docker/cangjie_ubuntu18_arm_kernel:v2.2
> ```

### 2.2 系统工具安装

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
  sudo apt update && sudo apt install -y \
    tar unzip wget curl libcurl4 expat openssl make gcc g++ gettext \
    nfs-common libtool sqlite3 zlib1g-dev libssl-dev cmake ninja-build\
    libcurl4-openssl-dev sudo autoconf build-essential rapidjson-dev \
    texinfo binutils expat libelf-dev libdwarf-dev openssh-client ssh \
    dos2unix libxext-dev libxtst-dev libxt-dev libcups2-dev clang clang-15 libedit-dev\
    libxrender-dev zip bzip2 libopenmpi-dev vim gdb lldb libclang-15-dev libgtest-dev\
    rpm patch libtinfo5 cpio rpm2cpio libncurses5 libncurses5-dev strace net-tools;
  ```

### 2.3 编译MinGW-w64及配套工具链 (关键步骤)

```bash
# 创建工作目录
export BUILD_ROOT=/opt/buildtools;
sudo mkdir -p $BUILD_ROOT;
sudo chown $USER:$USER $BUILD_ROOT;
export INSTALL_PATH=${BUILD_ROOT}/llvm-mingw-w64
export tmp_cpus=$(grep -w processor /proc/cpuinfo|wc -l);

cd $BUILD_ROOT;

# install mingw llvm toolchain
wget https://github.com/mstorsjo/llvm-mingw/archive/refs/tags/20220906.tar.gz -q --no-check-certificate;
tar xf 20220906.tar.gz;
cd llvm-mingw-20220906
git init llvm-project && cd llvm-project;
git remote add origin https://gitee.com/openharmony/third_party_llvm-project.git;
git fetch --depth 1 origin 5c68a1cb123161b54b72ce90e7975d95a8eaf2a4 && git checkout FETCH_HEAD;
cd .. && git clone https://gitee.com/openharmony/third_party_mingw-w64.git mingw-w64;
export TOOLCHAIN_ARCHS=x86_64
./build-llvm.sh ${INSTALL_PATH} --disable-lldb
./strip-llvm.sh ${INSTALL_PATH}
./install-wrappers.sh ${INSTALL_PATH}
./build-mingw-w64.sh ${INSTALL_PATH} --with-default-msvcrt=msvcrt
./build-mingw-w64-tools.sh ${INSTALL_PATH}
./build-compiler-rt.sh ${INSTALL_PATH}
./build-libcxx.sh ${INSTALL_PATH}
./build-mingw-w64-libraries.sh ${INSTALL_PATH}
cp ${INSTALL_PATH}/x86_64-w64-mingw32/lib/libmingwex.a ${INSTALL_PATH}/x86_64-w64-mingw32/lib/libssp.a
cp ${INSTALL_PATH}/x86_64-w64-mingw32/lib/libmingwex.a ${INSTALL_PATH}/x86_64-w64-mingw32/lib/libssp_nonshared.a

# build openssl
wget https://github.com/openssl/openssl/archive/refs/tags/openssl-3.0.9.tar.gz -q --no-check-certificate;
tar xf openssl-3.0.9.tar.gz;
cd "$src_root/openssl-openssl-3.0.9";
mkdir build;
cd build;
../Configure mingw64 --prefix="${INSTALL_PATH}/x86_64-w64-mingw32" --cross-compile-prefix=x86_64-w64-mingw32- --libdir=lib;
make -j $tmp_cpus > build.log;
make install > install.log;
```

### 2.4. 编译ohos-x86_64、ohos-aarch64工具链
[请参考编译ohos-x86_64、ohos-aarch64工具链](linux_ohos_toolchain.md)

### 2.5. 设置环境变量

请务必在构建前配置以下环境变量：

```bash
export MINGW_PATH=${BUILD_ROOT}/llvm-mingw-w64
export PATH=${MINGW_PATH}/bin:/usr/lib/llvm-15/bin:$PATH; # 用于将mingw、clang-15加入环境变量
# 以下变量仅影响文件名
export CANGJIE_VERSION=1.0.0
export STDX_VERSION=1
```

## 3 源码准备

### 3.1 创建工作区

**<font color="red">\* 下方示例的`/path/to/workspace`需更换为您构建仓颉SDK的工作目录</font>**

```bash
export WORKSPACE=/path/to/workspace
mkdir -p $WORKSPACE;
cd $WORKSPACE;
```

### 3.2 获取仓颉源码

```bash
git clone https://gitcode.com/Cangjie/cangjie_compiler.git;
git clone https://gitcode.com/Cangjie/cangjie_runtime.git;
git clone https://gitcode.com/Cangjie/cangjie_tools.git;
git clone https://gitcode.com/Cangjie/cangjie_stdx.git;
```

## 4 编译流程

### 4.1 编译仓颉编译器和调试器

```bash
# 编译linux x64 host
cd ${WORKSPACE}/cangjie_compiler;
python3 build.py clean;
python3 build.py build -t release --no-tests;

# 编译windows sdk
export CMAKE_PREFIX_PATH=${MINGW_PATH}/x86_64-w64-mingw32;
python3 build.py build -t release \
  --product cjc \
  --no-tests \
  --target windows-x86_64 \
  --target-sysroot ${MINGW_PATH}/ \
  --target-toolchain ${MINGW_PATH}/bin \
  --build-cjdb;
python3 build.py build -t release \
  --product libs \
  --target windows-x86_64 \
  --target-sysroot ${MINGW_PATH}/ \
  --target-toolchain ${MINGW_PATH}/bin;

python3 build.py install --host windows-x86_64;
python3 build.py install;
cp -rf output-x86_64-w64-mingw32/* output;
export OHOS_ROOT=/opt/buildtools/ohos_root;
python3 build.py build -t release \
  --product libs \
  --target ohos-x86_64  \
  --target-toolchain ${OHOS_ROOT}/prebuilts/clang/ohos/${kernel}-${cmake_arch}/llvm/bin \
  --target-sysroot ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot;
python3 build.py install;
python3 build.py build -t release \
  --product libs \
  --target ohos-aarch64  \
  --target-toolchain ${OHOS_ROOT}/prebuilts/clang/ohos/${kernel}-${cmake_arch}/llvm/bin \
  --target-sysroot ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot;
python3 build.py install;
```

验证安装：
```bash
source output/envsetup.sh;
cjc -v;
```

### 4.2 编译仓颉运行时

```bash
cd ${WORKSPACE}/cangjie_runtime/runtime;
python3 build.py clean;
python3 build.py build -t release \
  --target windows-x86_64 \
  --target-toolchain ${MINGW_PATH}/bin \
  -v ${CANGJIE_VERSION};
python3 build.py install;
cp -rf ${WORKSPACE}/cangjie_runtime/runtime/output/common/windows_release_x86_64/{lib,runtime} ${WORKSPACE}/cangjie_compiler/output;
cp -rf ${WORKSPACE}/cangjie_runtime/runtime/output/common/windows_release_x86_64/{lib,runtime} ${WORKSPACE}/cangjie_compiler/output-x86_64-w64-mingw32;
python3 build.py build -t release \
  --target ohos-x86_64 \
  --target-toolchain ${OHOS_ROOT} \
  -v ${cangjie_version};
python3 build.py install;
cp -rf ${WORKSPACE}/cangjie_runtime/runtime/output/common/linux_ohos_release_x86_64/{lib,runtime} ${WORKSPACE}/cangjie_compiler/output;
python3 build.py build -t release \
  --target ohos-aarch64 \
  --target-toolchain ${OHOS_ROOT} \
  -v ${cangjie_version};
python3 build.py install;
cp -rf ${WORKSPACE}/cangjie_runtime/runtime/output/common/linux_ohos_release_aarch64/{lib,runtime} ${WORKSPACE}/cangjie_compiler/output;

```

### 4.3 编译仓颉标准库

```bash
cd ${WORKSPACE}/cangjie_runtime/std;
python3 build.py clean;
python3 build.py build -t release \
  --target windows-x86_64 \
  --target-lib=${WORKSPACE}/cangjie_runtime/runtime/output \
  --target-lib=${MINGW_PATH}/x86_64-w64-mingw32/lib \
  --target-sysroot ${MINGW_PATH}/ \
  --target-toolchain ${MINGW_PATH}/bin;
python3 build.py build -t release \
    --target ohos-x86_64 \
    --target-lib=${WORKSPACE}/cangjie_runtime/runtime/output \
    --target-lib=${WORKSPACE}/cangjie_runtime/runtime/output/common \
    --target-toolchain ${OHOS_ROOT}/prebuilts/clang/ohos/${kernel}-${cmake_arch}/llvm/bin \
    --target-sysroot ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot;
python3 build.py build -t release \
    --target ohos-aarch64 \
    --target-lib=${WORKSPACE}/cangjie_runtime/runtime/output \
    --target-lib=${WORKSPACE}/cangjie_runtime/runtime/output/common \
    --target-toolchain ${OHOS_ROOT}/prebuilts/clang/ohos/${kernel}-${cmake_arch}/llvm/bin \
    --target-sysroot ${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot;
python3 build.py install;
cp -rf ${WORKSPACE}/cangjie_runtime/std/output/* ${WORKSPACE}/cangjie_compiler/output/;
cp -rf ${WORKSPACE}/cangjie_runtime/std/output/* ${WORKSPACE}/cangjie_compiler/output-x86_64-w64-mingw32/;
```

### 4.4 编译 STDX 扩展库

```bash
cd ${WORKSPACE}/cangjie_stdx;
python3 build.py clean;
python3 build.py build -t release \
	--include=$WORKSPACE/cangjie_compiler/include \
    --target-lib=${MINGW_PATH}/x86_64-w64-mingw32/lib \
	--target windows-x86_64 \
    --target-sysroot ${MINGW_PATH}/ \
    --target-toolchain ${MINGW_PATH}/bin;
python3 build.py install;
export CANGJIE_STDX_PATH=${WORKSPACE}/cangjie_stdx/target/windows_x86_64_cjnative/static/stdx;
```

### 4.5 编译工具集

#### (1) cjpm

```bash
cd ${WORKSPACE}/cangjie_tools/cjpm/build;
python3 build.py clean;
python3 build.py build -t release --target windows-x86_64;
python3 build.py install;
```

#### (2) cjfmt

```bash
cd ${WORKSPACE}/cangjie_tools/cjfmt/build;
python3 build.py clean;
python3 build.py build -t release --target windows-x86_64;
python3 build.py install;
```

#### (3) HyperLangExtension

```bash
cd ${WORKSPACE}/cangjie_tools/hyperlangExtension/build;
python3 build.py clean;
python3 build.py build -t release --target windows-x86_64;
python3 build.py install;
```

#### (4) LSP Server

```bash
cd ${WORKSPACE}/cangjie_tools/cangjie-language-server/build;
python3 build.py clean;
python3 build.py build -t release --target windows-x86_64;
python3 build.py install;
```

## 5 打包发布

### 5.1 组织文件

```bash
# 清空历史构建
mkdir -p $WORKSPACE/software;
rm -rf $WORKSPACE/software/*;
cd $WORKSPACE/software;

# 拷贝cangjie目录
cp -R $WORKSPACE/cangjie_compiler/output-x86_64-w64-mingw32 cangjie;

# 删除 ast-support.a
rm -rf cangjie/lib/windows_x86_64_cjnative/libcangjie-ast-support.a

# 组织文件
cp $WORKSPACE/cangjie_tools/cjpm/dist/cjpm.exe cangjie/tools/bin;
mkdir -p cangjie/tools/config;
cp $WORKSPACE/cangjie_tools/cjfmt/build/build/bin/cjfmt.exe cangjie/tools/bin;
cp $WORKSPACE/cangjie_tools/cjfmt/config/*.toml cangjie/tools/config;
cp $WORKSPACE/cangjie_tools/hyperlangExtension/target/bin/main.exe cangjie/tools/bin/hle.exe;
cp -r $WORKSPACE/cangjie_tools/hyperlangExtension/src/dtsparser cangjie/tools;
rm -rf cangjie/tools/dtsparser/*.cj;
cp $WORKSPACE/cangjie_tools/cangjie-language-server/output/bin/LSPServer.exe cangjie/tools/bin;

# 打包和设置权限
chmod -R 750 cangjie
zip -qr cangjie-sdk-windows-x64-${CANGJIE_VERSION}.zip cangjie;
chmod 550 cangjie-sdk-windows-x64-${CANGJIE_VERSION}.zip;
```

### 5.2 组织并打包 STDX

```bash
cd $WORKSPACE/software;
# 拷贝stdx目录
cp -r $WORKSPACE/cangjie_stdx/target/windows_x86_64_cjnative ./;
chmod -R 750 windows_x86_64_cjnative;
zip -qr cangjie-stdx-windows-x64-${CANGJIE_VERSION}.${STDX_VERSION}.zip windows_x86_64_cjnative;
chmod 550 cangjie-stdx-windows-x64-${CANGJIE_VERSION}.${STDX_VERSION}.zip;
```

## 6 编译Hello, Cangjie程序

检查生成的文件：

```bash
ls -lh $WORKSPACE/software
# 应包含：
# - cangjie-sdk-windows-x64-${CANGJIE_VERSION}.zip
# - cangjie-stdx-windows-x64-${CANGJIE_VERSION}.${STDX_VERSION}.zip
```

验证hello cangjie程序（请在windows执行以下命令）：

1. 安装仓颉SDK

   - 若使用 Windows 命令提示符（CMD）环境，请执行：

     ```bash
     path\to\cangjie\envsetup.bat
     ```

   - 若使用 PowerShell 环境，请执行：

     ```bash
     . path\to\cangjie\envsetup.ps1
     ```

   - 若使用 MSYS shell、bash 等环境，请执行：

     ```bash
     source path/to/cangjie/envsetup.sh
     ```

2. 创建并运行cj文件：

```bash
   echo "main() { println(\"Hello, Cangjie\") }" > hello.cj
   # 手机平台
   cjc hello.cj --target=aarch64-linux-ohos -B${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/aarch64-linux-ohos -L${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/aarch64-linux-ohos
   # 模拟器平台
   cjc hello.cj --target=x86_64-linux-ohos -B${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/x86_64-linux-ohos -L${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/x86_64-linux-ohos
   # 应打印：
   # Hello, Cangjie
   ```

🎉 恭喜您成功构建了仓颉SDK并运行了Hello, Cangjie程序！
   
