# Chromium Development / Porting for RISC-V

## Build host

Ubuntu 20.04.4 LTS


## Setup cross-compile toolchain

### LLVM / Clang

1. Build a cross-compile toolchain for llvm and clang.
```
$ sudo apt-get -y install binutils build-essential libtool \
  texinfo gzip zip unzip patchutils curl git make cmake \
  ninja-build automake bison flex gperf grep sed gawk \
  python bc zlib1g-dev libexpat1-dev libmpc-dev \
  libglib2.0-dev libfdt-dev libpixman-1-dev 

$ mkdir ~/riscv
$ cd ~/riscv
$ mkdir _install
$ export PATH=`pwd`/_install/bin:$PATH
$ hash -r

# gcc, binutils, newlib, qemu
$ git clone --recursive https://github.com/riscv/riscv-gnu-toolchain
$ pushd riscv-gnu-toolchain
$ ./configure --prefix=`pwd`/../_install --enable-multilib
$ make -j`nproc` linux
$ make -j`nproc` build-qemu
$ popd

# llvm
$ git clone https://github.com/llvm/llvm-project.git riscv-llvm
$ pushd riscv-llvm
$ ln -s ../../clang llvm/tools || true
$ mkdir _build
$ cd _build
$ cmake -G Ninja -DCMAKE_BUILD_TYPE="Release" \
  -DBUILD_SHARED_LIBS=True -DLLVM_USE_SPLIT_DWARF=True \
  -DCMAKE_INSTALL_PREFIX="../../_install" \
  -DLLVM_OPTIMIZED_TABLEGEN=True -DLLVM_BUILD_TESTS=False \
  -DDEFAULT_SYSROOT="../../_install/riscv64-unknown-linux-gnu" \
  -DLLVM_DEFAULT_TARGET_TRIPLE="riscv64-unknown-linux-gnu" \
  -DLLVM_TARGETS_TO_BUILD="RISCV" \
  ../llvm
$ cmake --build . --target install
$ popd
```


## Chromium

1. Follow the Chromium official build instruction for Linux host to get the source code.
https://chromium.googlesource.com/chromium/src/+/refs/heads/main/docs/linux/build_instructions.md

2. Install additional build dependencies (by using the script available within the source code.)

3. Edit the .gclient file in //chromium top level. This is to disable gclient from syncing NaCL code base.
```
solutions = [
  {
    "name": "src",
    "url": "https://chromium.googlesource.com/chromium/src.git",
    "managed": False,
    "custom_deps": {},
    "custom_vars": { "checkout_nacl": False },
  },
]
```

4. Checkout to specific commits where the patchset are based on.
```
$ cd ~/chromium/src
$ git checkout 9fc37dca35728

```

5. Run the Chromium-specific hooks, this will download additional toolchain / binaries that we might need for the build later.
```
$ cd ~/chromium/src
$ gclient runhooks
```

6. Set up the build configurations in args.gn
```
$ mkdir -p out/riscv64
$ vim out/riscv64/args.gn
target_os="linux"
target_cpu="riscv64"

is_component_build = true
is_debug=false
symbol_level=0
v8_symbol_level=0
blink_symbol_level=0

# Disable broken features
use_gnome_keyring=false
enable_nacl=false
treat_warnings_as_errors=false
fatal_linker_warnings=false
enable_dav1d_decoder = false
enable_reading_list=false
enable_vr=false
enable_swiftshader=false
supports_subzero=false
enable_libaom=false
enable_openscreen=false
enable_jxl_decoder=false

# For clang
is_clang = true
clang_base_path = getenv("HOME") + "/riscv/_install"
clang_use_chrome_plugins = false
use_gold = false
use_lld = true
```

7. Apply the patches from this repository.
```
$ cd ~/chromium/src
$ git am <patch>
```

8. Apply the patches in the untracked folder. This is due to third_party components are not tracked under the same git history.

NOTE: There is no script to help in this yet. So we have to run patch command manually in the respective folders.

TODO: Add a script to help user to patch these patches for third_party components.
```
$ cd third_party/<component>
$ patch -p1 < ~/riscv64-chromium-patch/third_party/XXX.patch
```

9. Setup ffmpeg.
```
$ cd third_party/ffmpeg
$ ./chromium/scripts/build_ffmpeg.py linux riscv64
$ ./chromium/scripts/generate_gn.py
$ ./chromium/scripts/copy_config.sh
```

10. Run the build and start resolving build issues.
```
$ autoninja -C out/riscv64 chrome
```

## Debugging

1. Generate dependency tree from GN.
```
$ gn desc out/risc64 chrome --tree
```

2. Launch chromium browser on headless server.
```
# Print screenshot and save as PDF
$ chrome --headless --disable-gpu --print-to-pdf https://www.google.com

# Capture screenshot
$ chrome --headless --disable-gpu --screenshot https://www.google.com
```

Note: Running with --screenshot will produce a file named screenshot.png in the current working directory.

## Credits

1. PPC Porting Guide: https://wiki.raptorcs.com/wiki/Porting/Chromium
2. SUSE Chromium Port: https://build.opensuse.org/package/show/network:chromium/chromium?expand=1
3. Fedora rawhide Chromium Port: https://src.fedoraproject.org/rpms/chromium/tree/rawhide
4. Github User: https://github.com/felixonmars/archriscv-packages/tree/master/chromium
