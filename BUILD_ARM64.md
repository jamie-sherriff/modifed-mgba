# Building mgba libretro for ARM64 (Anbernic 34xxH / MUOS)

This guide covers cross-compiling mgba libretro core for aarch64 and deploying to Anbernic 34xxH running MUOS.

## Prerequisites

- Docker installed and running
- ADB enabled on Anbernic device
- Device connected via USB or network ADB

## Build with Docker (Cross-compile for ARM64)

### One-liner (PowerShell / Bash)

```bash
docker run --rm --user root -v ${PWD}:/home/mgba/src debian:bookworm bash -c "cd /home/mgba/src && mkdir -p build-output && dpkg --add-architecture arm64 && apt-get update && apt-get install -y git cmake build-essential gcc-aarch64-linux-gnu g++-aarch64-linux-gnu libpng-dev:arm64 zlib1g-dev:arm64 && mkdir -p build-aarch64 && cd build-aarch64 && cmake .. -DCMAKE_TOOLCHAIN_FILE=../toolchain-aarch64.cmake -DBUILD_LIBRETRO=ON -DBUILD_QT=OFF -DBUILD_SDL=OFF -DBUILD_STATIC=OFF -DBUILD_SHARED=ON -DCMAKE_BUILD_TYPE=Release -DDISABLE_DEPS=ON -DMINIMAL_CORE=ON && make -j4 mgba_libretro && cp mgba_libretro.so /home/mgba/src/build-output/"
```

### Step by step

```bash
# 1. Ensure toolchain file exists (already in repo)
ls toolchain-aarch64.cmake

# 2. Run Docker build
docker run --rm --user root \
  -v ${PWD}:/home/mgba/src \
  debian:bookworm \
  bash -c "
    cd /home/mgba/src &&
    mkdir -p build-output &&
    dpkg --add-architecture arm64 &&
    apt-get update &&
    apt-get install -y \
      git cmake build-essential \
      gcc-aarch64-linux-gnu g++-aarch64-linux-gnu \
      libpng-dev:arm64 zlib1g-dev:arm64 &&
    mkdir -p build-aarch64 && cd build-aarch64 &&
    cmake .. \
      -DCMAKE_TOOLCHAIN_FILE=../toolchain-aarch64.cmake \
      -DBUILD_LIBRETRO=ON \
      -DBUILD_QT=OFF \
      -DBUILD_SDL=OFF \
      -DBUILD_STATIC=OFF \
      -DBUILD_SHARED=ON \
      -DCMAKE_BUILD_TYPE=Release \
      -DDISABLE_DEPS=ON \
      -DMINIMAL_CORE=ON &&
    make -j4 mgba_libretro &&
    cp mgba_libretro.so /home/mgba/src/build-output/
  "
```

### Verify output

```bash
docker run --rm -v ${PWD}/build-output:/out debian:bookworm \
  bash -c "apt-get update -qq && apt-get install -y -qq file && file /out/mgba_libretro.so"
```

Expected: `ELF 64-bit LSB shared object, ARM aarch64`

## Deploy to Anbernic 34xxH (MUOS)

### Via ADB (USB)

```bash
# 1. Backup original core
adb shell "cp /opt/muos/share/core/mgba_libretro.so /opt/muos/share/core/mgba_libretro.so.bak"

# 2. Push new core
adb push build-output/mgba_libretro.so /tmp/

# 3. Install
adb shell "cp /tmp/mgba_libretro.so /opt/muos/share/core/mgba_libretro.so"

# 4. Restart RetroArch on device
```

### Via ADB (Network)

```bash
# Enable network ADB on device: Settings → Network → ADB
adb connect <device-ip>:5555

# Then same push commands as above
```

### Via SD Card

```bash
# Copy to SD card
cp build-output/mgba_libretro.so /path/to/sdcard/roms/cores/

# Insert SD card, then in MUOS:
# Main Menu → File Manager → Copy to /opt/muos/share/core/
```

## Revert to Original

```bash
# Revert to first backup
adb shell "mv /opt/muos/share/core/mgba_libretro.so.bak /opt/muos/share/core/mgba_libretro.so"

# Or if you made multiple backups
adb shell "mv /opt/muos/share/core/mgba_libretro.so.bak3 /opt/muos/share/core/mgba_libretro.so"
```

## Features in This Build

- **Fast Forward Button** (R2, matches gpSP):
  - Hold R2 → Fast forward
  - Release R2 → Normal speed
  - Works on MUOS via `FASTFORWARDING_OVERRIDE` API
  - Remappable in RetroArch: Quick Menu → Controls → Port 1 Controls → "Fast Forward"

- **Turbo Buttons** (preserved from original mgba):
  - X = Turbo A
  - Y = Turbo B
  - L2 = Turbo L
  - All remappable in RetroArch control menu

- **Optimized for handheld**: Minimal core, no Qt/SDL, Release build

## Troubleshooting

### Fast Forward not working
- Ensure RetroArch on MUOS is recent enough (supports `FASTFORWARDING_OVERRIDE`)
- Check: Quick Menu → Controls → Port 1 Controls → "Fast Forward" should be bindable
- If not working, try RetroArch's built-in: Settings → Input → Hotkey Binds → Fast Forward Toggle

### Core not showing in RetroArch
- Verify file: `/opt/muos/share/core/mgba_libretro.so`
- Restart RetroArch
- Check: Settings → Core → Load Core → mgba_libretro.so

### Build fails
```bash
# Clean and rebuild
rm -rf build-output build-aarch64
# Re-run docker command
```

### Fast Forward button not visible
- RetroArch must support `RETRO_ENVIRONMENT_SET_FASTFORWARDING_OVERRIDE`
- gpSP works on MUOS, so this core will too
- If still missing, restart RetroArch after core load

## Toolchain File (toolchain-aarch64.cmake)

```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)
set(CMAKE_C_COMPILER aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)
set(CMAKE_ASM_COMPILER aarch64-linux-gnu-gcc)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--no-undefined")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--no-undefined")
```