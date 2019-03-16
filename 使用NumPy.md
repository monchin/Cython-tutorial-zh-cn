# 使用NumPy

注意：`Cython 0.16`引入了typed memoryviews作为这里描述的NumPy集成的继承者。它们比下面的缓冲区语法更容易使用，开销也更少，并且可以在不需要GIL的情况下传递。它们应该优于本页面中显示的语法。请看[为NumPy用户准备的Cython](https://cython.readthedocs.io/en/latest/src/userguide/numpy_tutorial.html#numpy-tutorial)。

你可以使用与常规Python完全一致的使用方法来在Cython中使用NumPy，但这样的话你会丢掉一些潜在的加速效果，因为Cython已经支持对NumPy数组的快速访问了。让我们用一个小例子来看看应该怎么操作。

下面的代码使用滤波器对一幅图像做了2D离散卷积（我相信你可以做得更好，这只是一个示范）。这段代码在Python和Cython中都有效。我将以`convolve_py.py`作为Python版本的名字，而以`convolve1.pyx`作为Cython版本的名字——Cython用“`.pyx`”作为文件扩展名。

```python
import numpy as np


def naive_convolve(f, g):
    # f是图像，以(v, w)索引
    # g是卷积核，以(s, t)索引，维数为奇数
    # h是输出图像，以(x, y)索引，未被裁剪
    if g.shape[0] % 2 != 1 or g.shape[1] % 2 != 1:
        raise ValueError("Only odd dimensions on filter supported")
    # smid和tmid是中心像素和边缘像素之间的像素数，比如对5x5的卷积核，smid和tmid
    # 就是2
    # 输出图像尺寸的计算方法是输入图像的维数加上每条边的smid、tmid
    vmax = f.shape[0]
    wmax = f.shape[1]
    smax = g.shape[0]
    tmax = g.shape[1]
    smid = smax // 2
    tmid = tmax // 2
    xmax = vmax + 2 * smid
    ymax = wmax + 2 * tmid
    # 给结果分配空间
    h = np.zeros([xmax, ymax], dtype=f.dtype)
    # 卷积
    for x in range(xmax):
        for y in range(ymax):
            # 计算坐标为(x,y)的h的像素值。对滤波器g的每个像
            # 素(s, t)求和一个分量。
            s_from = max(smid - x, -smid)
            s_to = min((xmax - x) - smid, smid + 1)
            t_from = max(tmid - y, -tmid)
            t_to = min((ymax - y) - tmid, tmid + 1)
            value = 0
            for s in range(s_from, s_to):
                for t in range(t_from, t_to):
                    v = x - smid + s
                    w = y - tmid + t
                    value += g[smid - s, tmid - t] * f[v, w]
            h[x, y] = value
    return h
```

这个文件在Linux系统下将会被编译生成yourmod.so，在Windows系统下将会生成yourmod.pyd。我们运行一个Python会话来测试Python版本(从.py文件导入)和编译好的Cython模块。

```python
In [1]: import numpy as np
In [2]: import convolve_py
In [3]: convolve_py.naive_convolve(np.array([[1, 1, 1]], dtype=np.int),
...     np.array([[1],[2],[1]], dtype=np.int))
Out [3]:
array([[1, 1, 1],
    [2, 2, 2],
    [1, 1, 1]])
In [4]: import convolve1
In [4]: convolve1.naive_convolve(np.array([[1, 1, 1]], dtype=np.int),
...     np.array([[1],[2],[1]], dtype=np.int))
Out [4]:
array([[1, 1, 1],
    [2, 2, 2],
    [1, 1, 1]])
In [11]: N = 100
In [12]: f = np.arange(N*N, dtype=np.int).reshape((N,N))
In [13]: g = np.arange(81, dtype=np.int).reshape((9, 9))
In [19]: %timeit -n2 -r3 convolve_py.naive_convolve(f, g)
2 loops, best of 3: 1.86 s per loop
In [20]: %timeit -n2 -r3 convolve1.naive_convolve(f, g)
2 loops, best of 3: 1.41 s per loop
```
差别并不大，因为C代码仍然执行Python解释器所执行的操作(例如，为使用的每个数字分配一个新对象)。查看生成的html文件，看看即使是最简单的语句也需要做些什么，你很快就会明白这一点。我们需要给Cython更多的信息，我们需要添加类型。

## 添加类型

为了添加类型，我们使用定制的Cython语法，所以我们现在正在破坏Python源代码的兼容性。考虑以下代码(请阅读注释)：
```python
# tag: numpy_old
# 你可以忽略这几行
# 它用于cython文档的内部测试

import numpy as np

# “cimport”用于导入关于numpy模块的特殊编译时信息
# 它存储在一个叫做numpy.pxd的文件中，这个文件是目前Cython
# 分布的一部分
cimport numpy as np

# 我们需要固定数组的数据类型。我为此使用了变量DTYPE
# 它被分配给通常numpy运行时的info对象
DTYPE = np.int

# “ctypedef”将相应的编译时类型分配给DTYPE_t。对于numpy模块中
# 的每一个类型，都有一个对应的编译时类型，并带有_t后缀。
ctypedef np.int_t DTYPE_t

# “def”可以输入它的参数，但没有返回类型。一个“def”函数的参数
# 类型会在运行时进入这个函数时被检查。
#
# 数组f、g和h的类型为“np.ndarray”实例。它仅有的效果是：a）插
# 入检查函数参数确实是NumPy数组，以及b）使类似f.shape[0]这样
# 的属性访问更有效率。（虽然这个例子中这点并没有什么影响）
def naive_convolve(np.ndarray f, np.ndarray g):
    if g.shape[0] % 2 != 1 or g.shape[1] % 2 != 1:
        raise ValueError("Only odd dimensions on filter supported")
    assert f.dtype == DTYPE and g.dtype == DTYPE

    # “cdef”关键字也用于函数中定义变量类型。它只能在顶部标识
    # 层使用（在其他地方使用的话会出一些问题，尽管我们希望能
    # 在其他地方也可以使用，也提出了一些方案）。
    #
    # 对于索引，使用“int”类型。这对应于一个C int，其他C类型
    # (如“unsigned int”)也可以使用。纯粹主义者可以使用
    # “Py_ssize_t”，这是数组索引的正确Python类型。 
    cdef int vmax = f.shape[0]
    cdef int wmax = f.shape[1]
    cdef int smax = g.shape[0]
    cdef int tmax = g.shape[1]
    cdef int smid = smax // 2
    cdef int tmid = tmax // 2
    cdef int xmax = vmax + 2 * smid
    cdef int ymax = wmax + 2 * tmid
    cdef np.ndarray h = np.zeros([xmax, ymax], dtype=DTYPE)
    cdef int x, y, s, t, v, w

    # 为所有变量定义类型是非常重要的。如果没有这样做也不会得
    # 到任何警告，只是会得到更慢的代码(它们被隐式地类型化为
    # Python对象)。
    cdef int s_from, s_to, t_from, t_to

    # 对于value变量，我们希望使用与数组中存储的相同的数据类
    # 型，因此我们使用上面定义的“DTYPE_t”。注意！这样做的一个
    # 重要副作用是，如果“value”溢出了，它就会像C语言那样简单
    # 地溢出，而不会像Python那样引发错误。
    cdef DTYPE_t value
    for x in range(xmax):
        for y in range(ymax):
            s_from = max(smid - x, -smid)
            s_to = min((xmax - x) - smid, smid + 1)
            t_from = max(tmid - y, -tmid)
            t_to = min((ymax - y) - tmid, tmid + 1)
            value = 0
            for s in range(s_from, s_to):
                for t in range(t_from, t_to):
                    v = x - smid + s
                    w = y - tmid + t
                    value += g[smid - s, tmid - t] * f[v, w]
            h[x, y] = value
    return h
```
我们得到结果：
```python
In [21]: import convolve2
In [22]: %timeit -n2 -r3 convolve2.naive_convolve(f, g)
2 loops, best of 3: 828 ms per loop
```
## 高效索引
还有一个拖累性能的瓶颈，就是数组的查找和赋值。[]运算符仍然在使用Python的操作，而我们希望以C的速度直接访问数据缓冲区。

我们需要做的是为ndarray对象的内容指定类型。我们使用一个特殊的“buffer”语法来实现这一点，它必须被告知数据类型(第一个参数)和维数(“ndim”关键字参数，如果没有提供，认为是一维的)。

以下是需要的更改：
```python
...
def naive_convolve(np.ndarray[DTYPE_t, ndim=2] f, np.ndarray[DTYPE_t, ndim=2] g):
...
cdef np.ndarray[DTYPE_t, ndim=2] h = ...
```
使用：
```python
In [18]: import convolve3
In [19]: %timeit -n3 -r100 convolve3.naive_convolve(f, g)
3 loops, best of 100: 11.6 ms per loop
```
注意此次更改的重要性。

知识点：这种高效的索引只影响某些索引操作，即具有精确的ndim数的指定类型后的整数索引数的索引操作。因此，如果没有指定v的类型，那么查找f[v, w]就没有优化。另一方面，这意味着你可以继续使用Python对象进行复杂的动态切片等，就像数组没有被指定类型时一样。
## 进一步优化索引
数组查找依然被两个因素拖慢了：

1. 执行了边界检查；
2. 负索引的检查和正确处理。上面的代码是显式编码的，因此它不使用负索引，并且(希望)总是在一定范围内访问。我们可以添加一个装饰器来禁用边界检查:
```python
...
cimport cython
@cython.boundscheck(False) # 对整个函数，关闭边界检查
@cython.wraparound(False)  # 对整个函数，关闭负索引包装
def naive_convolve(np.ndarray[DTYPE_t, ndim=2] f, np.ndarray[DTYPE_t, ndim=2] g):
...
```
现在不执行边界检查了(副作用是，如果你的访问确实超出了界限，那么在最好的情况下，你的程序将崩溃，而在最坏的情况下，你的数据将被破坏)。可以通过多种方式切换边界检查模式，有关更多信息，请查阅[编译器指令](https://cython.readthedocs.io/en/latest/src/userguide/source_files_and_compilation.html#compiler-directives)。

此外，我们还禁用了对负索引包装的检查(例如，给出最后一个值的g[-1])。与禁用边界检查一样，如果我们在禁用这个检查之后还试图使用负索引，就会发生不好的事情。

函数调用开销现在开始发挥作用，所以我们使用更大的N来对后两个例子进行比较:
```python
In [11]: %timeit -n3 -r100 convolve4.naive_convolve(f, g)
3 loops, best of 100: 5.97 ms per loop
In [12]: N = 1000
In [13]: f = np.arange(N*N, dtype=np.int).reshape((N,N))
In [14]: g = np.arange(81, dtype=np.int).reshape((9, 9))
In [17]: %timeit -n1 -r10 convolve3.naive_convolve(f, g)
1 loops, best of 10: 1.16 s per loop
In [18]: %timeit -n1 -r10 convolve4.naive_convolve(f, g)
1 loops, best of 10: 597 ms per loop
```
（这也是一个混合基准测试，因为结果数组是在函数调用中分配的。）

<table><tr><td bgcolor=#FF7F50>
警告：速度需要一些代价，特别是将指定类型的对象（如示例代码中的f、g和h）设置为None可能会很危险。将这些对象设置为None是完全合法的，但是您只能检查它们是否为None。所有其他的使用(属性查找或索引)都可能导致段错误或数据损坏（而不是像在Python中那样引发异常）。

实际的规则稍微复杂一些，但是主要的信息很清楚：如果不知道没有将指定类型的对象设置为None，就不要使用指定类型的对象。
</td></tr></table>

## 更通用的代码
你可以：
```python
def naive_convolve(object[DTYPE_t, ndim=2] f, ...):
```
即使用object而不是np.ndarray。在Python 3.0下，这允许你的算法应用于任何支持缓冲区接口的库；如果有人对Python 2.x也感兴趣，可以很容易地添加对比如Python Imaging Library（PIL）的支持。

这样做会有一些速度损失(因为如果类型设置为np.ndarray，就会有更多的编译时假设，具体来说，它假定数据以strided模式存储，而不是以间接模式存储)。