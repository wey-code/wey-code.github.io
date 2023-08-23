---
layout: post
title: "Pytorch 2.0.0 Debug版 报段错误"
subtitle: ""
author: "Slc"
header-img: "img/post-bg-dreamer.jpg"
header-mask: 0.4
tags:
  - Pytorch 
  - Bug
---

# 编译命令

通常编译pytorch debug版的命令为：

```bash
function init_torch() {
    # export CMAKE_PREFIX_PATH=${CONDA_PREFIX:-"$(dirname $(which conda))/../"}
    # echo ${CONDA_PREFIX:-"$(dirname $(which conda))/../"}
    
    export CMAKE_BUILD_TYPE=debug
    export _GLIBCXX_USE_CXX11_ABI=1

    # export USE_LEVELDB=0
    # export USE_LMDB=0
    # export USE_OPENCV=0
    # export USE_TENSORRT=0
    # export USE_FFMPEG=0
    # export USE_REDIS=0

    # export USE_XNNPACK=0
    # export USE_XNNPACK=OFF

    # export USE_QNNPACK=0
    # export USE_NNPACK=0
    # export USE_NNPACK=OFF
    # export USE_MKLDNN=0
    # export USE_NNPACK=0
    # export USE_NINJA=0
    # export USE_PYTORCH_QNNPACK=0
    # export USE_PYTORCH_QNNPACK=OFF
    # export USE_ROCM=0
    # export USE_FBGEMM=0
    # export USE_OPENMP=0
    # export USE_NINJA=OFF
    # export USE_NINJA=0

    # export USE_OPENBLAS=0
    # export USE_OPENBLAS=OFF

    export BUILD_TEST=0
    export BUILD_TEST=OFF

    # export USE_CUDA=0
    # export USE_CUDA=OFF
    export USE_DISTRIBUTED=ON
    export CCACHE_DISABLE=1

    # cd build && rm -rf ./* && cd ../
}

cd pytorch
init_torch
BUILD_BINARY=0 USE_PRECOMPILED_HEADERS=1 BUILD_TEST=0  DEBUG=1 python setup.py build_ext -i
```

# 问题描述

但是针对pytorch2.0.0，如果编译debug版本。在使用时，如果`import torch`，再退出的话会报如下错误：

```bash
Python 3.8.10 (default, Jun 22 2022, 20:18:18)
[GCC 9.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import torch
>>> quit()
pure virtual method called
terminate called without an active exception
Aborted
```
# 解决方法

经过查询，是因为其中间有一个变量的生命周期没有正确控制，导致过早析构。

可以修改：

![avatar](/img/in-post/pytorch_bug/pytorch2.0.0_bug.png "修复方法")

再重新编译可以解决该问题。（目前最新版的pytorch已经解决该问题）

# 参考资料
[1] [https://github.com/pytorch/pytorch/issues/101185](https://github.com/pytorch/pytorch/issues/101185)