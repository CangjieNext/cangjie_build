# Cangjie SDK Build Guide (Ubuntu 22.04)

## 1 Build Overview

### 1.1 Build Process Diagram

```mermaid
graph LR
    A[Prepare Environment] --> B[Compile Core<br>Dependencies]
    B --> C[Create<br>Workspace]
    C --> D[Fetch<br>Cangjie Source Code]
    D --> E[Compile Cangjie<br>Compiler and Debugger]
    E --> F[Compile<br>Cangjie Runtime]
    F --> G[Compile<br>Cangjie Standard Library]
    G --> H[Compile<br>STDX Extension Library]
    H --> I[Compile<br>Cangjie Toolset]
    I --> J[Organize and<br>Package Files]
    J --> K[Compile<br>Hello,Cangjie Program]
```

### 1.2 Key Considerations

1. **Environment Isolation**: All custom dependencies are installed in `/opt/buildtools` to avoid polluting system paths
2. **Memory Requirements**: Full build requires ≥8GB RAM, recommend adding 4GB swap space
3. **Network Requirements**: Initial build downloads ~3GB data, ensure stable network connection
4. **Windows Toolchain Support**: Requires compiling MinGW-w64 and supporting toolchain
5. **CPU Instruction Set**: **Must** set corresponding environment variables based on your computer's architecture (x86_64 or aarch64), and select appropriate system tools and docker images (uname -m)

## 2 Environment Preparation

### 2.1 System Requirements

- **OS**: Ubuntu 22.04 LTS
- **Disk Space**: ≥50GB
- **Memory**: ≥8GB (physical + swap)
- **User Permissions**: Regular user + sudo privileges

> You can set up the build environment yourself or use our provided Docker environment which includes all necessary system and build tools for Cangjie:
>
> ```bash
> docker pull swr.cn-north-4.myhuaweicloud.com/cj-docker/cangjie_ubuntu22_x86_kernel:v1.2
> # or
> docker pull swr.cn-north-4.myhuaweicloud.com/cj-docker/cangjie_ubuntu22_arm_kernel:v1.2
> ```

### 2.2 System Tools Installation

- Use Huawei mirror source (optional)

  ```bash
  # 1.Backup config file:
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

### 2.3 Compile MinGW-w64 and Supporting Toolchain (Critical Step)

```bash
# Create working directory
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
cd "$BUILD_ROOT/openssl-openssl-3.0.9";
mkdir build;
cd build;
../Configure mingw64 --prefix="${INSTALL_PATH}/x86_64-w64-mingw32" --cross-compile-prefix=x86_64-w64-mingw32- --libdir=lib;
make -j $tmp_cpus > build.log;
make install > install.log;
```


### 2.4 Set Environment Variables

Must configure these environment variables before building:

```bash
export MINGW_PATH=${BUILD_ROOT}/mingw-w64
export PATH=${MINGW_PATH}/bin:/usr/lib/llvm-15/bin:$PATH; # Adds mingw and clang-15 to PATH
# Following variables only affect filenames
export CANGJIE_VERSION=1.0.0
export STDX_VERSION=1
```

## 3 Source Code Preparation

### 3.1 Create Workspace

**<font color="red">\* Replace `/path/to/workspace` in the example below with your actual Cangjie SDK build directory</font>**

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

### 4.1 Compile Cangjie Compiler and Debugger

```bash
# Compile linux x64 compiler
cd ${WORKSPACE}/cangjie_compiler;
python3 build.py clean;
python3 build.py build -t release --no-tests;

# Compile windows compiler + cjdb
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
```

Verify installation:
```bash
source output/envsetup.sh;
cjc -v;
```

### 4.2 Compile Cangjie Runtime

```bash
cd ${WORKSPACE}/cangjie_runtime/runtime;
mkdir target;

# compile linux x64 runtime
python3 build.py clean;
python3 build.py build -t release -v ${CANGJIE_VERSION};
python3 build.py install;
cp -rf ${WORKSPACE}/cangjie_runtime/runtime/output/* target;
cp -rf ${WORKSPACE}/cangjie_runtime/runtime/output/common/linux_release_x86_64/{lib,runtime} ${WORKSPACE}/cangjie_compiler/output;

# compile windows x64 runtime
python3 build.py clean;
python3 build.py build -t release \
  --target windows-x86_64 \
  --target-toolchain ${MINGW_PATH}/bin \
  -v ${CANGJIE_VERSION};
python3 build.py install;
cp -rf ${WORKSPACE}/cangjie_runtime/runtime/output/* target;
cp -rf ${WORKSPACE}/cangjie_runtime/runtime/output/common/windows_release_x86_64/{lib,runtime} ${WORKSPACE}/cangjie_compiler/output;
cp -rf ${WORKSPACE}/cangjie_runtime/runtime/output/common/windows_release_x86_64/{lib,runtime} ${WORKSPACE}/cangjie_compiler/output-x86_64-w64-mingw32;
```

### 4.3 Compile Cangjie Standard Library

```bash
cd ${WORKSPACE}/cangjie_runtime/std;
# compile linux x64 std
python3 build.py clean;
python3 build.py build -t release \
  --target-lib=$WORKSPACE/cangjie_runtime/runtime/target \
  --target-lib=$OPENSSL_PATH;
python3 build.py install;
cp -rf ${WORKSPACE}/cangjie_runtime/std/output/* ${WORKSPACE}/cangjie_compiler/output/;

# compile windows x64 std
python3 build.py clean;
python3 build.py build -t release \
  --target windows-x86_64 \
  --target-lib=${WORKSPACE}/cangjie_runtime/runtime/target \
  --target-lib=${MINGW_PATH}/x86_64-w64-mingw32/lib \
  --target-sysroot ${MINGW_PATH}/ \
  --target-toolchain ${MINGW_PATH}/bin;
python3 build.py install;
cp -rf ${WORKSPACE}/cangjie_runtime/std/output/* ${WORKSPACE}/cangjie_compiler/output/;
cp -rf ${WORKSPACE}/cangjie_runtime/std/output/* ${WORKSPACE}/cangjie_compiler/output-x86_64-w64-mingw32/;
```

### 4.4 Compile STDX Extension Library

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

### 4.5 Build Toolset

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
   cjc hello.cj -o hello.exe && ./hello.exe
   # Should print:
   # Hello, Cangjie
   ```

🎉 Congratulations! You have successfully built and run the Cangjie SDK and executed the Hello, Cangjie program!