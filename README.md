# NW.js Development / Porting for RISC-V

## Build host

Ubuntu 20.04.4 LTS
Debian 11


## Get the code

1. Setup depot_tools.
```
cd $HOME
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH=$PATH:${HOME}/depot_tools
```

2. Download a custom .gclient file for NW.js build.
```
mkdir nwjs && cd nwjs
wget https://raw.githubusercontent.com/rebeccasf/nwjs.config/master/gclient -O .gclient
```

3. Sync all source code.
```
cd src
gclient sync --with_branch_heads
```

4. Unpack the sysroot tarball into a specific folder path.
```
mkdir -p build/linux/debian_sid_riscv64-sysroot
pushd build/linux/debian_sid_riscv64-sysroot
tar xf $HOME/riscv-chromium-patch/debian_sid_riscv64-sysroot/debian_starfive_chroot.tar.gz --strip-components=1
popd
```

5. Run GN.
```
gn gen out/nw --args='target_cpu="riscv64" target_os="linux" is_debug=false is_component_ffmpeg=true symbol_level=1 blink_symbol_level=0 use_gnome_keyring=false is_clang=true use_lld=false use_gold=false'
```

6. Build the llvm/clang for RISCV.
```
pushd tools/clang/scripts
./build.py --without-android --without-fuchsia
popd
```

7. Configure for NW.js.
```
export GYP_DEFINES="target_arch=riscv64"
GYP_CHROMIUM_NO_ACTION=0 ./build/gyp_chromium -I third_party/node-nw/common.gypi -D building_nw=1 -D clang=1 third_party/node-nw/node.gyp
```

8. Start ninja-build.
```
ninja -C out/nw nwjs -j128 -k0
```

9. Build libnode for NW.js.
```
ninja -C out/Release node -k0
ninja -C out/nw copy_node
```


## Misc

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
