# OHOS Cangjie SDK Build Guide (Ubuntu 22.04)

## 1 Build Overview

### 1.1 Build Process Flowchart

```mermaid
graph LR
    A[Prepare Environment] --> B[Build Core<br>Dependencies]
    B --> C[Create<br>Workspace]
    C --> D[Fetch<br>Cangjie Source Code]
    D --> E[Build Cangjie<br>Compiler and Debugger]
    E --> F[Build<br>Cangjie Runtime]
    F --> G[Build<br>Cangjie Standard Library]
    G --> H[Build<br>STDX Extension Library]
    H --> I[Build<br>Cangjie Toolset]
    I --> J[Organize and<br>Package Files]
    J --> K[Build<br>Hello,Cangjie Program]
```

### 1.2 Key Considerations

1. **Environment Isolation**: All custom dependencies installed in `/opt/buildtools` to avoid polluting system paths
2. **Memory Requirements**: Full build requires ≥8GB RAM, recommend adding 4GB swap space
3. **Network Requirements**: Initial build downloads ~3GB data, ensure stable connection
4. **Windows Toolchain Support**: Requires MinGW-w64 and supporting toolchain compilation
5. **CPU Instruction Set**: x86_64

## 2 Environment Preparation

### 2.1 System Requirements

- **OS**: Ubuntu 22.04 LTS
- **Disk Space**: ≥200GB
- **Memory**: ≥8GB (physical + swap)
- **User Permissions**: Regular user + sudo privileges

> You can set up the build environment yourself or use our provided Docker environment which includes all necessary tools:
>
> ```bash
> docker pull swr.cn-north-4.myhuaweicloud.com/cj-docker/cangjie_ubuntu22_x86_kernel:v1.2
> # or
> docker pull swr.cn-north-4.myhuaweicloud.com/cj-docker/cangjie_ubuntu22_arm_kernel:v1.2
> ```

### 2.2 System Tools Installation

- Use Huawei mirror (optional)

  ```bash
  # 1.Backup config:
  sudo cp -a /etc/apt/sources.list /etc/apt/sources.list.bak;
  # 2.Modify sources.list, replace http://archive.ubuntu.com and http://security.ubuntu.com with http://mirrors.huaweicloud.com:
  sudo sed -i "s@http://.*archive.ubuntu.com@http://mirrors.huaweicloud.com@g" /etc/apt/sources.list;
  sudo sed -i "s@http://.*security.ubuntu.com@http://mirrors.huaweicloud.com@g" /etc/apt/sources.list;
  # 3.Update index
  sudo apt-get update;
  ```

- Install system tools

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

### 2.3 Build MinGW-w64 Toolchain (Critical Step)

