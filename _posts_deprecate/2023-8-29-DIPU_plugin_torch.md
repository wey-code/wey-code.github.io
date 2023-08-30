---
layout: post
title: "DIPU如何注册到torch"
subtitle: "源码分析"
author: "Slc"
header-img: "img/post-bg-dreamer.jpg"
header-mask: 0.4
tags:
  - Pytorch 
  - Bug
---

# 整体分析

从用户角度看来，DIPU是一个设备，我们将计算发到DIPU上，再由DIPU指派具体的硬件。

实际操作中，我们为XPU设备注册我们的DIPU具体的执行函数，并且采用mock代码，使得调用cuda时会自动转发到XPU上（具体mock方法见后文）。

# __Init__文件分析

DIPU模块一般使用方法是使用`import torch_dipu`，此时会调用https://github.com/DeepLink-org/DIPU/blob/main/torch_dipu/__init__.py函数进行初始化，在该文件里，我们会进行c++模块的导入、以及一些mock代码的逻辑。

## 1.C++代码导入

这主要是在https://github.com/DeepLink-org/DIPU/blob/cf4d70934e75f3635425f8a1f167b03f1e3b53c2/torch_dipu/__init__.py#L17C2-L17C2调用了

```python
from torch_dipu import _C
from torch_dipu import dipu
from torch_dipu.dipu import *

```

此时会将cpp库全部导入（cpp库里有一些注册代码的逻辑，例如`torch_dipu/csrc_dipu/aten/ops/AutoGenedKernels.cpp`里的算子，为导入库时即时注册）。

```c++
namespace at {

TORCH_LIBRARY_IMPL(aten, DIPU_DEVICE_TYPE_MACRO, m) {

    DIOPI_ATEN_FUNC("fill_.Scalar", ::diopiFill, dipu::native::dipu_fill__scalar);

    DIOPI_ATEN_FUNC("add.Scalar_out", ::diopiAddScalar, dipu::native::dipu_add_scalar_out);

    DIOPI_ATEN_FUNC("add_.Scalar", ::diopiAddInpScalar, dipu::native::dipu_add__scalar);

    DIOPI_ATEN_FUNC("add_.Tensor", ::diopiAddInp, dipu::native::dipu_add__tensor);

    DIOPI_ATEN_FUNC("add.out", ::diopiAdd, dipu::native::dipu_add_out);

    DIOPI_ATEN_FUNC("sub.Scalar_out", ::diopiSubScalar, dipu::native::dipu_sub_scalar_out);

    DIOPI_ATEN_FUNC("sub.out", ::diopiSub, dipu::native::dipu_sub_out);

    // .....
}
} // namespace at

```

但是不是所有算子都是即时注册的。例如在使用MMCV时，编译可能会导入DIPU的_C库。但我们有时在使用MMCV且不使用torch_dipu时（例如在cpu跑基准数据时），并不希望DIPU的某些算子注册，以覆盖原有算子行为。

针对此，目前DIPU开发了延时注册的策略，即只有显式调用_C.init_resource()，有些算子才会注册（如pin_memory类算子，相关代码见https://github.com/DeepLink-org/DIPU/blob/5fcec5e16484335e0ccf758bb8abc7e25f692e4f/torch_dipu/csrc_dipu/aten/RegisterDIPU.cpp#L88）。

值得注意的是，`_C`模块严重依赖`libtorch_dipu_python.so`和`libtorch_dipu.so`。`_C`模块本身的源文件只负责暴露可供python调用的接口。

## 2.dipu子模块导入

同时还导入torch_dipu的一些子模块的初始化代码，例如https://github.com/DeepLink-org/DIPU/blob/5fcec5e16484335e0ccf758bb8abc7e25f692e4f/torch_dipu/dipu/device.py#L9，此时会进行一些设备变量的定义和资源初始化。

```python
# torch_dipu/dipu/device.py

__dipu__ = 'dipu'
__diputype__ = 'xpu'
__vendor__ = _C.dipu_vendor  # need update when compile
_device_t = Union[torch.device, str, int, None]
_C.init_resource()   #进行cpp层的资源初始化，包括延迟注册算子(如pin_memory等)

```

## 3.Mock cuda入口

Mock cuda的逻辑，这时会对于调用cuda的逻辑进行截获，使得调用cuda设备时实际调用的是dipu设备，具体由一个环境变量控制。

```python
mockcuda = False if os.environ.get("DIPU_MOCK_CUDA", 'True').lower()=='false' else True

```

