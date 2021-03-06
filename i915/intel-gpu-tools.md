# intel-gpu-tools

## source
https://gitlab.freedesktop.org/drm/igt-gpu-tools

## build

#### 1. install dependencies
```bash
sudo apt install \
gtk-doc-tools \
libcairo2-dev \
libdrm-dev \
libkmod-dev \
libpixman-1-dev \
libpciaccess-dev \
libprocps-dev \
libunwind-dev \
liblzma-dev \
libdw-dev \
python-docutils \
x11proto-dri2-dev \
xutils-dev \
libxmlrpc-core-c3-dev \
libudev-dev \
libglib2.0-dev \
libgsl-dev \
libasound2-dev \
libgsl-dev \
libcurl3
```

#### 2. install meson tool
```bash
# build requires newer meson version, but "apt install meson" cannot meet the requirement, use below cmd to update meson
sudo apt-get install python3 python3-pip ninja-build
pip3 install meson -i https://pypi.tuna.tsinghua.edu.cn/simple
meson -v
```
#### 3. meson build
```bash
cd SOURCE_DIR
mkdir build
meson build
cd build
ninja
```

#### 4. run tests
```bash
cd build/benchmark
sudo ./gem_exec_ctx
```
*Need disable UEFI Security Boot in BOIS* due to "Restrict debugfs when the kernel is locked down"
https://patchwork.kernel.org/patch/10335801
https://news.ycombinator.com/item?id=16761827