```bash
# Create working directory
export BUILD_ROOT=/opt/buildtools;
sudo mkdir -p $BUILD_ROOT;
sudo chown $USER:$USER $BUILD_ROOT;
cd $BUILD_ROOT;
tmp_cpus=$(grep -w processor /proc/cpuinfo|wc -l);
wget http://gcc.gnu.org/pub/gcc/infrastructure/gmp-6.2.1.tar.bz2 --no-check-certificate;
wget http://gcc.gnu.org/pub/gcc/infrastructure/mpfr-4.1.0.tar.bz2 --no-check-certificate;
wget http://gcc.gnu.org/pub/gcc/infrastructure/mpc-1.2.1.tar.gz --no-check-certificate;
wget http://gcc.gnu.org/pub/gcc/infrastructure/isl-0.24.tar.bz2 --no-check-certificate;
wget https://gcc.gnu.org/pub/gcc/releases/gcc-10.3.0/gcc-10.3.0.tar.gz --no-check-certificate;
wget https://gcc.gnu.org/pub/binutils/releases/binutils-2.41.tar.gz --no-check-certificate;
wget https://github.com/mingw-w64/mingw-w64/archive/refs/tags/v11.0.1.tar.gz --no-check-certificate;
wget https://github.com/libffi/libffi/releases/download/v3.4.2/libffi-3.4.2.tar.gz --no-check-certificate;
wget https://ftp.gnu.org/gnu/texinfo/texinfo-6.8.tar.gz --no-check-certificate;
wget https://github.com/openssl/openssl/archive/refs/tags/openssl-3.0.9.tar.gz -q --no-check-certificate;

tar xf gmp-6.2.1.tar.bz2;
tar xf mpfr-4.1.0.tar.bz2;
tar xf mpc-1.2.1.tar.gz;
tar xf isl-0.24.tar.bz2;
tar xf gcc-10.3.0.tar.gz;
tar xf binutils-2.41.tar.gz;
tar xf v11.0.1.tar.gz;
tar xf libffi-3.4.2.tar.gz;
tar xf texinfo-6.8.tar.gz;
tar xf openssl-3.0.9.tar.gz;

cp gmp-6.2.1.tar.bz2 mpfr-4.1.0.tar.bz2 mpc-1.2.1.tar.gz isl-0.24.tar.bz2 gcc-10.3.0;
export install_dir=$BUILD_ROOT/mingw-w64;
mkdir -p $install_dir;

# prepare build envs
src_root=$(pwd);
tmp_cpus=$(grep -w processor /proc/cpuinfo|wc -l);
CFLAG_BASE="-O2 -pipe -fno-ident";
CXXFLAGS_BASE="-O2 -pipe -fno-ident -std=c++17";
export PATH="$install_dir/bin:$PATH";
export CFLAGS="$CFLAGS_BASE -Wl,-z,now -fPIE -pie -fstack-protector-all -s";
export CXXFLAGS="$CXXFLAGS_BASE -Wl,-z,now -fPIE pie -fstack-protector-all -s";
export LDFLAGS="-pipe -fno-ident";

# build texinfo
cd "$src_root/texinfo-6.8";
mkdir -p build;
cd build;
../configure --prefix="$src_root/texinfo";
make -j $tmp_cpus > build.log;
make install > install.log;
export PATH="$src_root/texinfo/bin:$PATH";

# build binutils
cd "$src_root/binutils-2.41";
mkdir build;
cd build;
../configure --prefix="$install_dir" --disable-shared --enable-static --with-sysroot="$install_dir" --target=x86_64-w64-mingw32 --disable-nls --enable-lto --disable-gdb;

make -j $tmp_cpus > build.log;
make install > install.log;

# build mingw-w64-headers
export CFLAGS="$CFLAG_BASE";
export CXXFLAGS="$CXXFLAG_BASE";
cd "$src_root/mingw-w64-11.0.1";
mkdir header-build;
cd header-build;
../mingw-w64-headers/configure --build=x86_64-pc-linux-gnu --host=x86_64-w64-mingw32 --prefix="$install_dir/x86_64-w64-mingw32" --with-default-msvcrt=msvcrt;
make install;

# build gcc
cd "$src_root/gcc-10.3.0";
sed -i 's/gmp-6.1.0.tar.bz2/gmp-6.2.1.tar.bz2/g' ./contrib/download_prerequisites;
sed -i 's/mpfr-3.1.4.tar.bz2/mpfr-4.1.0.tar.bz2/g' ./contrib/download_prerequisites;
sed -i 's/mpc-1.0.3.tar.gz/mpc-1.2.1.tar.gz/g' ./contrib/download_prerequisites;
sed -i 's/isl-0.18.tar.bz2/isl-0.24.tar.bz2/g' ./contrib/download_prerequisites;
./contrib/download_prerequisites --no-verify;
sed -i 's/$(CFLAGS)/-g -O2 -fstack-protector-strong/g' libstdc++-v3/Makefile.in;
sed -i 's/$(CXXFLAGS)/-g -O2 -fstack-protector-strong/g' libstdc++-v3/Makefile.in;
sed -i 's/-fno-stack-protector/-fstack-protector-strong/g' libgcc/Makefile.in;
mkdir build;
cd build;
../configure --target=x86_64-w64-mingw32 --disable-multilib --prefix="$install_dir" --enable-languages=c,c++ --disable-nls --enable-threads=posix --enable-lib64 --disable-lib32;
make -j $tmp_cpus all-gcc > build.log;
make install-gcc > install.log;

# build mingw-w64-crt
cd "$src_root/mingw-w64-11.0.1";
mkdir crt-build;
cd crt-build;
../mingw-w64-crt/configure --build=x86_64-pc-linux-gnu --host=x86_64-w64-mingw32 --prefix="$install_dir/x86_64-w64-mingw32" --with-default-msvcrt=msvcrt --with-sysroot="$install_dir/x86_64-w64-mingw32" --enable-lib64 --disable-lib32 --enable-wildcard;
make -j $tmp_cpus > build.log;
make install > install.log;

# build winpthreads
cd "$src_root/mingw-w64-11.0.1";
mkdir -p winpthreads-build;
cd winpthreads-build;
../mingw-w64-libraries/winpthreads/configure --build=x86_64-pc-linux-gnu --host=x86_64-w64-mingw32 --prefix="$install_dir/x86_64-w64-mingw32";
make -j $tmp_cpus > build.log;
make install > install.log;

# finish build gcc
cd "$src_root/gcc-10.3.0/build";
make -j $tmp_cpus > build.log;
make install > install.log;
touch "$install_dir/lib/gcc/x86_64-w64-mingw32/10.3.0/libgcc_eh.a";

# building windows-native binutils
cd "$src_root/binutils-2.41";
mkdir -p native-build;
cd native-build;
../configure --prefix="$install_dir/x86_64-w64-mingw32" --disable-shared --enable-static --host=x86_64-w64-mingw32 --target=x86_64-w64-mingw32 --disable-multilib --disable-nls --enable-lto --disable-gdb --enable-64-bit-bfd --enable-plugins --enable-gold --disable-rpath;
make -j $tmp_cpus > build.log;
make install > install.log;

# build openssl
cd "$src_root/openssl-openssl-3.0.9";
mkdir build;
cd build;
../Configure mingw64 --prefix="$install_dir/x86_64-w64-mingw32" --cross-compile-prefix=x86_64-w64-mingw32- --libdir=lib;
make -j $tmp_cpus > build.log;
make install > install.log;
```

