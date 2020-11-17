## Gdev: Open-Source GPGPU Runtime and Driver Software

### About Gdev 
Gdev is an open-source software for NVIDIA GPGPUs. Further information can be found [here](https://github.com/CPFL/gdev)  
By looking into Gdev, I expect to understand how GPU drivers should be implemented, especially memory management and communication with the CPU.

### Log
- 2020-09-15: Start looking into Gdev
- 2020-10-07: Finished setting up Gdev that works with Nouveau [Installing Gdev](#installing-gdev)
- 2020-11-06: Found out how to rebuild Nouveau driver to use with Gdev [Rebuilding Nouveau](#rebuiliging-nouveau)

### Installing Gdev
- Note that there are a lot of trials and errors here, meaning you need to *pay close attention to the information in* **boldface** *to avoid making the same mistake that I have made*
- Gdev is built on Ubuntu 18.04, hardware spec: Intel i7-7700 CPU @ 3.6 GHz with NVIDIA GTX 480 GPU
  - After installation, use `sudo apt-get update` and `sudo apt-get upgrade` as well as Software updates provided by Ubuntu
- First attempt was to try to build Gdev and use it with NVIDIA proprietary driver (**UNSUCCESSFUL**):
  - Referred to [this](https://linuxhint.com/ubuntu_nvidia_ppa/) to install NVIDIA driver by Ubuntu PPA (at this point, `nvidia-smi` should work)
  - Now follow instructions of [Using Gdev with NVIDIA proprietary driver](https://github.com/CPFL/gdev/blob/master/docs/README.nvrm.md) in [Gdev git](https://github.com/CPFL/gdev) and Gdev build succeeds at this point
  - I've moved on to [Building/Using Gdev with CMake](https://github.com/CPFL/gdev/blob/master/docs/README.cmake.md) and in '2. envytools', a lot of libraries, such as `flex`, `bison`, and `libxml2-dev` are installed
  - `cmake ..` works (with a lot of errors) and then `make` and `sudo make install` builds `envytools` as well
  - Before '3. Building Gdev', switch to `~/gdev` directory and run `cmake -H. -Brelease -Ddriver=nvrm -Duser=ON -Druntime=ON -Dusched=OFF -Duse_as=OFF -DCMAKE_BUILD_TYPE=Release` because we're trying to use *NVIDIA driver*
  - `make -C release` raises error saying it cannot find `boost/bind.hpp`, so I tried `sudo apt-get install libboost-all-dev` to install `boost` library (300MB+ of storage used)
  - All is good *until* **cuda9.0` and up is not supported for Fermi architectures (GTX480)**, so **I changed to cuda8.0** by referring to [this site](https://rodrigodzf.github.io/setup/cuda/2019/04/15/cuda-setup.html) but a few changes needed to be made:
    - change `-run` to `.run`
    - change `cuda/InstallUtils.pm` to `InstallUtils.pm`
  - At this point, `nvcc --version` told me I used CUDA 8.0, not CUDA 9.1 (which was what I was using before)
  - So proceeding to run `make` in `~/gdev/test/cuda/user/madd` says **gcc4 or lower is required** and I was using `gcc-7.5`
    - It turns out I can do this with `gcc-5`: `sudo apt install gcc-5` and `sudo apt install g++-5`, and having `gcc` and `g++` to point to `gcc-5` and `g++-5`, respectively, according to an answer of [this post](https://askubuntu.com/questions/1087150/install-gcc-5-on-ubuntu-18-04)
  - **Tips:** 
    - Sometimes, after rebooting or being idle for a long period of time, `nvidia-smi` will fail to work or `gcc` will not point to anything (and `gcc-5` exists as `gcc-5`, not `gcc`) - in such cases:
      - `nvidia-smi` doesn't work -> go to software & updates (Ubuntu app) and see what driver it is using. make sure to change it to Nvidia's driver metapackage... it just seems to reset on reboot every time
      - `make` will not work -> most likely, `gcc` will point to nothing or `gcc` will point to `gcc 7.5` (first version I have installed long ago) -> simply follow the steps mentioned above, which is:
      ```
      sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-5 10
      sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-5 20
      sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-5 10
      sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-5 20
      sudo update-alternatives --install /usr/bin/cc cc /usr/bin/gcc 30
      sudo update-alternatives --set cc /usr/bin/gcc
      sudo update-alternatives --install /usr/bin/c++ c++ /usr/bin/g++ 30
      sudo update-alternatives --set c++ /usr/bin/g++
      ```
    - After just making symbolic links with `sudo ln -s/usr/bin/gcc-5 /usr/bin/gcc`, it is changed (permanently? not sure...)
    - Unless I'm trying to alternate between 2+ `gcc` versions, creating symbolic link is probably more efficient than doing `update-alternatives`
  - However, running `./user_test 256` results in the following error:
  ```
  [gdev] Failed to query 3
  [gdev] NV0 not supported.
  [gdev] Failed to create a virtual address space object
  cuCtxCreate failed: res = 999
  Test failed
  ```
  - After trying different solutions but having no success, I've decided to *change to using Nouveau* instaed of NVRM (NVIDIA proprietary driver)
- Second attempt is to use Nouveau, and open-source NVIDIA GPGPU driver for Linux
  - Follow instructions of [Using Gdev's user-space runtime library with Nouveau driver](https://github.com/CPFL/gdev/blob/master/docs/README.nouveau.md) and up to `make oldconfig` works well
  - During `make`, there is an error that says:
  ```
  build error: code model kernel does not support PIC mode
  ```
  - This is because I'm trying to compile an old version kernel (Nouveau uses 2.6.x) with new `gcc` (problems with `gcc` with version 6+ or some `gcc-5`)
  - I followed [this site](https://askubuntu.com/questions/851433/kernel-doesnt-support-pic-mode-for-compiling) to get a patch file from [here](https://lists.ubuntu.com/archives/kernel-team/2016-May/077178.html)
  - However, actually calling `patch` does not work - I needed to actually write the following four lines trying to patch the thing into the Makefile of kernel:
  ```
  KBUILD_CFLAGS += $(call cc-option, -fno-pie)
  KBUILD_CFLAGS += $(call cc-option, -no-pie)
  KBUILD_AFLAGS += $(call cc-option, -fno-pie)
  KBUILD_CPPFLAGS += $(call cc-option, -fno-pie)
  ```
  - Afterwards, compile is successful
  - `sudo make modules_install install` has some minor warning/errors but we can move on
  - After running `modprobe -r nouveau; modprobe nouveau modeset=1 noaccel=0`, two errors showed up:
  ```
  modprobe: ERROR: ../libkmod/libkmod-module.c:832 kmod_module_insert_module() could not find module by name='off'
  modprobe: ERROR: could not insert 'off': Unknown symbol in module, or unknown parameter (see dmesg)
  ```
    - `dmesg -w` said some signature was not verified -> edited `.config` in `linux-2.6` directory after doing `make clean && make oldconfig` to add `CONFIG_MODULE_SIG=n` and `CONFIG_MODULE_SIG_ALL=n`  (I think this is so that it will stop asking for verifications of the device GPU)
    - `modprobe` error occurs because NVIDIA performed some updates and the syntax for `.conf` files changed (or something similar happened)
      - Error message seemed to say something like `off` did not exist, and on the internet, people have suggested finding a `~~~/modprobe.d/~~~~.conf` file that says `alias nvidia off` and comment it out (for thye acse of NVIDIA drivers)
      - In my case, I needed the `nouveau` driver, so I found `/lib/modprobe.d/nvidia-graphics-drivers.conf` which had `alias nouveau off` and `aliast lbm-nouveau off` and I commented both of them out
      - Also, because `modprobe nouveau modeset=1 noaccel=0` had an error that said the operation is not permitted to insert the module, I did `sudo modprobe nouveau modeset=1 noaccel=0` and no errors appeared
  - After cloning the git `drm`, it is important to **change the branch of the git so that it has the Makefiles, scripts, and config files I need**
  - I've changed to `libdrm-2.4.98`, which works with the installation of a few new packages
  - In `sudo make install`, there's an error with copying libraries - but the library already exists in the targeted location

### Rebuilding Nouveau
- Nouveau (DRM) library built is *not* used for Gdev + Nouveau builds
- This is because, the default symbolic link is reset to the wrong Nouveau shared library `libdrm_nouveau.so.2.0.0`
- To fix this, I've written a script that *relinks libdrm_nouveau:*
```
#!/bin/bash
# file name: libdrm_nouveau-relink.sh
ll /usr/lib/x86_64-linux-gnu/libdrm_nouveau.so.2
sudo rm /usr/lib/x86_64-linux-gnu/libdrm_nouveau.so.2
sudo ln -s /usr/lib/libdrm_nouveau.so.2.0.0 /usr/lib/x86_64-linux-gnu/libdrm_nouveau.so.2
```
- Thus, the following script can rebuild Nouveau library and relink it immediately:
```
#!/bin/bash
# file name: nouveau-rebuild.sh
cd ~/drm
make
sudo make install
cd
# relink
. libdrm_nouveau-relink.sh
```

### Others:
- [Home](/..)
- Feel free to email me at **gobbly.gobble@gmail.com**