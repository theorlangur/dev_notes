# GCC 16.1 Cross-Compiler for Zephyr with Picolibc and libstdc++

**Target:** ARM Cortex-M33 (armv8-m.main, fpv5-sp-d16, hard-float)  
**Toolchain:** GCC 16.1 + Binutils 2.42 + Picolibc 1.8.8 + libstdc++  

This document describes the complete process of building GCC 16.1 as a cross-compiler for Zephyr OS on nRF54L15 (Cortex-M33), including C and C++ support with picolibc as the C library and libstdc++ for C++ standard library.

---

## Table of Contents
1. [Architecture & Target Configuration](#1-architecture--target-configuration)
2. [Build Environment Setup](#2-build-environment-setup)
3. [Build Sequence](#3-build-sequence)
   - [Pass 1: Binutils 2.42](#pass-1-binutils-242)
   - [Pass 2: GCC Standalone (C + C++, no C library)](#pass-2-gcc-standalone-c--c-no-c-library)
   - [Pass 3: Picolibc 1.8.8](#pass-3-picolibc-188)
   - [Pass 4: Full GCC with Picolibc](#pass-4-full-gcc-with-picolibc)
   - [Pass 5: GDB 16.1](#pass-5-gdb-161)
4. [Zephyr Integration](#4-zephyr-integration)
   - [Toolchain File](#toolchain-file)
   - [Environment Setup](#environment-setup)
   - [Building Zephyr Projects](#building-zephyr-projects)
5. [C++ Standard Library Configuration](#5-c-standard-library-configuration)
6. [Directory Layout](#6-directory-layout)
7. [Key Changes & Workarounds](#7-key-changes--workarounds)
8. [Known Issues & Troubleshooting](#8-known-issues--troubleshooting)
9. [Verification Checklist](#9-verification-checklist)
10. [Usage Examples](#10-usage-examples)

---

## 1. Architecture & Target Configuration

### Cortex-M33 Specifications
- **Architecture:** ARMv8-M with Main Extension (armv8-m.main)
- **Instruction Set:** 32-bit Thumb / Thumb-2 subset
- **FPU:** IEEE 754 single-precision, FPv5 (fpv5-sp-d16), 16 D-registers
- **Float ABI:** Hard-float (required for Cortex-M33 FPU)

### Toolchain Flags
```
--with-arch=armv8-m.main       # Binutils + GCC configure
--with-multilib-list=rmprofile # M-profile multi-lib variants (critical!)
-mcpu=cortex-m33               # GCC compile time
-mfpu=fpv5-sp-d16              # Enable FPU code generation
-mfloat-abi=hard               # Use FPU for float/double arguments
```

**Important:** `armv8-m.main` does NOT imply FPU. You MUST add `-mfpu=fpv5-sp-d16` to enable FPU code generation.

---

## 2. Build Environment Setup

### Directory Layout
```
$ROOT   = ~/gcc/16.1/
$BD     = $ROOT/gcc16-build-arm/           (build directory root)
$INST   = $BD/install/opt/gcc16-arm-zephyr-eabi/  (install prefix)
$GCC_SRC = $ROOT/gcc-16.1.0/               (GCC source, already on disk)
$BIN_SRC = $BD/binutils-2.42/              (binutils source, already on disk)
$PIC_SRC = ~/zephyr_workspace/modules/lib/picolibc/  (picolibc source)
```

You need to have a working NCS toolchain as the required environment file will be created there and will references pieces from it.
In this guide it's assumed to be located at `~/ncs/toolchains/7cbc0036f4` (ver 3.0.2)

You need to have an installed NCS (or an add-on like ncs-zigbee). In this guide it's assumed to be located at:
`~/zephyr_workspace/` (with `zephyr` being correspondingly at `~/zephyr_workspace/zephyr`, various modules at `~/zephyr_workspace/modules`)

### Prerequisites
```bash
sudo apt install build-essential bzip2 xz-utils wget tar \
    cmake meson ninja-build \
    libgmp-dev libmpfr-dev libmpc-dev libisl-dev zlib1g-dev
```

### Source Files
- GCC 16.1: `gcc-16.1.0.tar.xz` (extracted to `$ROOT/gcc-16.1.0/`)
- Binutils 2.42: `binutils-2.42.tar.xz` (extracted to `$BD/binutils-2.42/`)
- Picolibc 1.8.8: Zephyr module at `$PIC_SRC/`

---

## 3. Build Sequence

### Pass 1: Binutils 2.42

Build the assembler and linker for the target.

```bash
cd $BD
mkdir -p b1-binutils && cd b1-binutils

../binutils-2.42/configure \
    --target=arm-zephyr-eabi \
    --prefix=$INST \
    --with-arch=armv8-m.main \
    --disable-nls \
    --disable-shared \
    --disable-libgomp \
    --disable-libatomic \
    --disable-libquadmath \
    --disable-libssp

make -j$(nproc)
make install
```

**Verify:**
```bash
$INST/bin/arm-zephyr-eabi-as --version
# Should show: GNU assembler (GNU Binutils) 2.42
```

---

### Pass 2: GCC Standalone (C + C++, no C library)

Build the compiler driver and libgcc without a C library. This pass builds libgcc with multi-lib support for M-profile variants.

```bash
cd $BD
mkdir -p b2-gcc && cd b2-gcc

PATH=$INST/bin:$PATH \
$GCC_SRC/configure \
    --target=arm-zephyr-eabi \
    --prefix=$INST \
    --without-newlib \
    --with-multilib-list=rmprofile \
    --disable-nls \
    --disable-shared \
    --disable-libgomp \
    --disable-libatomic \
    --disable-libquadmath \
    --disable-libssp \
    --disable-lto \
    --disable-threads \
    --disable-libobjc \
    --disable-bootstrap \
    --enable-languages=c,c++

make all-gcc -j$(nproc)
make install-gcc
make all-target-libgcc -j$(nproc)
make install-target-libgcc
```

**Key options:**
- `--without-newlib`: No C library available yet
- `--with-multilib-list=rmprofile`: **Critical** — enables `thumb/v8-m.main+fp/hard` multi-lib variant
- `--enable-languages=c,c++`: Must include `c++` for libstdc++ support later
- `--disable-threads`: Bare-metal, no POSIX threads
- `--disable-bootstrap`: Faster build, not needed for cross-compiler

**Verify:**
```bash
$INST/bin/arm-zephyr-eabi-gcc --version
# Should show: GCC 16.1.0

ls $INST/lib/gcc/arm-zephyr-eabi/16.1.0/thumb/v8-m.main+fp/hard/libgcc.a
# Should exist
```

**Why `--with-multilib-list=rmprofile`?**  
GCC 16's default multi-lib list does NOT include `thumb/v8-m.main+fp/hard`. The reference Zephyr SDK (GCC 12.2.0) uses this flag. Without it, the linker cannot find the correct libgcc.a with VFP support, resulting in:
```
error: /tmp/test_cpp.elf uses VFP register arguments, libgcc.a does not
```

---

### Pass 3: Picolibc 1.8.8

Build picolibc as the C library for bare-metal targets. Picolibc is the modern replacement for newlib in Zephyr SDKs.

```bash
cd $BD
mkdir -p b3-picolibc && cd b3-picolibc

# Create CMake toolchain file for armv8m/hard-float
cat > TC-arm-zephyr-eabi.cmake << 'EOF'
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)
set(CMAKE_C_COMPILER arm-zephyr-eabi-gcc)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(TARGET_COMPILE_OPTIONS "-mthumb" "-march=armv8-m.main" "-mfpu=fpv5-sp-d16" "-mfloat-abi=hard")
set(PICOLIBC_LINK_FLAGS "-nostartfiles" "-T" "${CMAKE_CURRENT_SOURCE_DIR}/cmake/TC-arm-none-eabi.ld" "-lgcc")
set(_HAVE_PICOLIBC_TLS_API TRUE)
EOF

PATH=$INST/bin:$PATH \
cmake "$PIC_SRC" \
    -DCMAKE_TOOLCHAIN_FILE="$BD/b3-picolibc/TC-arm-zephyr-eabi.cmake" \
    -DCMAKE_SYSTEM_NAME=Generic \
    -DCMAKE_SYSTEM_PROCESSOR=arm \
    -DCMAKE_C_COMPILER="$INST/bin/arm-zephyr-eabi-gcc" \
    -DTARGET_COMPILE_OPTIONS="-mthumb;-march=armv8-m.main;-mfpu=fpv5-sp-d16;-mfloat-abi=hard" \
    -DPICOLIBC_LINK_FLAGS="-nostartfiles -T $PIC_SRC/cmake/TC-arm-none-eabi.ld -lgcc" \
    -DTINY_STDIO=ON \
    -DPOSIX_IO=ON \
    -DPOSIX_CONSOLE=OFF \
    -DPREFER_SIZE_OVER_SPEED=ON \
    -D_NANO_MALLOC=ON \
    -DPICOLIBC_TLS=ON \
    -DNEWLIB_GLOBAL_ERRNO=OFF \
    -D__SINGLE_THREAD__=OFF \
    -D_HAVE_SEMIHOST=1 \
    -D_LITE_EXIT=1 \
    -D_PICO_EXIT=1 \
    -D_MB_CAPABLE=OFF \
    -D__HAVE_LOCALE_INFO__=OFF \
    -D_WANT_IO_IO_POS_ARGS=OFF \
    -D_WANT_IO_C99_FORMATS=ON \
    -D_ASSERT_VERBOSE=ON \
    -D_TEST=OFF \
    -DCMAKE_INSTALL_PREFIX="$INST/arm-zephyr-eabi"

make -j$(nproc)
make install
```

**Key options:**
- `CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY`: **Critical** — CMake cannot link test programs without a C library yet
- `CMAKE_INSTALL_PREFIX`: Must be `${INST}/arm-zephyr-eabi` (NOT `${INST}`), so headers end up in `${INST}/arm-zephyr-eabi/include/` and libraries in `${INST}/arm-zephyr-eabi/lib/`
- `D_NANO_MALLOC=ON`: Reduces memory footprint (recommended for Cortex-M)
- `D_TINY_STDIO=ON`: Tiny stdio implementation (smaller code size)
- `D_PICOLIBC_TLS=ON`: Thread-local storage support

**Verify:**
```bash
ls ${INST}/arm-zephyr-eabi/lib/*.a
# Should show: libc.a 

ls ${INST}/arm-zephyr-eabi/lib/picolibc.specs
# Should exist
```

---

### Pass 4: Full GCC with Picolibc

Rebuild GCC linked against picolibc. This pass builds the full compiler with libstdc++ for C++ support.

```bash
cd $BD
mkdir -p b4-gcc-full && cd b4-gcc-full

PATH=$INST/bin:$PATH \
$GCC_SRC/configure \
    --target=arm-zephyr-eabi \
    --prefix=$INST \
    --with-newlib \
    --with-multilib-list=rmprofile \
    --disable-nls \
    --disable-shared \
    --disable-libgomp \
    --disable-libatomic \
    --disable-libquadmath \
    --disable-libssp \
    --disable-lto \
    --disable-threads \
    --disable-libobjc \
    --disable-bootstrap \
    --enable-languages=c,c++

make -j$(nproc)
make install
```

**Key options:**
- `--with-newlib`: GCC convention — tells GCC to look for a newlib-compatible C library (picolibc is API-compatible with newlib)
- `--with-multilib-list=rmprofile`: Same as Pass 2, ensures multi-lib variants match

**Verify:**
```bash
$INST/bin/arm-zephyr-eabi-gcc --version
# Should show: arm-zephyr-eabi-gcc (GCC) 16.1.0

$INST/bin/arm-zephyr-eabi-g++ --version
# Should show: arm-zephyr-eabi-g++ (GCC) 16.1.0

ls ${INST}/arm-zephyr-eabi/lib/libstdc++*
# Should exist (built as part of Pass 4)
```

**Note:** libstdc++ is built automatically as part of the full GCC build when `--enable-languages=c,c++` is specified. No separate libstdc++-v3 pass is needed — GCC 16 handles this internally.

---

### Pass 5: GDB 16.1

Build the cross-debugger for debugging ARM Cortex-M targets.

```bash
cd $BD
wget https://ftpmirror.gnu.org/gdb/gdb-16.1.tar.xz
tar xf gdb-16.1.tar.xz

mkdir -p b5-gdb && cd b5-gdb

PATH=$INST/bin:$PATH \
../gdb-16.1/configure \
    --host=x86_64-linux-gnu \
    --target=arm-zephyr-eabi \
    --prefix=$INST \
    --with-python \
    --with-readline \
    --with-expat \
    --disable-nls \
    --disable-gdbtk \
    --without-x \
    --without-uiop \
    --without-guile \
    --without-lzma \
    --without-babeltrace \
    --without-json \
    --without-libunwind \
    --disable-werror

make -j$(nproc)
make install
```

**Key options:**
- `--host=x86_64-linux-gnu`: GDB runs on the host (x86_64 Linux)
- `--target=arm-zephyr-eabi`: Debug ARM Zephyr targets
- `--with-python`: Python support for GDB scripting (required for modern GDB features)
- `--with-readline`: Command-line editing support
- `--with-expat`: XML target description support

**Note:** GDBserver is not built as part of this pass. For debugging Cortex-M targets, use OpenOCD or J-Link GDB Server as the debug server, and connect `arm-zephyr-eabi-gdb` to it via `target extended-remote :3333`.

**Verify:**
```bash
$INST/bin/arm-zephyr-eabi-gdb --version
# Should show: GNU gdb (GDB) 16.1

$INST/bin/arm-zephyr-eabi-gdb -batch -ex "python import sys; print(sys.version)"
# Should show Python version (e.g., 3.12.3)
```

---

## 4. Zephyr Integration

### Toolchain File

A dedicated CMake toolchain file is required to pass include/library paths to Zephyr's build system. This file lives at:
`~/ncs/toolchains/7cbc0036f4/gcc16_cross_compile.cmake`

```cmake
set(triple arm-zephyr-eabi)
set(arch thumb)
set(instr v8-m.main+fp)
set(fp hard)
set(GCC16_ROOT ~/gcc/16.1/gcc16-build-arm/install/opt/gcc16-arm-zephyr-eabi)
set(GCC16_SYSROOT ${GCC16_ROOT}/${triple})
set(GCC16_VERSION 16.1.0)
set(ARCH_FLAGS "-mcpu=cortex-m33 -mthumb -mabi=aapcs -mfpu=fpv5-sp-d16 -mfloat-abi=hard -mfp16-format=ieee")
set(OPTIMIZER_WORKAROUNDS "-fno-tree-dse") #atm for whatever reason dead-store-elimination optimization of GCC 16.1 produces faulty code

# C++ headers (libstdc++)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -nostdinc++ ${ARCH_FLAGS} ${OPTIMIZER_WORKAROUNDS} -I${GCC16_SYSROOT}/include/c++/${GCC16_VERSION} -I${GCC16_SYSROOT}/include/c++/${GCC16_VERSION}/${triple}")

# C headers (GCC internal)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -nostdinc ${ARCH_FLAGS} ${OPTIMIZER_WORKAROUNDS}")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -I${GCC16_SYSROOT}/include")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -I${GCC16_ROOT}/lib/gcc/${triple}/${GCC16_VERSION}/include")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -I${GCC16_ROOT}/lib/gcc/${triple}/${GCC16_VERSION}/include-fixed")

# Library search paths
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -L${GCC16_SYSROOT}/lib/${arch}/${instr}/${fp}")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -L${GCC16_ROOT}/lib/gcc/${triple}/${GCC16_VERSION}/${arch}/${instr}/${fp}")

# Sysroot for Zephyr's --sysroot flag
set(CMAKE_SYSROOT ${GCC16_SYSROOT})
```

**Important notes:**
- Do NOT include `-specs=${GCC16_SYSROOT}/lib/picolibc.specs` in this file — Zephyr's picolibc module already handles this via its own mechanism
- `-nostdinc` and `-nostdinc++` are required because Zephyr uses its own include paths

### Environment Setup

An environment script sets up the cross-compile variant:
`~/ncs/toolchains/7cbc0036f4/gcc16_cross_compile_env.sh`

```bash
#!/bin/sh
export PYTHONHOME=~/ncs/toolchains/7cbc0036f4/usr/local
export PYTHONPATH=~/ncs/toolchains/7cbc0036f4/usr/local/lib/python3.12:~/ncs/toolchains/7cbc0036f4/usr/local/lib/python3.12/site-packages
export PATH=~/gcc/16.1/gcc16-build-arm/install/opt/gcc16-arm-zephyr-eabi/bin:~/ncs/toolchains/7cbc0036f4/usr/local/bin:$PATH
export ZEPHYR_TOOLCHAIN_VARIANT=cross-compile
export CROSS_COMPILE=~/gcc/16.1/gcc16-build-arm/install/opt/gcc16-arm-zephyr-eabi/bin/arm-zephyr-eabi-
```

**Key variables:**
- `ZEPHYR_TOOLCHAIN_VARIANT=cross-compile`: Zephyr's generic cross-compile mode (recommended for custom GCC)
- `CROSS_COMPILE`: Must end with a dash (`arm-zephyr-eabi-`)

### Building Zephyr Projects

```bash
# Load environment
source ~/ncs/toolchains/7cbc0036f4/gcc16_cross_compile_env.sh

# Build with toolchain file
cd /path/to/zephyr/project
west build -b <board_name> -- \
    -DTARGET_TOOLCHAIN_FILE=~/ncs/toolchains/7cbc0036f4/gcc16_cross_compile.cmake \
    -DTOOLCHAIN_HAS_PICOLIBC=ON
```

---

## 5. C++ Standard Library Configuration

### libstdc++ Headers

libstdc++ headers are installed to:
```
$INST/arm-zephyr-eabi/include/c++/16.1.0/
├── bits/           # Generic libstdc++ bits
├── ext/            # GNU extensions
├── arm-zephyr-eabi/
│   ├── bits/       # Target-specific bits (c++config.h)
│   └── thumb/      # Thumb-specific bits
└── ... (all standard libstdc++ headers)
```

### libstdc++ Libraries

libstdc++ libraries are installed to:
```
$INST/arm-zephyr-eabi/lib/thumb/v8-m.main+fp/hard/
├── libstdc++.a
├── libsupc++.a
```

### C++ Standard Version

Zephyr defaults to C++11. For C++17 features (`<string_view>`, etc.), set:

**In prj.conf:**
```
CONFIG_CPP=y
CONFIG_PICOLIBC_USE_MODULE=n #tell Zephyr not to build picolib from module, toolchain should provide it
CONFIG_STD_CPP2B=y  # Enables C++2b (C++23) which includes C++17 features
```

#### Adding C++26 support to Zephyr

If it's missing it could be relatively easily added by adding config at
`<zephyr>/lib/cpp/Kconfig`

```
config STD_CPP26
	bool "C++ 26"
	help
	  2026 C++ standard
```

and modifying `<zephyr>/CMakeLists.txt`
find a place `elseif(CONFIG_STD_CPP20)` and add another `elseif` like:

```
  elseif(CONFIG_STD_CPP26)
    set(STD_CPP_DIALECT_FLAGS $<TARGET_PROPERTY:compiler-cpp,dialect_cpp26>)
    list(APPEND CMAKE_CXX_COMPILE_FEATURES ${compile_features_cpp20})
```

at `<zephyr>/cmake/compiler/gcc/compiler_flags.cmake`:

```
set_property(TARGET compiler-cpp PROPERTY dialect_cpp26 "-std=c++26"
  "-Wno-register" "-Wno-volatile")
```

at `<zephyr>/cmake/compiler/compiler_flags_template.cmake`:

```
set_property(TARGET compiler-cpp PROPERTY dialect_cpp26)
```

and finally in your project:

```
CONFIG_STD_CPP26=y
```

### Header-Only C++ Facilities

For bare-metal targets, only header-only facilities should be used:
- `<atomic>` — works (header-only, no runtime)
- `<string_view>` — works (header-only, no runtime)
- `<memory>` — works (header-only)
- `<vector>` — requires `operator new` (may work with custom allocator)
- `<iostream>` — does NOT work (requires hosted mode, stdio)

**Bare-metal C++ requirements:**
- `-ffreestanding` or `_GLIBCXX_HOSTED=0` for freestanding mode
- Zephyr provides its own memory management
- No dynamic loading, no POSIX threads, no locale data

## 6. Key Changes & Workarounds

### 1. `--with-multilib-list=rmprofile`

**Problem:** GCC 16's default multi-lib list does NOT include `thumb/v8-m.main+fp/hard`.  
**Fix:** Add `--with-multilib-list=rmprofile` to Pass 2 and Pass 4 configure.

Without this, the linker cannot find libgcc.a with VFP support:
```
error: /tmp/test_cpp.elf uses VFP register arguments, libgcc.a does not
```

### 2. CMake `STATIC_LIBRARY` for Picolibc Build

**Problem:** CMake tries to link a test program during configure, but no C library is available yet.  
**Fix:** Add `CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY` to the picolibc CMake toolchain file.

Error without fix:
```
CMake Error: CMAKE_C_COMPILER cannot run C program
CMake Error: CMAKE_C_COMPILER cannot run target executable
```

### 3. Picolibc Install Prefix

**Problem:** Picolibc installs headers to `${INST}/include/` if `CMAKE_INSTALL_PREFIX` is set to `${INST}`.  
**Fix:** Set `CMAKE_INSTALL_PREFIX` to `${INST}/arm-zephyr-eabi` so headers end up in `${INST}/arm-zythe-eabi/include/`.

### 4. `--with-newlib` in Pass 4

**Problem:** Pass 4 needs to find picolibc headers and libraries.  
**Fix:** Use `--with-newlib` (GCC convention for bare-metal C library detection). Picolibc is API-compatible with newlib, so GCC finds it in `${INST}/arm-zephyr-eabi/`.

---