__init__文件主要调用`apply_patches()`，来对于torch一些函数进行重写，从而实现mock cuda的逻辑。具体而言

```python
def apply_patches():
    apply_tensor_method_patch()
    apply_torch_function_patch()
    apply_dist_patch()
    apply_tensor_type_patch()
    apply_profiler_patch()
    apply_temp_patch()
    apply_dataloader_patch()
    apply_optim_patch()

def apply_tensor_method_patch():
    torch.Tensor.to = GetDeviceProxy(torch.Tensor.to)
    torch.Tensor.is_pinned = GetDeviceProxy(torch.Tensor.is_pinned)
    torch.Tensor.pin_memory = GetDeviceProxy(torch.Tensor.pin_memory)
    #......

```

# Mock Cuda逻辑重点分析

实现mock的逻辑重点是`GetDeviceProxy`函数，该函数声明在https://github.com/DeepLink-org/DIPU/blob/5fcec5e16484335e0ccf758bb8abc7e25f692e4f/torch_dipu/dipu/device.py#L54，具体如下：

```python
torch.device = _DIPUDevice

# wrap device related func
def GetDeviceProxy(rawfunc, pos = 0, name = "device", caller = "obj"):
    def _replaceDevice(args, kwargs):
        # pos device
        if pos >= 0 and pos < len(args) and (isinstance(args[pos], int)
                or isinstance(args[pos], str)):
            argList = list(args)
            argList[pos] = torch.device(args[pos])
            args = tuple(argList)
        deviceValue = kwargs.get(name, None)
        if deviceValue != None and (isinstance(deviceValue, int)
                or isinstance(deviceValue, str)):
            kwargs[name] = torch.device(deviceValue)
        return args, kwargs

    def _proxyFuncInst(self, *args, **kwargs):
        args, kwargs = _replaceDevice(args, kwargs)
        return rawfunc(self, *args, **kwargs)

    def _proxyFuncStatic(*args, **kwargs):
        args, kwargs = _replaceDevice(args, kwargs)
        return rawfunc(*args, **kwargs)

    # class __new__ always pass cls parameter to args
    def _proxyNewClass(cls, *args, **kwargs):
        args, kwargs = _replaceDevice(args, kwargs)
        return rawfunc(*args, **kwargs)

    if caller == "static":
        return _proxyFuncStatic
    elif caller == "class_new":
        return _proxyNewClass
    else:
        return _proxyFuncInst

```

可以看到，我们对于`torch.device`进行了覆盖，再对传入的函数加入了装饰器进行修饰。具体而言，将传入参数的`device`信息使用我们的`torch.device（_DIPUDevice）`进行了替换。

而我们的`_DIPUDevice`是基于原生的`torch.device`，只有在开启`mockcuda`并且调用`cuda`设备时，我们会将设备信息更换为`xpu`。同时如果直接调用`dipu`，也会更换为`xpu`。从而实现了mock cuda的逻辑。

```python
__dipu__ = 'dipu'
__diputype__ = 'xpu'
__vendor__ = _C.dipu_vendor  # need update when compile

#....

class _MetaDeviceType(type):
    device_ = torch.device
    def __instancecheck__(cls, instance):
        if isinstance(instance, _MetaDeviceType.device_):
            return True
        return False

# torch.Device is a final class. cannot inherit
# csrc/Device.cpp THPDevice_pynew:
# "Device(Device device)" Device type can be Device, Long, String
# "Device(c10::string_view type, int64_t? index=-1)"
class _DIPUDevice(metaclass=_MetaDeviceType):
    @staticmethod
    def __doreplace(arg):
        if (__dipu__ in arg):
            arg = arg.replace(__dipu__, __diputype__)
        if (mockcuda and "cuda" in arg):
            arg = arg.replace("cuda", __diputype__)
        return arg

    def __new__(cls, *args, **kwargs):
        if len(args) == 1 and isinstance(args[0], int) and mockcuda:
            # modify default int device type only when "mock cuda".
            dev_name = __diputype__ + ":" + str(args[0])
            return _MetaDeviceType.device_(dev_name)
        # handle device as str
        if len(args) >= 1 and isinstance(args[0], str):
            argList = list(args)
            argList[0] = cls.__doreplace(args[0])
            args = tuple(argList)
        # handle device in type key, not support int type but str and device
        deviceValue = kwargs.get("type", None)
        if deviceValue != None and isinstance(deviceValue, str):
            kwargs["type"] = cls.__doreplace(deviceValue)
        return _MetaDeviceType.device_(*args, **kwargs)

```