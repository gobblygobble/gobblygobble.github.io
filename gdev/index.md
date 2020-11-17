## Gdev: Open-Source GPGPU Runtime and Driver Software

### About Gdev 
Gdev is an open-source software for NVIDIA GPGPUs. Further information can be found [here](https://github.com/CPFL/gdev)  
By looking into Gdev, I expect to understand how GPU drivers should be implemented, especially memory management and communication with the CPU.

### Log
 - 2020-09-15: Start looking into Gdev
 - 2020-10-07: Finished setting up Gdev that works with Nouveau [Gdev setup](#installing-gdev)
 - 2020-11-06: Found out how to rebuild Nouveau driver to use with Gdev [Building Nouveau](#relinking-gdev)

### Installing Gdev
 - Note that there are a lot of trials and errors here, meaning you need to *pay close attention to the information in* **boldface** *to avoid making the same mistake that I have made*
 - Gdev is built on Ubuntu 18.04, hardware spec: Intel i7-7700 CPU @ 3.6 GHz with NVIDIA GTX 480 GPU
   - After installation, use `sudo apt-get update` and `sudo apt-get upgrade` as well as Software updates provided by Ubuntu
 - First attempt was to try to build Gdev and use it with NVIDIA proprietary driver (**UNSUCCESSFUL**):
   - Referred to [this](https://linuxhint.com/ubuntu_nvidia_ppa/) to install NVIDIA driver by Ubuntu PPA (at this point, `nvidia-smi` should work)
   - Now follow instructions of ['Using Gdev with NVIDIA proprietary driver'](https://github.com/CPFL/gdev/blob/master/docs/README.nvrm.md) in the [Gdev git](https://github.com/CPFL/gdev) and Gdev build succeeds at this point
   - I've moved on to ['Building/Using Gdev with CMake'](https://github.com/CPFL/gdev/blob/master/docs/README.cmake.md) and in '2. envytools', a lot of libraries, such as `flex`, `bison`, and `libxml2-dev` are installed. `cmake ..` works (with a lot of errors) and then `make` and `sudo make install` builds `envytools` as well
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