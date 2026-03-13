---
title: Linux 下安装流体力学仿真工具 IBAMR
description: Linux折腾记录
date: 2021-11-20 20:46:00 +0800
categories: [Linux]
tags: [IBAMR]
---

前几天有位学流体力学的朋友问我怎么安装 IBAMR，说是一个有限元分析的开源软件，无奈不太懂这方面的知识，不知道怎么折腾。我去 Github 一看，好嘛，原来还是个 Linux 下的软件，这下他更不会弄了，我想着正好顺便熟悉下 Linux，就答应帮他折腾下了。

## 准备

看下 [Github](https://github.com/IBAMR/IBAMR) 页面，发现有[官方网站](https://ibamr.github.io/)，还自带安装教程，工作应该轻松了许多，跟着教程走就行了。

不过因为是 Linux 程序，首先得去装个虚拟机，这里使用 VMWare，安装 ubuntu 20 LTS，配置方面内存分配 24 GiB，硬盘给60GiB，为啥要给这么高配置，后面就知道了。

## 软件安装

安装完 ubuntu 并配置好后，桌面右键或者按 `Ctrl+Alt+T` 打开终端，开始按照教程安装。这里我将要打的命令列出来以便参考。

不过我们先安装下必要软件，因为我到后面才发现 ubuntu 并没有自带这些软件。

```shell
#安装必要的程序
sudo apt install ssh
sudo service sshd start
sudo apt install make
sudo apt install git
sudo apt install gcc
sudo apt install g++
sudo apt install gfortran
sudo apt install m4
sudo apt install python3-distutils
sudo apt install zlib1g-dev
```

安装完了之后我们要下载需要的软件源码包，IBAMR 有足足十个依赖的库，这意味着我们需要先把依赖装了才能装 IBAMR。为了方便管理我把源码包都提前下载好并放在桌面上，而不是下载在安装目录旁边。

```shell
#下载依赖
cd ~/Desktop
wget https://github.com/IBAMR/IBAMR/archive/v0.9.0.tar.gz
wget https://sourceforge.net/projects/boost/files/boost/1.60.0/boost_1_60_0.tar.gz
wget https://hdf-wordpress-1.s3.amazonaws.com/wp-content/uploads/manual/HDF5/HDF5_1_10_6/source/hdf5-1.10.6.tar.gz
wget https://download.open-mpi.org/release/open-mpi/v4.0/openmpi-4.0.2.tar.gz
wget https://github.com/LLNL/SAMRAI/archive/refs/tags/2.4.4.tar.gz
wget https://github.com/IBAMR/IBAMR/releases/download/v0.9.0/0001-Collect-necessary-IBAMR-patches.patch
wget https://wci.llnl.gov/sites/wci/files/2021-01/silo-4.10.tgz
wget https://github.com/libMesh/libmesh/releases/download/v1.6.2/libmesh-1.6.2.tar.gz
wget http://ftp.mcs.anl.gov/pub/petsc/release-snapshots/petsc-3.13.4.tar.gz
```

提示：安装的时候尽量安装教程给的软件版本，不要装太新的版本，不要问我为什么。

命令执行结束的时候注意看是否下载成功，然后我们先来配置各个依赖库。

### 配置 boost 库

```shell
#配置boost库
cd ~
mkdir sfw
cd sfw
mkdir linux
cd linux
mkdir boost
cd boost
tar xvfz ~/Desktop/boost_1_60_0.tar.gz
mv boost_1_60_0 1.60.0
export BOOST_ROOT=$HOME/sfw/linux/boost/1.60.0
mkdir $BOOST_ROOT/include
ln -s $BOOST_ROOT/boost $BOOST_ROOT/include
```

教程里说 IBAMR 并不需要编译 boost 库，只需要里面的头文件即可。

### 安装 hdf5库

```shell
#安装hdf5库
cd ~/sfw/linux/
mkdir hdf5
cd hdf5
tar xvfz ~/Desktop/hdf5-1.10.6.tar.gz
cd hdf5-1.10.6
./configure \
  CC=gcc \
  CXX=g++ \
  FC=gfortran \
  F77=gfortran \
  --enable-build-mode=production \
  --prefix=$HOME/sfw/linux/hdf5/hdf5-1.10.6
make -j32
make -j32 check
make -j32 install
```

hdf5 是用于管理储存数据的库，注意上面第 7 行至第 13 行要一起复制到终端里执行。

这里开始官方教程出现了一些差错，第 13 行最后的文件夹应为 `hdf5-1.10.6`，后面有用到 hdf5 路径的命令全部都要改。

### 安装 Silo

```shell
#安装Silo
cd ~/sfw/linux
tar xvfz ~/Desktop/silo-4.10.tgz
cd silo-4.10
./configure \
  CC=gcc \
  CXX=g++ \
  FC=gfortran \
  F77=gfortran \
  --prefix=$HOME/sfw/linux/silo/4.10 \
  --disable-silex
make -j32
make -j32 install
```

Silo 是用于读取数据的库。

### 安装 openmpi 库

```shell
#安装openmpi库
cd ~/sfw/linux
tar xvfz ~/Desktop/openmpi-4.0.2.tar.gz
cd openmpi-4.0.2
./configure \
  CC=gcc \
  CXX=g++ \
  FC=gfortran \
  F77=gfortran \
  --prefix=$HOME/sfw/linux/openmpi/4.0.2 \
  --enable-orterun-prefix-by-default
make -j32
make -j32 check
make -j32 install
```

openmpi 是用于高性能计算的库。

### 安装 Hypre 和 PETSc

```shell
#配置Hypre和PETSc
cd ~/sfw
mkdir petsc
cd petsc
tar xvfz ~/Desktop/petsc-3.13.4.tar.gz
mv petsc-3.13.4 3.13.4
cd 3.13.4

export PETSC_DIR=$PWD
export PETSC_ARCH=linux-debug
./configure \
  --CC=$HOME/sfw/linux/openmpi/4.0.2/bin/mpicc \
  --CXX=$HOME/sfw/linux/openmpi/4.0.2/bin/mpicxx \
  --FC=$HOME/sfw/linux/openmpi/4.0.2/bin/mpif90 \
  --with-debugging=1 \
  --download-hypre=1 \
  --download-fblaslapack=1 \
  --with-x=0
make -j32
make -j32 test

export PETSC_DIR=$PWD
export PETSC_ARCH=linux-opt
./configure \
  --CC=$HOME/sfw/linux/openmpi/4.0.2/bin/mpicc \
  --CXX=$HOME/sfw/linux/openmpi/4.0.2/bin/mpicxx \
  --FC=$HOME/sfw/linux/openmpi/4.0.2/bin/mpif90 \
  --COPTFLAGS="-O3" \
  --CXXOPTFLAGS="-O3" \
  --FOPTFLAGS="-O3" \
  --PETSC_ARCH=$PETSC_ARCH \
  --with-debugging=0 \
  --download-hypre=1 \
  --download-fblaslapack=1 \
  --with-x=0
make -j32
make -j32 test
```

这里要分别编译 Debug 和 Release 的版本。

终端执行完命令后一定要看看有没有出错，不然后面会有问题，比如说遇到下面这种情况：

![problem1](/assets/img/2021/problem1.png)

发生如上情况是因为某些原因源码包下载失败，可以多试几次或者更换网络环境。

### 安装 SAMRAI

```shell
#安装SAMRAI
cd ~/sfw
mkdir samrai
cd samrai
mkdir 2.4.4
cd 2.4.4
tar xvfz ~/Desktop/2.4.4.tar.gz

cd SAMRAI-2.4.4
patch -p1 < ~/Desktop/0001-Collect-necessary-IBAMR-patches.patch

cd ..
mkdir objs-debug
cd objs-debug
../SAMRAI-2.4.4/configure \
  --prefix=$HOME/sfw/samrai/2.4.4/linux-g++-debug \
  --with-CC=$HOME/sfw/linux/openmpi/4.0.2/bin/mpicc \
  --with-CXX=$HOME/sfw/linux/openmpi/4.0.2/bin/mpicxx \
  --with-F77=$HOME/sfw/linux/openmpi/4.0.2/bin/mpif90 \
  --with-hdf5=$HOME/sfw/linux/hdf5/hdf5-1.10.6 \
  --without-petsc \
  --without-hypre \
  --with-silo=$HOME/sfw/linux/silo/4.10 \
  --without-blaslapack \
  --without-cubes \
  --without-eleven \
  --without-kinsol \
  --without-petsc \
  --without-sundials \
  --without-x \
  --with-doxygen \
  --with-dot \
  --enable-debug \
  --disable-opt \
  --enable-implicit-template-instantiation \
  --disable-deprecated
make -j32
make -j32 install

cd ~/sfw/samrai/2.4.4
mkdir objs-opt
cd objs-opt
../SAMRAI-2.4.4/configure \
  CFLAGS="-O3" \
  CXXFLAGS="-O3" \
  FFLAGS="-O3" \
  --prefix=$HOME/sfw/samrai/2.4.4/linux-g++-opt \
  --with-CC=$HOME/sfw/linux/openmpi/4.0.2/bin/mpicc \
  --with-CXX=$HOME/sfw/linux/openmpi/4.0.2/bin/mpicxx \
  --with-F77=$HOME/sfw/linux/openmpi/4.0.2/bin/mpif90 \
  --with-hdf5=$HOME/sfw/linux/hdf5/hdf5-1.10.6 \
  --without-hypre \
  --with-silo=$HOME/sfw/linux/silo/4.10 \
  --without-blaslapack \
  --without-cubes \
  --without-eleven \
  --without-kinsol \
  --without-petsc \
  --without-sundials \
  --without-x \
  --with-doxygen \
  --with-dot \
  --disable-debug \
  --enable-opt \
  --enable-implicit-template-instantiation \
  --disable-deprecated
make -j32
make -j32 install
```

SAMRAI 安装之前要先打 IBAMR 给的补丁后再编译（上面已包含）。

### 安装libMesh

```shell
#安装libMesh
cd ~/sfw/linux
mkdir libmesh
cd libmesh
mkdir 1.6.2
cd 1.6.2
tar xvfz ~/Desktop/libmesh-1.6.2.tar.gz
mv libmesh-1.6.2 LIBMESH

cd ~/sfw/linux/libmesh/1.6.2
mkdir objs-debug
cd objs-debug
../LIBMESH/configure \
    --prefix=$HOME/sfw/linux/libmesh/1.6.2/1.6.2-debug \
    --with-methods=dbg \
    PETSC_DIR=$HOME/sfw/petsc/3.13.4 \
    PETSC_ARCH=linux-debug \
    CC=$HOME/sfw/linux/openmpi/4.0.2/bin/mpicc \
    CXX=$HOME/sfw/linux/openmpi/4.0.2/bin/mpicxx \
    FC=$HOME/sfw/linux/openmpi/4.0.2/bin/mpif90 \
    F77=$HOME/sfw/linux/openmpi/4.0.2/bin/mpif90 \
    --enable-exodus \
    --enable-triangle \
    --enable-petsc-required \
    --disable-boost \
    --disable-eigen \
    --disable-hdf5 \
    --disable-openmp \
    --disable-perflog \
    --disable-pthreads \
    --disable-tbb \
    --disable-timestamps \
    --disable-reference-counting \
    --disable-strict-lgpl \
    --disable-glibcxx-debugging \
    --disable-vtk \
    --with-thread-model=none
make -j32
make -j32 install

cd ~/sfw/linux/libmesh/1.6.2
mkdir objs-opt
cd objs-opt
../LIBMESH/configure \
    --prefix=$HOME/sfw/linux/libmesh/1.6.2/1.6.2-opt \
    --with-methods=opt \
    PETSC_DIR=$HOME/sfw/petsc/3.13.4 \
    PETSC_ARCH=linux-opt \
    CC=$HOME/sfw/linux/openmpi/4.0.2/bin/mpicc \
    CXX=$HOME/sfw/linux/openmpi/4.0.2/bin/mpicxx \
    FC=$HOME/sfw/linux/openmpi/4.0.2/bin/mpif90 \
    F77=$HOME/sfw/linux/openmpi/4.0.2/bin/mpif90 \
    --enable-exodus \
    --enable-triangle \
    --enable-petsc-required \
    --disable-boost \
    --disable-eigen \
    --disable-hdf5 \
    --disable-openmp \
    --disable-perflog \
    --disable-pthreads \
    --disable-tbb \
    --disable-timestamps \
    --disable-reference-counting \
    --disable-strict-lgpl \
    --disable-glibcxx-debugging \
    --disable-vtk \
    --with-thread-model=none
make -j32
make -j32 install
```

Github 下载 libMesh 的时候要下载 Release 里上传的压缩文件，下载 Github 自动生成的源码包配置的时候会出错。

这里被教程坑惨了，第 18 行末尾应该是小写的 mpicc 而不是 mpiCC，导致编译 C++ 文件时使用了 C 的编译器，浪费了非常多的时间，虽然写这篇文章的时候已经改正了，但还是给我留下了难忘的经历。

![problem2](/assets/img/2021/problem2.png)

最开始是由于一些在 C++ 里不兼容的 C 语法导致的编译错误，导致了很长的排错时间，最后发现是配置错了的时候差点没气昏过去。

当我以为排除万难的时候，编译器冷不丁地抛出了错误：

![problem3](/assets/img/2021/problem3.png)

内部编译错误？！你特么在逗我呢，还当不当人了，但凡给出点错误信息来都不至于让我束手无策，最后经历N久发现，居然是tmd 内存不够了。。。

Fxxk GCC...

### 安装 IBAMR

```shell
cd ~/sfw
mkdir ibamr
cd ibamr
tar xf ~/Desktop/v0.9.0.tar.gz
mv IBAMR-0.9.0 IBAMR
mkdir ibamr-objs-dbg
cd ibamr-objs-dbg

export BOOST_ROOT=$HOME/sfw/linux/boost/1.60.0
export PETSC_ARCH=linux-debug
export PETSC_DIR=$HOME/sfw/petsc/3.13.4
../IBAMR/configure \
  CFLAGS="-g -O1 -Wall" \
  CXXFLAGS="-g -O1 -Wall -Wno-deprecated-copy" \
  FCFLAGS="-g -O1 -Wall" \
  CC=$HOME/sfw/linux/openmpi/4.0.2/bin/mpicc \
  CXX=$HOME/sfw/linux/openmpi/4.0.2/bin/mpicxx \
  FC=$HOME/sfw/linux/openmpi/4.0.2/bin/mpif90 \
  CPPFLAGS="-DOMPI_SKIP_MPICXX" \
  --with-hypre=$PETSC_DIR/$PETSC_ARCH \
  --with-samrai=$HOME/sfw/samrai/2.4.4/linux-g++-debug \
  --with-hdf5=$HOME/sfw/linux/hdf5/hdf5-1.10.6 \
  --with-silo=$HOME/sfw/linux/silo/4.10 \
  --with-boost=$HOME/sfw/linux/boost/1.60.0 \
  --enable-libmesh \
  --with-libmesh=$HOME/sfw/linux/libmesh/1.6.2/1.6.2-debug \
  --with-libmesh-method=dbg
make -j32 lib
make -j32 examples
```

然后就没有然后了
