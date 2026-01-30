# 让 VSCode 调试器像 PyCharm 一样显示 Tensor Shape、变量形状、变量长度、维度信息

**Author:** 60

**Date:** 2025-11-22

**Link:** https://blog.csdn.net/weixin_51524504/article/details/149293252

#### 文章目录

-   -   [🎯 目标：在 VS Code 调试器中自动显示这些变量信息](#__VS_Code__11)
    -   [🔍 原理简介](#__76)
    -   -   [⚠️ 其他方案的局限性](#__80)
        -   -   [❌ 方案一：重写 \`\_\_repr\_\_\`](#____repr___84)
            -   [❌ 方案二：向 debugpy 注册自定义变量显示器（StrPresentationProvider）](#__debugpy_StrPresentationProvider_88)
    -   [✅ 我的方案优势](#__92)
    -   [🛠️ 具体实现步骤](#__101)
    -   -   [1\. 找到 debugpy 对应的文件目录](#1__debugpy__103)
        -   -   [对于 Windows 用户](#_Windows__107)
            -   [对于 Ubuntu / Linux 用户](#_Ubuntu__Linux__125)
        -   [2\. 修改 \`get\_variable\_details()\` 函数](#2__get_variable_details__173)
    -   [📊 工作流程解析](#__356)
    -   [⚠️ 注意事项](#__371)
    -   [📚 参考文献](#__382)

* * *

你是否也有这样的痛点：在 [PyCharm](https://so.csdn.net/so/search?q=PyCharm&spm=1001.2101.3001.7020) 中调试深度学习模型或代码时，变量区会清晰显示每个变量的 `shape` 和[类型信息](https://so.csdn.net/so/search?q=%E7%B1%BB%E5%9E%8B%E4%BF%A1%E6%81%AF&spm=1001.2101.3001.7020)，而在 VS Code 中却只能看到一团 `tensor(...)`？别急，这篇文章带你一步一步打造 VS Code 的“PyCharm 式调试体验”。

先看 VS Code 调试效果图：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/0d374f5cb1b0488dab6843599ba016f9.png)

### 🎯 目标：在 VS Code 调试器中自动显示这些变量信息

-   ✅ `torch.Tensor`: 显示 `{Tensor: (3, 4)}`
-   ✅ `numpy.ndarray`: 显示 `{ndarray: (2, 2)}`
-   ✅ `pandas.DataFrame`: 显示 `{DataFrame: (5, 3)}`
-   ✅ `list`、`dict`、`set`、`tuple`: 显示长度 `{list: 3}`，`{dict: 3}`，`{set: 3}`，`{tuple: 3}`

为此，你可以使用以下代码片段进行验证调试效果（对比修改前后的差异）：

```python
import torch
import numpy as np
import pandas as pd

# 字符串类型赋值
string_data = "这是一个字符串类型的数据"

# Tensor 类型赋值（使用 PyTorch）
tensor_data = torch.tensor([[1, 2, 3], [4, 5, 6]], dtype=torch.float32)

# numpy 数组类型赋值
numpy_data = np.array([[7, 8, 9], [10, 11, 12]], dtype=np.float64)

# 集合类型赋值
set_data = {1, 2, 3, 4, 4}  # 集合会自动去重

# 列表类型赋值
list_data = [13, 14, 15, 16]

# 字典类型赋值
dict_data = {
    "key1": "value1",
    "key2": 2,
    "key3": [20, 21, 22]
}

# 元组类型赋值
tuple_data = (23, 24, 25)

# DataFrame 类型赋值（使用 pandas）
dataframe_data = pd.DataFrame({
    'col1': [26, 27],
    'col2': ['a', 'b']
})

# 打印各类型数据，方便查看
print("字符串数据:")
print(string_data)
print("\nTensor 数据:")
print(tensor_data)
print("\nnumpy 数据:")
print(numpy_data)
print("\n集合数据:")
print(set_data)
print("\n列表数据:")
print(list_data)
print("\n字典数据:")
print(dict_data)
print("\n元组数据:")
print(tuple_data)
print("\nDataFrame 数据:")
print(dataframe_data)
```

* * *

### 🔍 原理简介

VS Code 的 Python 调试器底层使用的是 [debugpy](https://github.com/microsoft/debugpy)，其中，变量的显示格式由 `pydevd_xml.py` 中的 `get_variable_details()` 函数控制。通过修改该函数逻辑，我们可以为常见的数据结构注入形状（shape）或长度（len）信息，使其直接显示在调试面板中。

#### ⚠️ 其他方案的局限性

在社区中也存在一些尝试解决此问题的方案，但大多存在以下缺陷：

##### ❌ 方案一：重写 `__repr__`

一种直观的做法是通过自定义 `__repr__` 方法来改变变量在调试器中的显示方式[【在 VS Code 中调试 Tensor 形状不显示的问题及解决方案】](https://blog.csdn.net/weixin_51524504/article/details/143101401?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522f917c13fa0880f9e37386b208c30ff74%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=f917c13fa0880f9e37386b208c30ff74&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-1-143101401-null-null.nonecase&utm_term=%E5%BD%A2%E7%8A%B6&spm=1018.2226.3001.4450)。这种方式可以实现变量显示的定制化，但它 **无法影响调试器中内置类型（如 `bool`、`int`、`str` 等）的显示行为**。

##### ❌ 方案二：向 debugpy 注册自定义变量显示器（StrPresentationProvider）

另一种方法是利用 debugpy 提供的扩展机制，注册一个 `StrPresentationProvider`，告诉调试器如何渲染特定类型的变量。[【在 VS Code 调试器中自动显示变量形状和维度信息】](https://blog.csdn.net/weixin_51524504/article/details/149274267?spm=1001.2014.3001.5502)[【VS Code 中为调试器增强变量显示：自动显示张量 Shape、DataFrame 维度和容器长度】](https://blog.csdn.net/qq_42754434/article/details/148600468?spm=1001.2014.3001.5506)。这种方法虽然理论上更优雅，但在实际使用中发现，它会 **读取原始的完整变量内容** 来生成字符串表示，这在面对大型数组、DataFrame 或嵌套结构时会导致 **严重卡顿甚至崩溃**，严重影响调试体验。

### ✅ 我的方案优势

我选择从 **debugpy 内部机制入手**，通过修改其源码中的 `get_variable_details()` 函数，**在变量渲染阶段注入形状信息**，从而避免了上述方法的性能问题和副作用。

这一改动仅作用于调试器前端显示层，不会影响程序运行逻辑，也不会因变量过大而造成性能瓶颈。

而且，debugpy 在内部已经对变量内容做了优化处理，**只读取必要的元数据（如 shape、dtype、len）而不加载整个对象内容**，因此能保持几乎与原始 VS Code 调试器相同的响应速度。

* * *

### 🛠️ 具体实现步骤

#### 1\. 找到 debugpy 对应的文件目录

根据你使用的编辑器、操作系统以及是否通过 SSH 远程工作，找到对应的文件目录：

##### 对于 Windows 用户

如果你在 **Windows 本地** 使用 **VS Code 或 Cursor**，路径通常如下：

-   **VS Code (本地)**:

```bash
C:\Users\<你的用户名>\.vscode\extensions\ms-python.debugpy-<版本号>-win32-x64\bundled\libs\debugpy\_vendored\pydevd\_pydevd_bundle\pydevd_xml.py
```

_示例:_ `C:\Users\Tang\.vscode\extensions\ms-python.debugpy-2025.10.0-win32-x64\...`

-   **Cursor (本地)**:

```bash
C:\Users\<你的用户名>\.cursor\extensions\ms-python.debugpy-<版本号>-win32-x64\bundled\libs\debugpy\_vendored\pydevd\_pydevd_bundle\pydevd_xml.py
```

_示例:_ `C:\Users\Tang\.cursor\extensions\ms-python.debugpy-2025.8.0-win32-x64\...`

> 💡 注意：请将 `<你的用户名>` 和 `<版本号>` 替换为你实际的用户名和插件版本号。

##### 对于 Ubuntu / Linux 用户

如果你在 **Linux 环境** 下工作，需区分是 **本地** 还是 **通过 SSH 远程连接**：

-   **在 Linux 本地工作**:
    
    -   **VS Code (本地)**:
        
        ```bash
        ~/.vscode/extensions/ms-python.debugpy-<版本号>-linux-x64/bundled/libs/debugpy/_vendored/pydevd/_pydevd_bundle/pydevd_xml.py
        ```
        
    -   **Cursor (本地)**:
        
        ```bash
        ~/.cursor/extensions/ms-python.debugpy-<版本号>-linux-x64/bundled/libs/debugpy/_vendored/pydevd/_pydevd_bundle/pydevd_xml.py
        ```
        
-   **通过 SSH 远程连接到 Linux 服务器工作**:
    
    -   **VS Code (远程)**:
        
        ```bash
        ~/.vscode-server/extensions/ms-python.debugpy-<版本号>-linux-x64/bundled/libs/debugpy/_vendored/pydevd/_pydevd_bundle/pydevd_xml.py
        ```
        
    -   **Cursor (远程)**:
        
        ```bash
        ~/.cursor-server/extensions/ms-python.debugpy-<版本号>-linux-x64/bundled/libs/debugpy/_vendored/pydevd/_pydevd_bundle/pydevd_xml.py
        ```
        

> 💡 注意：远程路径的关键区别在于目录名是 `.vscode-server` 和 `.cursor-server`（带有 `-server` 后缀），而本地路径是 `.vscode` 和 `.cursor`。请根据你的实际情况调整 `<版本号>`。

对于`ubuntu系统`，如果你不确定具体路径，或者想快速找到所有相关文件，可以使用 find 命令进行全局搜索。在你的终端中执行以下命令：

```bash
sudo find / -name 'pydevd_xml.py' 2>/dev/null | grep -E '\/\.(vscode|cursor)(-server)?\/extensions\/ms-python\.debugpy'
```

查找结果如下：  
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/2dea1ce7cbf04bbbaa33bc097d1a06fb.png)

**如何解读和选择：**

**1\. 识别编辑器**：路径中包含 .vscode 的属于 VS Code，包含 `.cursor` 的属于 Cursor 编辑器。  
**2\. 识别环境**：路径中包含 `-server` 的（如 `.vscode-server`）是用于远程 SSH 连接的。没有 `-server` 的是用于本地开发的。  
**3\. 选择版本**：通常情况下，你应该选择**版本号最高**的那一个，因为它很可能是你当前正在使用的版本。例如，在上面的结果中，`2025.14.1` 就是 `远程 Cursor` 最新的版本。

根据你的当前工作场景（例如，使用 VS Code 通过 SSH 远程调试），从结果列表中选择最匹配的路径文件修改即可。

* * *

#### 2\. 修改 `get_variable_details()` 函数

修改之前做好备份

打开 `pydevd_xml.py` 文件，找到`get_variable_details()`函数，完整修改之后如下（可直接复制之后进行替换）：

```python
def get_variable_details(val, evaluate_full_value=True, to_string=None, context: Optional[str] = None):
    """
    :param context:
        This is the context in which the variable is being requested. Valid values:
            "watch",
            "repl",
            "hover",
            "clipboard"
    """
    try:
        # This should be faster than isinstance (but we have to protect against not having a '__class__' attribute).
        is_exception_on_eval = val.__class__ == ExceptionOnEvaluate
    except:
        is_exception_on_eval = False

    if is_exception_on_eval:
        v = val.result
    else:
        v = val

    _type, type_name, resolver = get_type(v)
    type_qualifier = getattr(_type, "__module__", "")
    if not evaluate_full_value:
        value = DEFAULT_VALUE
    else:
        try:
            # 添加形状信息
            shape_info = ""
            try:
                # 处理 PyTorch Tensor
                if type_qualifier == "torch" and hasattr_checked(v, 'shape') and hasattr_checked(v, 'dtype'):
                    shape = tuple(v.shape)
                    dtype = str(v.dtype)
                    shape_info = f"{{Tensor: {shape}}} "
                # 处理 NumPy ndarray
                elif type_qualifier == "numpy" and hasattr_checked(v, 'shape') and hasattr_checked(v, 'dtype'):
                    shape = tuple(v.shape)
                    dtype = str(v.dtype)
                    shape_info = f"{{ndarray: {shape}}} "
                # 处理 Pandas DataFrame
                elif type_qualifier == "pandas.core.frame" and hasattr_checked(v, 'shape'):
                    shape = tuple(v.shape)
                    shape_info = f"{{DataFrame: {shape}}} "
                # 处理 Pandas Series
                elif type_qualifier == "pandas.core.series" and hasattr_checked(v, 'shape'):
                    shape = tuple(v.shape)
                    dtype = str(v.dtype)
                    shape_info = f"{{Series: {shape}}} "
                # 处理其他有 shape 属性的对象
                elif hasattr_checked(v, 'shape'):
                    shape_info = f"{{{v.shape}}} "
                # 处理可计数对象
                elif hasattr_checked(v, '__len__'):
                    try:
                        length = len(v)
                        # 对于字符串类型，只显示 {str} 而不显示长度
                        if type_name == "str":
                            shape_info = f"{{{type_name}}} "
                        else:
                            shape_info = f"{{{type_name}: {length}}} "
                    except:
                        pass
            except:
                pass
            
            str_from_provider = _str_from_providers(v, _type, type_name, context)
            if str_from_provider is not None:
                value = shape_info + str_from_provider

            elif to_string is not None:
                value = shape_info + to_string(v)

            elif hasattr_checked(v, "__class__"):
                if v.__class__ == frame_type:
                    value = pydevd_resolver.frameResolver.get_frame_name(v)

                elif v.__class__ in (list, tuple):
                    if len(v) > 300:
                        value = "%s: %s" % (str(v.__class__), "<Too big to print. Len: %s>" % (len(v),))
                    else:
                        value = "%s: %s" % (str(v.__class__), v)
                else:
                    try:
                        cName = str(v.__class__)
                        if cName.find(".") != -1:
                            cName = cName.split(".")[-1]

                        elif cName.find("'") != -1:  # does not have '.' (could be something like <type 'int'>)
                            cName = cName[cName.index("'") + 1 :]

                        if cName.endswith("'>"):
                            cName = cName[:-2]
                    except:
                        cName = str(v.__class__)

                    value = "%s: %s" % (cName, v)
            else:
                value = shape_info + str(v)
        except:
            try:
                value = repr(v)
            except:
                value = "Unable to get repr for %s" % v.__class__

    # fix to work with unicode values
    try:
        if value.__class__ == bytes:
            value = value.decode("utf-8", "replace")
    except TypeError:
        pass

    return type_name, type_qualifier, is_exception_on_eval, resolver, value
```

具体来说，在 `get_variable_details()` 函数中添加了1处内容，并修改了3处内容，具体如下：

1.  添加了1处内容。  
    找到`if not evaluate_full_value:`这个地方，进行下面的添加：

```python
if not evaluate_full_value:
        value = DEFAULT_VALUE
else:
    try:
        # 添加形状信息
        shape_info = ""
        try:
            # 处理 PyTorch Tensor
            if type_qualifier == "torch" and hasattr_checked(v, 'shape') and hasattr_checked(v, 'dtype'):
                shape = tuple(v.shape)
                dtype = str(v.dtype)
                shape_info = f"{{Tensor: {shape}}} "
            # 处理 NumPy ndarray
            elif type_qualifier == "numpy" and hasattr_checked(v, 'shape') and hasattr_checked(v, 'dtype'):
                shape = tuple(v.shape)
                dtype = str(v.dtype)
                shape_info = f"{{ndarray: {shape}}} "
            # 处理 Pandas DataFrame
            elif type_qualifier == "pandas.core.frame" and hasattr_checked(v, 'shape'):
                shape = tuple(v.shape)
                shape_info = f"{{DataFrame: {shape}}} "
            # 处理 Pandas Series
            elif type_qualifier == "pandas.core.series" and hasattr_checked(v, 'shape'):
                shape = tuple(v.shape)
                dtype = str(v.dtype)
                shape_info = f"{{Series: {shape}}} "
            # 处理其他有 shape 属性的对象
            elif hasattr_checked(v, 'shape'):
                shape_info = f"{{{v.shape}}} "
            # 处理可计数对象
            elif hasattr_checked(v, '__len__'):
                try:
                    length = len(v)
                    shape_info = f"{{{type_name}: {length}}} "
                except:
                    pass
        except:
            pass
```

2.  然后在构建最终显示值时，将 `shape_info` 插入前面，共3处：

`value = str_from_provider`修改如下：

```python
value = shape_info + str_from_provider
```

`value = to_string(v)`修改如下：

```python
value = shape_info + to_string(v)
```

`value = str(v)`修改如下：

```python
value = shape_info + str(v)
```

* * *

### 📊 工作流程解析

当我们在 VS Code 中启动调试会话时，整个流程如下：

1.  VS Code 启动调试器并加载内置的 debugpy 模块。
2.  debugpy 连接到目标 Python 程序并开始监听断点。
3.  当程序暂停时，debugpy 收集当前作用域内的变量信息。
4.  在变量渲染阶段，调用 `get_variable_details()` 函数生成显示字符串。
5.  我们的修改在此处注入形状信息。
6.  最终结果返回给 VS Code 前端展示。

需要注意的是，VS Code 优先使用其自带的 debugpy，而不是环境中的 pip 安装版本。因此，我们的修改需针对 VS Code 扩展目录中的源文件。

* * *

### ⚠️ 注意事项

1.  **VS Code 更新覆盖修改**：每次更新 VS Code 或 Python 扩展后，可能需要重新应用修改。
2.  **备份原始文件**：修改前务必备份原文件，以便恢复或对比。
3.  `评论区中说`**改了没有生效**：这是改的文件不对，包含`pydevd_xml.py`文件的位置很多，根据文章中的说明找到正确的文件修改进行：

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/fdb503aedd8842a9b63d65073793d0c4.png)

* * *

### 📚 参考文献

1.  [在 VS Code 中调试 Tensor 形状不显示的问题及解决方案](https://blog.csdn.net/weixin_51524504/article/details/143101401?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522f917c13fa0880f9e37386b208c30ff74%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=f917c13fa0880f9e37386b208c30ff74&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-1-143101401-null-null.nonecase&utm_term=%E5%BD%A2%E7%8A%B6&spm=1018.2226.3001.4450)
2.  [VS Code 中为调试器增强变量显示：自动显示张量 Shape、DataFrame 维度和容器长度](https://blog.csdn.net/qq_42754434/article/details/148600468?spm=1001.2014.3001.5506)
3.  [在 VS Code 调试器中自动显示变量形状和维度信息](https://blog.csdn.net/weixin_51524504/article/details/149274267?spm=1001.2014.3001.5502)