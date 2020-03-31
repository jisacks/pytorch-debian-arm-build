# pytorch-debian-arm-build
Build instructions for PyTorch on Raspberry Pi 3 B+ running the 32-bit Raspbian Buster using a chroot on Ubuntu.

# Pre-Built Wheel File
The include .whl file was built on Ubuntu 18.04 using the below approach, with Python 3.7.3 and PyTorch v1.3.0.

# Build Instructions
Rather than build PyTorch from scratch on the device itself, we can instead build on a more powerful machine running Linux using a combination of chroot and QEMU. A chroot allows us to setup an isolated environment by changing the apparent root directory, while QEMU allows us to emulate the target ARM processor on a non-ARM machine. These instructions were adapted from [here](https://blog.lazy-evaluation.net/posts/linux/debian-armhf-bootstrap.html), [here](https://www.binarytides.com/setup-chroot-ubuntu-debootstrap/), and [here](https://nmilosev.svbtle.com/compling-arm-stuff-without-an-arm-board-build-pytorch-for-the-raspberry-pi).

First, we will create a basic Debian system using debootstrap:
```
sudo apt-get install debootstrap 
sudo mkdir -p <path-to-chroot>
sudo debootstrap --foreign --variant=buildd --arch=armhf buster <path-to-chroot>
```
where <path-to-chroot> is the path to our new file system. Now let's break down the arguments here a bit. The *--foreign* argument indicates that we should only perform the initial unpacking phase of bootstrapping. The second stage installs build-essential packages and must be done using the QEMU, since our target architecture is different than the host (e.g. ARM vs AMD64). The *--variant=buildd* installs the build-essential packages during the second stage and *--arch=armhf* sets the target architecture to armhf, which is for 32-bit architectures that support hardware floating point units (e.g., arm7l). Note that while the CPU on the Pi is technically 64 bits, Raspbian is a 32-bit OS. Finally, *buster* is the version of Debian we are installing.
  
Next, we need to install QEMU:
```
sudo apt-get install qemu-user-static qemu-system-arm virt-manager
```
and copy the QEMU static binary into the chroot file system:
```
sudo cp /usr/bin/qemu-arm-static <path-to-chroot>/usr/bin/
```
  
We also need to copy over the DNS information as the second stage of debootstrap requires network access:
```
sudo cp /etc/resolv.conf <path-to-chroot>/etc/
```
  
Now we enter the chroot and complete the second stage of debootstrap:
```
sudo chroot <path-to-chroot>
/debootstrap/debootstrap --second-stage
```
Again, this second stage should install all of the build-essential packages for Debian Buster.
  
Since we have created a new file system, we need to install all our dependencies to build PyTorch, regardless if they have been installed on the original file system:
```
apt install libopenblas-dev libblas-dev m4 cmake cython python3-dev python3-yaml python3-setuptools python3-wheel python3-pillow python3-numpy git
```
Make sure that you install the version of Python you intend to use with PyTorch on the Pi! Otherwise, you will run into 
```
ModuleNotFoundError: No module named 'torch._C'
```
errors when you try to import torch. Look at this GitHub [issue](https://github.com/pytorch/pytorch/issues/574) for more information regarding this issue.
  
We now need to clone the PyTorch repository and all of its submodules:
```
git clone https://github.com/pytorch/pytorch --recursive && cd pytorch
git checkout v<pytorch-version>
git submodule update --init --recursive  
```
where *<pytorch-version>* is the version of PyTorch we wish to build. Next, we set the following environment variables for the build:
```
export NO_CUDA=1
export NO_DISTRIBUTED=1
export NO_MKLDNN=1 
export BUILD_TEST=0
export MAX_JOBS=8
```
These are fairly self-explanatory, preventing the build script from building with CUDA, distributed support, use of the MKL-DNN library (which does not support ARM processors), and build tests. The last environment variable, *MAX_JOBS*, indicates the number of processors can be used to compile.

To build PyTorch, we simply run
```
python3 setup.py bdist_wheel
```
The entire build process should only take few hours (compared to the >12 hour time it would take on the Pi). The wheel will then be in */pytorch/dist/* folder, which you can install by copying to the Pi and installing with 
```
python -m pip install <path-to-wheel>
```