### 2.4. Build ohos-x86_64 and ohos-aarch64 Toolchains
[Refer to building ohos-x86_64 and ohos-aarch64 toolchains](linux_ohos_toolchain.md)

### 2.5. Set Environment Variables

Configure these variables before building:

```bash
export MINGW_PATH=${BUILD_ROOT}/mingw-w64
export PATH=${MINGW_PATH}/bin:/usr/lib/llvm-15/bin:$PATH; # Add mingw/clang-15 to PATH
# Following variables only affect filenames
export CANGJIE_VERSION=1.0.0
export STDX_VERSION=1
```

## 3 Source Code Preparation

### 3.1 Create Workspace

**<font color="red">\* Replace `/path/to/workspace` with your actual Cangjie SDK build directory</font>**

```bash
export WORKSPACE=/path/to/workspace
mkdir -p $WORKSPACE;
cd $WORKSPACE;
```

### 3.2 Fetch Cangjie Source Code

```bash
git clone https://gitcode.com/Cangjie/cangjie_compiler.git;
git clone https://gitcode.com/Cangjie/cangjie_runtime.git;
git clone https://gitcode.com/Cangjie/cangjie_tools.git;
git clone https://gitcode.com/Cangjie/cangjie_stdx.git;
```

## 4 Build Process

### 4.1 Build Cangjie Compiler and Debugger

