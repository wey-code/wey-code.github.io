---
layout: post
title: "Pytorch的cpp extension"
subtitle: ""
author: "Slc"
header-img: "img/post-bg-dreamer.jpg"
header-mask: 0.4
tags:
  - Pytorch
---

# Motivation

为了方便pytorch的算子拓展以及一些算子的手动融合，我们可以使用pytorch的cpp extension功能——先基于C++手动实现底层算子，之后通过pybind向上暴露接口。最后配置setup.py，将其作为pytorch的拓展包的形式进行安装。


# 一个简易的cpp拓展

## 1.定义底层算子
例如我们定义`deeplink_add`算子


```c++
// deeplink_add.cpp

#include <torch/extension.h>

#include <iostream>

torch::Tensor deeplink_add_forward(torch::Tensor a, torch::Tensor b){
    auto res = a + b;
    return res;
}

// C++扩展API目前没有为我们提供自动生成后向函数的方法。因此，我们还必须实现backward

std::vector<torch::Tensor> deeplink_add_backward(torch::Tensor d_res) {
    auto d_a = d_res;
    auto d_b = d_res;
    return {d_a, d_b};
}

PYBIND11_MODULE(TORCH_EXTENSION_NAME, m) {
  m.def("deeplink_add_forward", &deeplink_add_forward, "deeplink_add forward");
  m.def("deeplink_add_backward", &deeplink_add_backward, "deeplink_add backward");
}
```

## 2.配置编译文件

写好底层算子的实现之后，我们再来配置`setup.py`文件。它的作用是将底层的c++算子进行编译，方便python层进行调用。

例如这里的拓展包为`deeplink`,刚定义的拓展算子为`deeplink.ext_`，

```python
# setup.py

from setuptools import setup, Extension
from torch.utils.cpp_extension import CppExtension, BuildExtension
import glob
import os

def get_ext():
    extensions = []
    extension = CppExtension
    ext_name = 'deeplink.ext_'
    # 包含所有算子文件
    op_files = glob.glob('./deeplink_a/*.cpp')
    include_dirs = [os.path.abspath('./deeplink_a')]
    define_macros = []
    extra_objects = []
    library_dirs = []
    libraries = []
    extra_link_args = []

    extra_compile_args = {'cxx': []}
    extra_compile_args['cxx'] = ['-std=c++14']
    ext_ops = extension(
            name=ext_name,                # 拓展模块名字
            sources=op_files,
            include_dirs=include_dirs,
            define_macros=define_macros,  # 用于定义宏变量
            extra_objects=extra_objects,  # 传递object文件
            extra_compile_args=extra_compile_args,
            library_dirs=library_dirs,
            libraries=libraries,
            extra_link_args=extra_link_args)
    extensions.append(ext_ops)
    return extensions

setup(name='deeplink',
      ext_modules=get_ext(),
      cmdclass={'build_ext': BuildExtension})
```

将上述`setup.py`进行安装：
```bash
python setup.py build_ext #在测试之前，需要将build/lib.linux-x86_64-3.8加入pythonpath里

# 或者采用
python setup.py install
```

测试脚本为：
```python
import torch
import deeplink.ext_

a = torch.tensor([3, 5])
b = torch.tensor([2, 4])

c = deeplink.ext_.deeplink_add_forward(a, b)
d = deeplink.ext_.deeplink_add_backward(a)
print(c)
print(d)
```

## 拓展
事实上，我们可以不局限于cpu上实现。比如一个新的芯片，我们可以让其将内核编译成`so`文件，然后链接到我们的cpp文件里，进行调用，从而实现解耦。

调用kernel的逻辑外层可以包裹一层逻辑，实现一些fallback等操作。将该接口向上暴露，例如：
```c++

#include <torch/extension.h>

#include <iostream>

#include <diopi/diopirt.h>

#include "csrc_dipu/diopirt/diopirt_impl.h"
#include "csrc_dipu/base/basedef.h"

#include "ext_kernel.h"

using dipu::diopi_helper::toDiopiScalar;
using dipu::diopi_helper::toDiopiTensorHandle;

torch::Tensor nms_diopi(torch::Tensor boxes, torch::Tensor scores, float iou_threshold, int offset) {
  auto boxes_p = toDiopiTensorHandle(boxes);
  diopiDevice_t device;
  diopiGetTensorDevice(boxes_p, &device);
  // 如果在host上，则直接在host上运算
  if (device == diopi_host) {
    std::cout<<"we run this on host!"<<std::endl;
    return torch::Tensor();
  }
  diopiContext ctx(dipu::getCurrentDIPUStream().rawstream());
  diopiContextHandle_t ch = &ctx;
  torch::Tensor out;
  auto outp = toDiopiTensorHandle(out);
  diopiTensorHandle_t* outhandle = &outp;
  auto scores_p = toDiopiTensorHandle(scores);
  bool is_mock_cuda = boxes.device().type() == dipu::DIPU_DEVICE_TYPE;

  // 此处的my_nms需要在 ext_kernel.h和ext_kernel.cpp 进行声明和实现
  if (is_mock_cuda) {
    auto ret =
        my_nms(ch, outhandle, boxes_p, scores_p, iou_threshold, offset);
    if (ret == diopiSuccess) {
      auto tensorhandle = reinterpret_cast<torch::Tensor*>(*outhandle);
      return *tensorhandle;
    }
  }
  // 如果没有mock_cuda，则fallback到cpu
  LOG(WARNING) << "Fallback to cpu: ext op nms";
  auto boxes_cpu = boxes.cpu();
  auto scores_cpu = scores.cpu();
  return torch::Tensor();
}

PYBIND11_MODULE(TORCH_EXTENSION_NAME, m) {
  m.def("deeplink_nms", &nms_diopi, "deeplink nms");
}
```

# 参考资料：

[1] https://zhuanlan.zhihu.com/p/348555597

[2] https://pytorch.org/tutorials/advanced/cpp_extension.html