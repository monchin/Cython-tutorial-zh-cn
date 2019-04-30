# Unicode和传入字符串
与python3中的字符串语义类似，Cython严格分隔字节字符串和unicode字符串。最重要的是，这意味着默认情况下，字节字符串和unicode字符串之间没有自动转换(Python 2在字符串操作中所做的除外)。所有的编码和解码都必须经过一个显式的编码/解码步骤。为了在简单的情况下简化Python和C字符串之间的转换，可以使用模块级的`c_string_type`和`c_string_encoding`指令隐式地插入这些编码/解码步骤。
## Cython代码中的Python字符串类型
Cython支持四种Python字符串类型:[`bytes`](https://docs.python.org/3/library/stdtypes.html#str)、[`str`](https://docs.python.org/3/library/stdtypes.html#str)、`unicode`和`basestring`。`bytes`和`unicode`类型是普通Python 2中已知的特定类型（Python 3中就是`bytes`和`str`）。此外，Cython还支持`字节数组`类型，它类似于`bytes`类型，只是它是可变的。

`str`类型的特殊之处在于它是Python 2中的字节字符串和Python 3中的Unicode字符串(对于使用语言级别2编译的Cython代码，即默认值)。也就是说，它总是与Python运行时本身调用str的类型完全对应。因此，在Python 2中，`bytes`和`str`都表示字节字符串类型，而在Python 3中，`str`和`unicode`都表示Python Unicode字符串类型。切换是在C编译时进行的，用于运行Cython的Python版本无关紧要。

当以语言级别3来编译Cython代码时，`str`类型在Cython编译时被精确地标识为Unicode字符串类型，也就是说，即使在Python 2中运行时它也不会被标识为`bytes`。

请注意，`str`类型与Python 2中的`unicode`类型不兼容，也就是说，你不能将Unicode字符串分配给类型化为`str`的变量或参数。这种尝试将导致编译时错误(如果检测到)或运行时`TypeError`异常。因此，当你在必须与Python 2兼容的代码中类型化字符串变量时，你应该非常小心，因为这个Python版本允许数据混合使用字节字符串和unicode字符串，而且用户通常希望代码能够同时使用这两种字符串。只面向Python 3的代码可以安全地将变量和参数类型化为`bytes`或是`unicode`。

`basestring`类型同时表示`str`和`unicode`类型，即Python 2和Python 3中的所有Python文本字符串类型。这可以用于输入通常包含Unicode文本的文本变量(至少在Python 3中)，但是由于向后兼容的原因，必须在Python 2中接受`str`类型。它与`bytes`类型不兼容。它一般在Cython代码中应该很少使用，因为泛型对象类型(即非类型化代码)通常足够好，并且具有支持字符串子类型分配的额外优势。在Cython 0.20中添加了对`basestring`类型的支持。
## 字符串文法
Cython可以理解所有的Python字符串类型前缀：

* `b'bytes'` 字节字符串
* `u'text'` Unicode字符串
* `f'formatted {value}'` 格式化的Unicode字符串文法，在[PEP498](https://www.python.org/dev/peps/pep-0498/)中定义（Cython0.24中添加） 

对于没有前缀的字符串文法，在语言等级2时会被编译为`str`对象，而在语言等级3时会被编译为`unicode`对象（即Python 3 `str`）。
## C字符串的注意点
在许多实例中，C字符串(也就是字符指针)很慢，而且很麻烦。首先，它们通常需要以这样或那样的方式手工进行内存管理，这就更有可能在代码中引入bug。

其次，Python字符串对象缓存它们的长度，因此请求它(例如验证索引访问的界限或将两个字符串连接成一个字符串时)是一种有效的常量时间操作。相反，调用`strlen()`从C字符串中获取此信息需要线性时间，这使得对C字符串的许多操作的成本会很高。

在文本处理方面，Python内置了对Unicode的支持，而C完全不支持Unicode。如果要处理Unicode文本，通常使用Python Unicode字符串对象比使用C字符串编码的数据要好。Cython使这变得非常简单有效。

一般来说，除非你知道自己在做什么，否则尽量避免使用C字符串，而应使用Python字符串对象。一个明显的例外是从或向外部C代码来回传递字符串时。此外，C++字符串还记得它们的长度，因此在某些情况下，它们可以提供Python `bytes`对象的合适替代，例如在定义良好的上下文中不需要引用计数时。
## 传入字节字符串
我们在一个名为`c_func.pyx`的文件中声明了伪C函数。之后的这篇教程会反复使用这个`c_func.pyx`文件：
```cython
from libc.stdlib cimport malloc
from libc.string cimport strcpy, strlen

cdef char* hello_world = 'hello world'
cdef Py_ssize_t n = strlen(hello_world)


cdef char* c_call_returning_a_c_string():
    cdef char* c_string = <char *> malloc((n + 1) * sizeof(char))
    if not c_string:
        raise MemoryError()
    strcpy(c_string, hello_world)
    return c_string


cdef void get_a_c_string(char** c_string_ptr, Py_ssize_t *length):
    c_string_ptr[0] = <char *> malloc((n + 1) * sizeof(char))
    if not c_string_ptr[0]:
        raise MemoryError()

    strcpy(c_string_ptr[0], hello_world)
    length[0] = n
```
我们写了一个对应的`c_func.pxd`文件使我们可以cimport这些函数：
```cython
cdef char* c_call_returning_a_c_string()
cdef void get_a_c_string(char** c_string, Py_ssize_t *length)
```
在C代码和Python之间传递字节字符串非常容易。当从C库接收一个字节字符串时，你可以让Cython将其转换为Python字节字符串，只需将其分配给一个Python变量：
```cython
from c_func cimport c_call_returning_a_c_string

cdef char* c_string = c_call_returning_a_c_string()
cdef bytes py_string = c_string
```
类型转换为`对象`或`bytes`也会做同样的事情：
```cython
py_string = <bytes> c_string
```
这将创建一个Python字节字符串对象，该对象持有原始C字符串的副本。它可以在Python代码中安全地传递，当最后一个对它的引用超出范围时，它将被垃圾收集。重要的是要记住，C中都知道的，字符串中的空字节充当终止符。因此，上面的方法只适用于不包含空字节的C字符串。

除了不处理空字节之外，对于长字符串，上面的方法也非常低效，因为Cython必须首先调用C字符串上的strlen()，通过计算字节数直到结束空字节，来找出长度。在许多情况下，用户代码已经知道了长度。在本例中，通过给C字符串切片告诉Cython确切的字节数要有效率得多。举个例子：
```cython
from libc.stdlib cimport free
from c_func cimport get_a_c_string


def main():
    cdef char* c_string = NULL
    cdef Py_ssize_t length = 0

    # 从C函数中得到指针和长度
    get_a_c_string(&c_string, &length)

    try:
        py_bytes_string = c_string[:length]  # 执行对数据的复制
    finally:
        free(c_string)
```
这里不需要额外的字节计数，`c_string`中的`length`字节将被复制到Python bytes对象中，包括任何空字节。请记住，在这种情况下，切片索引会被假定是准确的，并且不会执行边界检查，因此不正确的切片索引将导致数据损坏和崩溃。

注意，Python字节字符串的创建可能会因为异常而导致失败，比如内存不足。如果你需要在转换后`free()`字符串，你应该用try-finally结构封装赋值：
```cython
from libc.stdlib cimport free
from c_func cimport c_call_returning_a_c_string

cdef bytes py_string
cdef char* c_string = c_call_returning_a_c_string()
try:
    py_string = c_string
finally:
    free(c_string)
```
若要将字节字符串转换回C char*，请使用相反的赋值:
```cython
cdef char* other_c_string = py_string  # other_c_string是一个以0结尾的字符串
```
这是一个非常快速的操作，之后`other_c_string`指向Python字符串本身的字节字符串缓冲区。它与Python字符串的生命周期相关联。当Python字符串被垃圾收集时，指针将失效。因此，只要使用`char*`，就必须保持对Python字符串的引用。通常，这只是将调用扩展到接收指针作为参数的C函数。但是，当C函数存储指针供以后使用时，必须特别注意。除了保存对字符串对象的Python引用外，不需要手动内存管理。

从Cython 0.20开始，支持[`字节数组`](https://docs.python.org/3/library/stdtypes.html#bytearray)类型，并以与`bytes`类型相同的方式强制执行。但是，当在C上下文中使用它时，必须特别注意不要在将对象缓冲区转换为C字符串指针后对其进行增长或收缩。这些修改可以更改内部缓冲区地址，这将使指针无效。
## 从Python代码中接收字符串
另一边，接收来自Python代码的输入，乍一看可能很简单，因为它只处理对象。然而，在不使API太窄或太不安全的情况下正确地实现这一点可能并不十分明显。

如果API只处理字节字符串，即二进制数据或编码的文本，最好不要让输入参数成为`bytes`之类的东西，因为这将精确限制允许输入类型，而排除亚型和其他类型的字节容器，例如中[`bytearray`](https://docs.python.org/3/library/stdtypes.html#bytearray)对象或内存视图。

根据处理数据的方式(和位置)，接收一维内存视图可能是一个好主意，例如：
```cython
def process_byte_data(unsigned char[:] data):
    length = data.shape[0]
    first_byte = data[0]
    slice_view = data[1:-1]
    # ...
```
Cython的内存视图在[类型化内存视图](https://github.com/monchin/Cython-tutorial-zh-cn/blob/master/%E7%B1%BB%E5%9E%8B%E5%8C%96%E5%86%85%E5%AD%98%E8%A7%86%E5%9B%BE.md)中有更详细的描述，但是上面的示例已经显示了一维字节视图的大部分相关功能。它们允许有效地处理数组，并接受任何可以解压到字节缓冲区中的内容，而不需要进行中间复制。经过处理的内容最终可以在内存视图本身（或其中的一个切片）中返回，但是通常最好将数据复制回一个平整且简单的`bytes`或`bytearray`对象中，特别是只返回一个小切片时。因为内存视图不复制数据，它们会保持整个初始缓冲区处于活动状态。这里的一般思想是通过接受任何类型的字节缓冲区来自由输入，但是通过返回一个简单的、适应良好的对象来严格输出。这可以简单地如下做到：
```cython
def process_byte_data(unsigned char[:] data):
    # ... 处理数据，在这里假处理
    cdef bint return_all = (data[0] == 108)

    if return_all:
        return bytes(data)
    else:
        # 返回一个切片的例子
        return bytes(data[5:7])
```
如果字节输入实际上是经过编码的文本，并且进一步的处理应该在Unicode级别进行，那么正确的做法是直接解码输入。这几乎只是Python 2.x中的一个问题，在Python 2.x中Python代码期望能够将带有编码文本的字节字符串（str）传递到文本API中。由于这通常发生在模块的API中的多个位置，所以一个辅助函数几乎总是正确的选择，因为它允许在后续操作中轻松地调整输入规范化函数。

这类输入规范化函数一般类似如下：
```cython
# to_unicode.pyx

from cpython.version cimport PY_MAJOR_VERSION

cdef unicode _text(s):
    if type(s) is unicode:
        # 对于多数情况(s)的快捷路径
        return <unicode>s

    elif PY_MAJOR_VERSION < 3 and isinstance(s, bytes):
        # 只在Python 2.x中接受字节字符串作为文本输入，不在Py3中
        return (<bytes>s).decode('ascii')

    elif isinstance(s, unicode):
        # 我们从上面的快捷路径中知道's'在这里只能作为亚型。
        # 根据之后进一步的操作，在某些情况下到<unicode>
        # 的操作可能仍会起作用。安全起见，我们可以总是创建一个副本。
        return unicode(s)

    else:
        raise TypeError("Could not convert to unicode.")
```
它应该以如下方式被使用：
```
from to_unicode cimport _text

def api_func(s):
    text_input = _text(s)
    # ...
```
类似地，如果进一步的处理发生在字节等级，但是Unicode字符串输入应该被接受，那么如果你在使用内存视图，下面的操作可能会有效：
Similarly, if the further processing happens at the byte level, but Unicode string input should be accepted, then the following might work, if you are using memory views:
```cython
# 为这个模块中使用的任意一个char类型定义一个全局的名称
ctypedef unsigned char char_type

cdef char_type[:] _chars(s):
    if isinstance(s, unicode):
        # 编码到模块内部使用的特定编码
        s = (<unicode>s).encode('utf8')
    return s
```
在这种情况下，您可能还需要确保字节字符串输入确实使用了正确的编码，例如，如果需要纯ASCII输入数据，可以在循环中遍历缓冲区并检查每个字节的最高位。这也应该在输入规范化函数中完成。
## 处理“const”
许多C库在API中使用`const`修饰符来声明它们不会修改字符串，或者要求用户不能修改它们返回的字符串，例如：
```c
typedef const char specialChar;
int process_string(const char* s);
const unsigned char* look_up_cached_string(const unsigned char* key);
```
Cython支持语言中的`const`修饰符，所以你可以直接声明上面的函数如下：
```cython
cdef extern from "someheader.h":
    ctypedef const char specialChar
    int process_string(const char* s)
    const unsigned char* look_up_cached_string(const unsigned char* key)
```
## 将bytes解码到文本
如果你的代码只处理字符串中的二进制数据，那么最初介绍的传递和接收C字符串的方法就足够了。然而，当我们处理编码文本时，最好的做法是在接收时将C字节字符串解码为Python Unicode字符串，在输出时将Python Unicode字符串编码为C字节字符串。

对于Python字节字符串对象，通常只需调用`bytes.decode()`方法将其解码为Unicode字符串：
```cython
ustring = byte_string.decode('UTF-8')
```
Cython允许你对一个C字符串执行相同的操作，只要它不包含空字节：
```cython
from c_func cimport c_call_returning_a_c_string

cdef char* some_c_string = c_call_returning_a_c_string()
ustring = some_c_string.decode('UTF-8')
```
更有效的是，对于已知长度的字符串：
```cython
from c_func cimport get_a_c_string

cdef char* c_string = NULL
cdef Py_ssize_t length = 0

# 从C函数中获取指针和长度
get_a_c_string(&c_string, &length)

ustring = c_string[:length].decode('UTF-8')
```
当字符串包含空字节时也应该使用相同的方法，例如，当它使用UCS-4这样的编码时，每个字符都被编码为四个字节，其中大多数都趋向于0。

同样，如果提供了切片索引，则不执行边界检查，因此不正确的索引会导致数据损坏和崩溃。不过，可以使用负索引，并且这样将产生对`strlen()`的调用以确定字符串长度。显然，这只适用于没有内部空字节的以0结尾的字符串。用UTF-8或ISO-8859编码的文本通常是一个很好的候选。如果不确定，最好是传入“明显”正确的索引，而不是依赖于预期的数据指望数据按照预期表现。

在专用函数中封装字符串转换(以及一般的非平凡类型转换)是一种常见的做法，因为无论何时接收来自C的文本，这都需要以完全相同的方式完成。如下所示：
```cython
from libc.stdlib cimport free

cdef unicode tounicode(char* s):
    return s.decode('UTF-8', 'strict')

cdef unicode tounicode_with_length(
        char* s, size_t length):
    return s[:length].decode('UTF-8', 'strict')

cdef unicode tounicode_with_length_and_free(
        char* s, size_t length):
    try:
        return s[:length].decode('UTF-8', 'strict')
    finally:
        free(s)
```
最可能的情况是，根据所处理的字符串类型，你会偏好在代码中使用更短的函数名。不同类型的内容通常意味着不同的接收处理方式。为了使代码更具可读性并预测将来的更改，最好为不同类型的字符串使用单独的转换函数。
## 将文本编码为bytes
反过来，将Python unicode字符串转换为C char*本身是非常高效的，假设你想要的是一个内存管理的字节字符串：
```cython
py_byte_string = py_unicode_string.encode('UTF-8')
cdef char* c_string = py_byte_string
```
如前所述，这将获取指向Python字节字符串的字节缓冲区的指针。尝试在不保留对Python字节字符串的引用的情况下执行相同的操作将会失败，并出现编译错误：
```cython
# 这将不会编译！
cdef char* c_string = py_unicode_string.encode('UTF-8')
```
在这里，Cython编译器注意到代码接受一个指向临时字符串结果的指针，该字符串结果将在赋值之后被垃圾收集。稍后对无效指针的访问将读取无效内存，并可能导致段错误。因此Cython将拒绝编译此代码。
## C++字符串
在包装c++库时，字符串通常以`std::string`类的形式出现。与C字符串一样，Python字节字符串自动强制从和到c++字符串：  
```cython
# distutils: language = c++

from libcpp.string cimport string

def get_bytes():
    py_bytes_object = b'hello world'
    cdef string s = py_bytes_object

    s.append('abc')
    py_bytes_object = s
    return py_bytes_object
```
内存管理的情况与C中不同，因为C++字符串的创建产生一个了字符串对象拥有的字符串缓冲区的独立副本。因此，可以将临时创建的Python对象直接转换为C++字符串。一种常用的方法是将Python unicode字符串编码为C++字符串：
```cython
cdef string cpp_string = py_unicode_string.encode('UTF-8')
```
注意，这涉及到一点开销，因为它首先将Unicode字符串编码到临时创建的Python bytes对象中，然后将其缓冲区复制到一个新的C++字符串中。

另一方面，Cython 0.17及以后版本提供了高效的解码支持：
```cython
# distutils: language = c++

from libcpp.string cimport string

def get_ustrings():
    cdef string s = string(b'abcdefg')

    ustring1 = s.decode('UTF-8')
    ustring2 = s[2:-2].decode('UTF-8')
    return ustring1, ustring2
```
对于C++字符串，解码切片总是考虑到字符串的适当长度，并应用Python切片语义（例如，为越界索引返回空字符串）。
## 自动编码和解码
Cython 0.19附带了两个新指令：`c_string_type`和`c_string_encoding`。它们可用于将Python字符串类型强制更改为C/C++字符串，或相反。默认情况下，它们只能强制更改bytes类型，编码或解码必须显式执行，如上所述。

有两个用例不方便这样做。第一个，如果正在处理的所有C字符串(或大部分)都包含文本，那么自动编码为Python unicode对象或从Python unicode对象自动解码可以稍微减少代码开销。在这种情况下，可以将模块中的`c_string_type`指令设置为unicode，将`c_string_encoding`设置为C代码使用的编码，例如：
```cython
# cython: c_string_type=unicode, c_string_encoding=utf8

cdef char* c_string = 'abcdefg'

# 隐式解码：
cdef object py_unicode_object = c_string

# 显式转换为Python bytes:
py_bytes_object = <bytes>c_string
```
第二个用例是，所有正在处理的C字符串都只包含ASCII可编码字符(例如数字)，并且你希望你的代码使用Python 2中的原生遗留字符串类型，而不是总是使用Unicode。在这种情况下，可以将字符串类型设置为`str`：
```cython
# cython: c_string_type=str, c_string_encoding=ascii

cdef char* c_string = 'abcdefg'

# 在Py3中隐式解码，在Py2中转换bytes
cdef object py_str_object = c_string

# 显式转换为Python bytes:
py_bytes_object = <bytes>c_string

# 显式转换为Python unicode:
py_bytes_object = <unicode>c_string
```
另一个指令，即自动编码到C字符串，只支持ASCII和“默认编码”，在Python 3中通常是UTF-8，在Python 2中通常是ASCII。在本例中，CPython通过将字符串的编码副本与原始unicode字符串一起保持活动状态来处理内存管理。否则，将无法以任何合理的方式限制已编码字符串的生命周期，从而使从其中提取C字符串指针的任何尝试都成为危险的尝试。以下代码安全地将Unicode字符串转换为ASCII（将`c_string_encoding`更改为`default`以使用默认编码）：
```cython
# cython: c_string_type=unicode, c_string_encoding=ascii

def func():
    ustring = u'abc'
    cdef char* s = ustring
    return s[0]    # returns u'a'
```
（本例使用函数上下文来安全地控制Unicode字符串的生命周期。全局Python变量可以从外部修改，这使得对它们的值的生命周期的依赖变得危险。）
## 源代码编码
当字符串文本出现在代码中时，源代码编码非常重要。它确定Cython在C代码中将要储存的字节序列（用于字节文本），以及Cython在解析字节编码的源文件时为Unicode文本构建的Unicode代码点。在[**PEP 263**](https://www.python.org/dev/peps/pep-0263/)之后，Cython支持显式声明源文件编码。例如，要在解析器中启用`ISO-8859-15`解码，需要在`ISO-8859-15 `(Latin-9)编码源文件的顶部（第一行或第二行)放置以下注释:
```cython
# -*- coding: ISO-8859-15 -*-
```
当没有提供显式编码声明时，源代码将被解析为UTF-8编码的文本，如[<b>PEP 3120</b>](https://www.python.org/dev/peps/pep-3120/)所指定的那样。[UTF-8](https://en.wikipedia.org/wiki/UTF-8)是一种非常常见的编码方式，它可以表示整个Unicode字符集，并且与高效编码的纯ASCII编码文本兼容。对于通常由ASCII字符组成的源代码文件，这是一个非常好的选择。

例如，将下面一行放入UTF-8编码的源文件中会打印出`5`，因为UTF-8将字母`“o”`编码为两个字节序列`“\xc3\xb6”`：
```cython
print( len(b'abcö') )
```
而以`ISO-8859-15`编码的源文件将打印`4`，因为编码只为这个字母使用了1个字节：
```cython
# -*- coding: ISO-8859-15 -*-
print( len(b'abcö') )
```
注意，在以上两种情况下，unicode文本`u'abcö'`都是一个被正确解码的四个字符的unicode字符串，而没有前缀的Pyton str文本的`abcö`在Python 2中会成为一个字节字符串（因此在上面的例子中有4或5的长度），而在Python 3中会成为一个4字符的Unicode字符串。如果你对编码不熟悉，第一次阅读时可能这里并不容易理解。详见[CEP 108](https://github.com/cython/cython/wiki/enhancements-stringliterals)。

根据经验，最好避免无前缀的非ASCII **str**文本，并对所有文本使用unicode字符串文本。Cython还支持`__future__`导入`unicode_literals`，它指示解析器将源文件中所有不带前缀的**str**文本读取为unicode字符串文本，就像Python 3一样。
## 单bytes和字符
Python C-API使用普通的C **char**类型来表示一个字节值，但是对于Unicode代码点值，它有两种特殊的整数类型，即一个单Unicode字符：**Py_UNICODE**和**Py_UCS4**。Cython本身支持第一个，而从Cython 0.15开始支持**Py_UCS4**。**Py_UNICODE**可以定义为无符号的2字节或4字节整数，也可以定义为**wchar_t**，具体取决于平台。确切的类型是CPython解释器构建中的编译时选项，扩展模块在C编译时继承这个定义。**Py_UCS4**的优点是，无论使用什么平台，它都保证足够大，可以容纳任何Unicode代码点值。它被定义为32位无符号整型或长整型。

在Cython中，强制使用Python对象时，**char**类型的行为与**Py_UNICODE**和**Py_UCS4**类型不同。与Python 3中字节类型的行为类似，**char**类型默认强制为Python整数值，因此下面的输出是65而不是`A`：
```cython
# -*- coding: ASCII -*-

cdef char char_val = 'A'
assert char_val == 65   # 'A'的ASCII编码byte值
print( char_val )
```
如果想要一个Python字节字符串，你必须显式地请求它，下面将打印`A`(或Python 3中的`b'A'`)：
```cython
print( <bytes>char_val )
```
显式强制适用于任何C整数类型。超出**char**或**unsigned char**范围的值将在运行时引发**OverflowError**。分配给一个类型化的变量时，强制也会自动发生，例如：
```cython
cdef bytes py_byte_string
py_byte_string = char_val
```
另一方面，**Py_UNICODE**和**Py_UCS4**类型很少在Python unicode字符串上下文之外使用，因此它们的默认行为是强制使用Python unicode对象。所以下面将打印字符`A`，而类型为**Py_UNICODE**时输出也是一致的：
```cython
cdef Py_UCS4 uchar_val = u'A'
assert uchar_val == 65 # character point value of u'A'
print( uchar_val )
```
同样，显式强制转换将允许用户覆盖此行为。以下将列印65：
```cython
cdef Py_UCS4 uchar_val = u'A'
print( <long>uchar_val )
```
注意，强制转换到C **long**（或**unsigned long**）将正常工作，因为Unicode字符可以拥有的最大代码点值是1114111 （`0x10FFFF`）。在32位或32位以上的平台上，**int**也可以正常工作。
## 窄Unicode构建
在3.3版之前的CPython的窄Unicode构建中，即在`sys.maxunicode`是65535（例如所有Windows版本，而不是宽版本中的1114111）的构建中，仍然可以使用不适合16位宽**Py_UNICODE**类型的Unicode字符代码点。例如，这样的CPython构建将接受unicode文字`u'\U00012345'`。但是，在本例中，底层系统级别的编码泄漏到Python空间中，因此这个文字的长度变为2而不是1。这也显示了什么时候遍历它或者什么时候索引它。在本例中，可见的子字符串是`u'\uD808'`和`u'\uDF45'`。它们形成一个表示上述字符的所谓代理对。

关于这个主题的更多信息，请阅读[Wikipedia关于UTF-16编码的文章](https://en.wikipedia.org/wiki/UTF-16/UCS-2)。

同样的属性也适用于为窄CPython运行时环境编译的Cython代码。在大多数情况下，例如，当搜索子字符串时，这种差异可以忽略，因为文本和子字符串都包含代理。因此，大多数Unicode处理代码在窄版本上也能正确工作。编码、解码和打印将按预期工作，因此上面的文字在窄Unicode和宽Unicode平台上都可以转换成完全相同的字节序列。

但是，程序员应该意识到，单**Py_UNICODE**值（或者CPython中的单“字符”unicode字符串）可能不足以在窄平台上表示一个完整的Unicode字符。例如，如果在unicode字符串中独立搜索`u'\uD808'`和`u'\uDF45'`成功，这并不一定意味着字符`u'\U00012345`是该字符串的一部分。很可能字符串中有两个不同的字符，它们恰好与所讨论的字符的代理对共享一个代码单元。查找子字符串会正常工作是因为代理对中的两个代码单元使用不同的值范围，所以始终可以在一系列代码点中识别出该对。

从0.15版开始，Cython已经扩展了对代理对的支持，这样就可以安全地使用`in`测试来搜索整个**Py_UCS4**范围内的字符值，即使是在窄平台上：
```cython
cdef Py_UCS4 uchar = 0x12345
print( uchar in some_unicode_string )
```
同样，在窄Unicode和宽Unicode平台上，它都可以将一个高Unicode编码点值的字符串强制转换为Py_UCS4值：
```cython
cdef Py_UCS4 uchar = u'\U00012345'
assert uchar == 0x12345
```
在CPython 3.3及更高版本中，**Py_UNICODE**类型是特定于系统的**wchar_t**类型的别名，并且不再与Unicode字符串的内部表示形式绑定。相反，任何Unicode字符都可以在所有平台上表示，而不必求助于代理对。这意味着从该版本开始，无论**Py_UNICODE**的大小如何，都不再存在窄构建。详见<b>[PEP 393](https://www.python.org/dev/peps/pep-0393/)</b>。

Cython 0.16稍后将在内部处理这一更改，并且只要将类型推断应用于非类型化变量，或者在源代码中显式使用可移植的**Py_UCS4**类型，而不是平台特定的**Py_UNICODE**类型，那么对于单个字符值也会执行正确的操作。Cython应用于Python unicode类型的优化将像往常一样在C编译时自动适应<b>[PEP 393](https://www.python.org/dev/peps/pep-0393/)</b>。
## 迭代
Cython 0.13支持对<b>char*</b>、bytes和unicode字符串进行有效的迭代，只要循环变量类型正确。因此，下面将生成预期的C代码：
```cython
cdef char* c_string = "Hello to A C-string's world"

cdef char c
for c in c_string[:11]:
    if c == 'A':
        print("Found the letter A")
```
这同样适用于bytes对象：
```cython
cdef bytes bytes_string = b"hello to A bytes' world"

cdef char c
for c in bytes_string:
    if c == 'A':
        print("Found the letter A")
```
对于unicode对象，Cython将自动推断循环变量的类型为**Py_UCS4**：
```
cdef unicode ustring = u'Hello world'

# NOTE: 'uchar'不需要类型化!
for uchar in ustring:
    if uchar == u'A':
        print("Found the letter A")
```
这里的自动类型推断通常会导致更有效率的代码。但是，请注意，有些unicode操作仍然要求值是Python对象，因此Cython可能最终会为循环内的循环变量值生成冗余的转换代码。如果这导致特定代码段的性能下降，你可以显式地将循环变量类型化为Python对象，或者将其值赋给循环内某个Python类型化的变量，以便在对其运行Python操作之前强制执行一次性强制。

在测试中也有一些优化，所以下面的代码将在纯C代码中运行（实际上使用了switch语句）：
```cython
cpdef void is_in(Py_UCS4 uchar_val):
    if uchar_val in u'abcABCxY':
        print("The character is in the string.")
    else:
        print("The character is not in the string")
```
结合上面的循环优化，这可以导致高效的字符切换代码，例如unicode解析器。
## Windows和宽字符API
Windows系统API本身支持Unicode的形式是零终止的UTF-16编码的<b>wchar_t*</b>字符串，即所谓的“宽字符串”。

默认情况下，CPython的Windows构建将**Py_UNICODE**定义为**wchar_t**的同义词。这使得内部**unicode**表示与UTF-16兼容，并允许高效的零拷贝转换。这也意味着Windows构建总是带有所有警告的[窄Unicode构建](#窄Unicode构建)。

为了支持与Windows API之间的互操作，Cython 0.19支持宽字符串（以<b>Py_UNICODE*</b>的形式），并隐式地将它们转换为**unicode**字符串对象或从**unicode**字符串对象转换过来。这些转换的行为与[传入字节字符串](#传入字节字符串)中描述的<b>char*</b>和**bytes**的行为相同。

除了自动转换之外，出现在C上下文中的unicode文字还变成了C级别的宽字符串文字，**len()**内置函数专门用来计算零终止的<b>Py_UNICODE*</b>字符串或数组的长度。

下面是一个如何在Windows上调用Unicode API的例子：
```cython
cdef extern from "Windows.h":

    ctypedef Py_UNICODE WCHAR
    ctypedef const WCHAR* LPCWSTR
    ctypedef void* HWND

    int MessageBoxW(HWND hWnd, LPCWSTR lpText, LPCWSTR lpCaption, int uType)

title = u"Windows Interop Demo - Python %d.%d.%d" % sys.version_info[:3]
MessageBoxW(NULL, u"Hello Cython \u263a", title, 0)
```
<table><tr><td bgcolor=red>
<b>警告</b>：强烈建议不要在Windows之外使用<b>Py_UNICODE*</b>字符串。<b>Py_UNICODE</b>本质上不能在不同平台和Python版本之间移植。

CPython 3.3已经转向unicode字符串的灵活内部表示（<b>[PEP 393](https://www.python.org/dev/peps/pep-0393/)</b>），这使得所有与<b>Py_UNICODE</b>相关的API都被弃用，而且效率低下。
</td></tr></table>

CPython 3.3变化的一个结果是**unicode**字符串的<b>len()</b>总是以代码点（“字符”）度量，而Windows API期望UTF-16代码单元的数量（其中每个代理都单独计量）。要始终获得代码单元的数量，请直接调用**PyUnicode_GetSize()**。