```bash
# Build linux x64 host
cd ${WORKSPACE}/cangjie_compiler;
python3 build.py clean;
python3 build.py build -t release --no-tests;

# Build windows sdk
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

Verify installation:
```bash
source output/envsetup.sh;
cjc -v;
```

### 4.2 Build Cangjie Runtime

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
  --target-toolchain ${OHOS_ROOT} \```markdown
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

### 4.3 Compiling the Cangjie Standard Library

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

### 4.4 Compiling the STDX Extension Library

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

### 4.5 Compiling Toolset

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

## 5 Packaging and Release

### 5.1 Organizing Files

```bash
# Clear previous builds
mkdir -p $WORKSPACE/software;
rm -rf $WORKSPACE/software/*;
cd $WORKSPACE/software;

# Copy cangjie directory
cp -R $WORKSPACE/cangjie_compiler/output-x86_64-w64-mingw32 cangjie;

# Remove ast-support.a
rm -rf cangjie/lib/windows_x86_64_cjnative/libcangjie-ast-support.a

# Organize files
cp $WORKSPACE/cangjie_tools/cjpm/dist/cjpm.exe cangjie/tools/bin;
mkdir -p cangjie/tools/config;
cp $WORKSPACE/cangjie_tools/cjfmt/build/build/bin/cjfmt.exe cangjie/tools/bin;
cp $WORKSPACE/cangjie_tools/cjfmt/config/*.toml cangjie/tools/config;
cp $WORKSPACE/cangjie_tools/hyperlangExtension/target/bin/main.exe cangjie/tools/bin/hle.exe;
cp -r $WORKSPACE/cangjie_tools/hyperlangExtension/src/dtsparser cangjie/tools;
rm -rf cangjie/tools/dtsparser/*.cj;
cp $WORKSPACE/cangjie_tools/cangjie-language-server/output/bin/LSPServer.exe cangjie/tools/bin;

# Package and set permissions
chmod -R 750 cangjie
zip -qr cangjie-sdk-windows-x64-${CANGJIE_VERSION}.zip cangjie;
chmod 550 cangjie-sdk-windows-x64-${CANGJIE_VERSION}.zip;
```

### 5.2 Organizing and Packaging STDX

```bash
cd $WORKSPACE/software;
# Copy stdx directory
cp -r $WORKSPACE/cangjie_stdx/target/windows_x86_64_cjnative ./;
chmod -R 750 windows_x86_64_cjnative;
zip -qr cangjie-stdx-windows-x64-${CANGJIE_VERSION}.${STDX_VERSION}.zip windows_x86_64_cjnative;
chmod 550 cangjie-stdx-windows-x64-${CANGJIE_VERSION}.${STDX_VERSION}.zip;
```

## 6 Compiling the Hello, Cangjie Program

Check generated files:

```bash
ls -lh $WORKSPACE/software
# Should include:
# - cangjie-sdk-windows-x64-${CANGJIE_VERSION}.zip
# - cangjie-stdx-windows-x64-${CANGJIE_VERSION}.${STDX_VERSION}.zip
```

Verify the hello cangjie program (execute the following commands on Windows):

1. Install Cangjie SDK

   - For Windows Command Prompt (CMD) environment, execute:

     ```bash
     path\to\cangjie\envsetup.bat
     ```

   - For PowerShell environment, execute:

     ```bash
     . path\to\cangjie\envsetup.ps1
     ```

   - For MSYS shell, bash, etc., execute:

     ```bash
     source path/to/cangjie/envsetup.sh
     ```

2. Create and run cj file:

```bash
   echo "main() { println(\"Hello, Cangjie\") }" > hello.cj
   # Mobile platform
   cjc hello.cj --target=aarch64-linux-ohos -B${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/aarch64-linux-ohos -L${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/aarch64-linux-ohos
   # Emulator platform
   cjc hello.cj --target=x86_64-linux-ohos -B${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/x86_64-linux-ohos -L${OHOS_ROOT}/out/sdk/obj/third_party/musl/sysroot/usr/lib/x86_64-linux-ohos
   # Should print:
   # Hello, Cangjie
   ```

🎉 Congratulations! You have successfully built and run the Cangjie SDK and executed the Hello, Cangjie program!
```