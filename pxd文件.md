# pxd文件
除了使用`.pyx`源文件之外，Cython还使用类似C头文件的`.pxd`文件——它们包含了只适用于Cython模块的Cython声明。我们使用`cimport`关键字来将一个`pxd`文件导入`pyx`模块中。

`pxd`文件有很多用例：

1. 可以用它们来共享外部C声明。

2. 它们可以包含适用于C编译器的内联函数。这些函数应该被标明为`inline`，例如：
```cython
cdef inline int int_min(int a, int b):
    return b if b < a else a
```
3. 当附带一个同名`pyx`文件时，它们向Cython模块提供一个Cython接口，以便其他Cython模块可以使用比Python更高效的协议与之通信。

在我们的积分例子中，我们可以把它分解成这样的`pxd`文件：
1. 添加一个`cmath.pxd`函数定义了C `math.h`头文件中可用的C函数，例如`sin`。然后我们就可以简单地在`integrate.pyx`中`from cmath cimport sin`。
2. 添加一个`integrate.pxd`，这样，用Cython编写的其他模块就可以定义快速自定义函数来做积分。
```cython
cdef class Function:
    cpdef evaluate(self, double x)
cpdef integrate(Function f, double a,
                double b, int N)
```
注意，如果你有一个带有属性的cdef类，那么属性必须在类声明`pxd`文件中声明（如果你使用了），而不是在`pyx`文件中声明。编译器会告诉你具体情况。