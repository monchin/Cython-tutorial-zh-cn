# 使用Python数组
Python有一个内置数组模块，支持基本类型的动态一维数组。可以从Cython内部访问Python数组的底层C数组。同时，它们是普通的Python对象，可以存储在列表中，并在使用<b>[多进程](https://docs.python.org/3/library/multiprocessing.html#module-multiprocessing)</b>时在进程之间进行序列化。

与使用<b>malloc()</b>和<b>free()</b>的手动管理方法相比，这为Python提供了安全且自动的内存管理，而且与Numpy数组相比，不需要安装依赖项，因为<strong>[数组](https://docs.python.org/3/library/array.html#module-array)</strong>模块内置在Python和Cython中。
## 安全使用内存视图
```cython
from cpython cimport array
import array
cdef array.array a = array.array('i', [1, 2, 3])
cdef int[:] ca = a

print(ca[0])
```
附注：import将常规Python数组对象带入名称空间，而cimport则添加可以从Cython访问的函数。

Python数组由类型签名和初始值的序列组成。有关可能的类型签名，请参阅[数组模块](https://docs.python.org/3/library/array.html)的Python文档。

请注意，当Python数组被分配给一个类型为内存视图的变量时，构造内存视图会有一点开销。但是，只要变量被类型化了，内存视图类型的变量可以无开销地传递给其他函数：
```cython
from cpython cimport array
import array

cdef array.array a = array.array('i', [1, 2, 3])
cdef int[:] ca = a

cdef int overhead(object a):
    cdef int[:] ca = a
    return ca[0]

cdef int no_overhead(int[:] ca):
    return ca[0]

print(overhead(a))  # 要建立新的内存视图，有开销
print(no_overhead(ca))  # ca已经是内存视图，无开销
```
## 对原始C指针的零开销不安全访问
为了零开销并能够将C指针传递给其他函数，可以将底层连续数组作为指针访问。要小心使用正确的类型和符号，因为没有类型或边界检查。
```cython
from cpython cimport array
import array

cdef array.array a = array.array('i', [1, 2, 3])

# 访问底层指针：
print(a.data.as_ints[0])

from libc.string cimport memset

memset(a.data.as_voidptr, 0, len(a) * sizeof(int))
```
注意，数组对象上的任何长度更改操作都可能使指针无效。
## 复制、扩展数组
为了避免使用Python模块中的数组构造函数，可以创建一个与模板类型相同的新数组，并预先分配给定数量的元素。数组在被请求时会初始化为零。
```cython
from cpython cimport array
import array

cdef array.array int_array_template = array.array('i', [])
cdef array.array newarray

# create an array with 3 elements with same type as template
newarray = array.clone(int_array_template, 3, zero=False)
```
数组还可以扩展和调整大小；这避免了当元素被一个接一个地添加或删除时会发生的内存不断重新分配的情况。
```cython
from cpython cimport array
import array

cdef array.array a = array.array('i', [1, 2, 3])
cdef array.array b = array.array('i', [4, 5, 6])

# extend a with b, resize as needed
array.extend(a, b)
# resize a, leaving just original three elements
array.resize(a, len(a) - len(b))
```
## API索引
### 数据层面
```
data.as_voidptr
data.as_chars
data.as_schars
data.as_uchars
data.as_shorts
data.as_ushorts
data.as_ints
data.as_uints
data.as_longs
data.as_ulongs
data.as_longlongs  # 需要Python >=3
data.as_ulonglongs  # 需要Python >=3
data.as_floats
data.as_doubles
data.as_pyunicodes
```
直接访问具有给定类型的底层连续C数组；例如，`myarray.data.as_ints`。
### 函数
Cython可以从数组模块中获得以下函数：
```cython
int resize(array self, Py_ssize_t n) except -1
```
快速调整大小/重分配内存。不适合重复的小增量；调整底层数组的大小，使其精确到请求的大小。
```cython
int resize_smart(array self, Py_ssize_t n) except -1
```
对小增量非常高效；使用提供平摊线性时间附加的增长模式。
```cython
cdef inline array clone(array template, Py_ssize_t length, bint zero)
```
快速创建一个新的数组，给定一个模板数组。类型将与`template`相同。如果`zero`为`True`，则用零初始化新数组。
```cython
cdef inline array copy(array self)
```
复制一个数组。
```cython
cdef inline int extend_buffer(array self, char* stuff, Py_ssize_t n) except -1
```
高效地附加相同类型的新数据（例如，相同数组类型） `n`：元素的数量（不是字节的数量！）
```cython
cdef inline int extend(array self, array other) except -1
```
用另一个数组中的数据扩展数组；类型必须匹配。
```cython
cdef inline void zero(array self)
```
把数组中所有元素设为0。