# 扩展类型（即cdef类）
为了支持面向对象编程，Cython支持像在Python中一样编写普通的Python类：
```cython
class MathFunction(object):
    def __init__(self, name, operator):
        self.name = name
        self.operator = operator

    def __call__(self, *operands):
        return self.operator(*operands)
```
然而，基于Python的“内置类型”，Cython支持第二种类型的类：<b><i>扩展类型</i></b>，由于声明中使用的关键字，有时被称为“cdef类”。与Python类相比，它们受到了一些限制，但是通常比一般的Python类内存效率更高，速度更快。主要的区别在于,他们使用一个C结构体存储字段和方法，而不是一个Python字典。这允许它们在字段中存储任意的C类型，而不需要Python包装器，并且可以在C语言层面上直接访问字段和方法，而不需要通过Python字典查找。

普通的Python类可以从cdef类继承，但是反过来不行。Cython需要知道完整的继承层次结构，以便安排它们的C结构体，并将其限制为单继承。另一方面，普通的Python类可以继承任意数量的Python类和扩展类型，包括Cython代码和纯Python代码。

到目前为止，我们的积分的例子还不是很有用，因为它只对一个硬编码函数做积分。为了弥补这一点，在几乎不牺牲速度的情况下，我们将使用一个cdef类来表示浮点数的函数：
```cython
cdef class Function:
    cpdef double evaluate(self, double x) except *:
        return 0
```
指令cpdef使该方法的两个版本可用；一个速度较快的用于Cython，另一个速度较慢的用于Python。然后：
```cython
from libc.math cimport sin

cdef class Function:
    cpdef double evaluate(self, double x) except *:
        return 0

cdef class SinOfSquareFunction(Function):
    cpdef double evaluate(self, double x) except *:
        return sin(x ** 2)
```
这比为cdef方法提供python包装器做得稍微多一点：与cdef方法不同，cpdef方法完全可以被Python子类中的方法和实例属性覆盖。与cdef方法相比，它增加了一点调用开销。

为了使类定义对其他模块可见，从而允许在执行它们的模块之外有高效的C语言层面的使用和继承，我们在`sin_of_square.pxd`文件中定义了它们：
```cython
cdef class Function:
    cpdef double evaluate(self, double x) except *

cdef class SinOfSquareFunction(Function):
    cpdef double evaluate(self, double x) except *
```
使用这个，我们现在可以改变我们的积分例子：
```cython
from sin_of_square cimport Function, SinOfSquareFunction

def integrate(Function f, double a, double b, int N):
    cdef int i
    cdef double s, dx
    if f is None:
        raise ValueError("f cannot be None")
    s = 0
    dx = (b - a) / N
    for i in range(N):
        s += f.evaluate(a + i * dx)
    return s * dx

print(integrate(SinOfSquareFunction(), 0, 1, 10000))
```
这几乎与前面的代码一样快，但是由于要积分的函数可以更改，因此更加灵活。我们甚至可以传入一个在Python空间中定义的新函数：
```cython
>>> import integrate
>>> class MyPolynomial(integrate.Function):
...     def evaluate(self, x):
...         return 2*x*x + 3*x - 10
...
>>> integrate(MyPolynomial(), 0, 1, 10000)
-7.8335833300000077
```
这大约比cdef类慢20倍，但仍然比原始的Python积分代码快约10倍。这显示了将整个循环从Python代码移动到Cython模块时，速度可以有多么快。

关于我们新推行的`evaluate`的一些注意事项：
* 这里的快速方法分派之所以有效，是因为`evaluate`是在`Functon`中声明的。如果在`SinOfSquareFunction`中引入`evaluate`，代码仍然可以工作，但是Cython将使用较慢的Python方法分派机制。
* 同样，如果没有类型化参数`f`，而只是作为Python对象传递，那么将使用较慢的Python分派。
* 由于参数是类型化的，我们需要检查它是否为`None`。在Python中，当查找`evaluate`方法时，这将导致AttributeError，但是Cython将尝试访问`None`（不兼容的）内部结构，就像它是一个`Function`一样，这会导致崩溃或数据损坏。

有一个*编译器指令*nonecheck，它以降低速度为代价开启检查。下面是如何用编译器指令来动态地打开或关闭`nonecheck`：
```cython
# cython: nonecheck=True
#        ^^^ 全局打开nonecheck

import cython

cdef class MyClass:
    pass

# 为该函数局部关闭nonecheck
@cython.nonecheck(False)
def func():
    cdef MyClass obj = None
    try:
        # 为该模块再次打开nonecheck
        with cython.nonecheck(True):
            print(obj.myfunc())  # 引发异常
    except AttributeError:
        pass
    print(obj.myfunc())  # 希望有一个崩溃！
```
属性在cdef类中的行为不同于在常规类中的属性：
* 所有属性必须在编译时预先声明
* 属性默认情况下只能通过Cython访问（类型化访问）
* 属性可以声明为向Python空间公开动态属性
```cython
from sin_of_square cimport Function

cdef class WaveFunction(Function):

    # Python空间中不可用：
    cdef double offset

    # Python空间中可用：
    cdef public double freq

    # Python空间中只读：
    cdef readonly double scale

    # Python空间中可用：
    @property
    def period(self):
        return 1.0 / self.freq

    @period.setter
    def period(self, value):
        self.freq = 1.0 / value
```