---
date: 
    created: 2024-11-13
    updated: 2024-11-13
categories:
    - Programming
tags:
    - Python
---

# Pytorch Extension

Illustrate how to write __cpp__ or __cuda__ extension for pytorch.

<!-- more -->

This post originates from my need to write a custom CUDA kernel for a neural network layer. Mainly based on:
- [Pytorch Extension](https://pytorch.org/tutorials/advanced/cpp_extension.html)

However, the official tutorial is a bit complicated. I will try to prune it down to the essential parts.

## Binding

First, create a cpp file `module.cpp` and include the header files:
```cpp
#include <torch/extension.h>
#include <pybind11/pybind11.h> // https://pybind11.readthedocs.io/en/stable/index.html

namespace py = pybind11;
```

Then, design your module in`module.cpp`:

```cpp
int foo() {
    return 0;
}

// TORCH_EXTENSION_NAME is a macro defined by pytorch, which will be replaced by the name of your extension.
PYBIND11_MODULE(TORCH_EXTENSION_NAME, m) {
	m.doc() = "A tiny extension"; // add docstring

	py::module sub = m.def_submodule("sub", "This is a Submodule"); // define a submodule
    sub.def("foo", &foo, "A function under the submodule"); // register the implementation &foo to sub.foo
    // ...
}
```

Write a `setup.py` file to compile the extension:

```py
from setuptools import setup
from torch.utils.cpp_extension import BuildExtension, CppExtension, CUDAExtension

setup(
    name='module',
    ext_modules=[
        # If containing CUDA code, use CUDAExtension instead
        CUDAExtension('module', [
            'module.cpp',
            ...
        ])
    ],
    cmdclass={'build_ext': BuildExtension}
)
```
## Using

Finally, compile the extension using `pip install .`, then you can use it in your python script:

```py
import torch
import module

print(module.sub.foo()) # 0
```

## Troubleshooting
This section is for some common problems I encountered when writing the extension.

!!! warning "Usage"
    To call your custom extension, you must `import torch` at the beginning of your script. Otherwise, the extension will not be loaded.

!!! warning "Linking"
    If you are under Windows, chances are that you will encounter a linking error. This is mainly because the link libraries are not included in the `setup.py`. You can add the following lines to `setup.py`:
    ```py
    extra_link_args = ['-LIBPATH:path/to/your/libs', 'your.lib']
    ```

!!! warning "VScode Intellisense"
    If you are using VScode, you may encounter a problem that the IntelliSense cannot find the header files. This is because the header files are not included in the `include_path`. You can add the following lines to a `.vscode/c_cpp_properties.json` file:
    ```json
    {
        "configurations": [
            {
                "includePath": [
                "${workspaceFolder}/**",
                ".../anaconda3/pkgs/python-3.9.20-h8205438_1/include/**",
                ".../anaconda3/Lib/site-packages/torch/include/**"
            ],
            }
        ],
    }
    ```
    You should replace `.../anaconda3` with the actual path of your anaconda installation. Or replace the path with the actual path of your `torch` installation.
## More
In this part, I will show more usage of **pybind11** and **setuptools**.
If thinis is too long, I will move some of them to another post.

### Pybind11
...

### Setuptools
...
