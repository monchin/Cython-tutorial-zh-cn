# Cython基础教程
## Cython的基础
Cython的本质可以总结为：Cython是具有C数据类型的Python。

Cython就是Python：几乎所有的Python代码都是有效的Cython代码。（有少量例外，但这里我们先承认这个结论。）Cython的编译器会把代码转换成等效于调用Python/C API的C代码。

但是Cython远不止于此，因为它的参数和变量可以被声明为具有C数据类型。我们可以自由地混用操作Python值和C值的代码，而Cython会帮助我们自动转换需要转换的地方。Python中的引用计数保存和错误检查也是自动的，而且Python的异常处理机制，包括try-except和try-finally同样可行——即使是在操作C数据时。
## Cython Hello World
由于Cython可以接受几乎所有有效的python源文件，在我们Cython的启程之路上最大的拦路虎之一就是如何去编译你的拓展文件。

让我们从标准的python hello world开始：
```Python
print("Hello World")
```
将这段代码保存为`helloworld.pyx`。现在我们需要创建一个`setup.py`，这就像一个python Makefile。你的`setup.py`应该像这样：
```Python
from distutils.core import setup
from Cython.Build import cythonize

setup(
    ext_modules = cythonize("helloworld.pyx")
)
```
使用下面的命令行选项来建立你的Cython文件：
```
$ python setup.py build_ext --inplace`
```
在unix系统中，这行命令会在你的本地文件夹里创建一个叫做`helloworld.so`的文件；而在Windows中它叫`helloworld.pyd`。现在，运行你的python解释器，然后把这个文件看成一个普通的python模块，简单地import它就可以使用了。
```Python
>>> import helloworld
Hello World
```
恭喜！你现在已经知道如何去创建一个Cython拓展了。但是，这个例子会给人一种不知道Cython有何优势的感觉，所以我们会来一个更有现实意义的例子。
## pyximport：为开发者准备的Cython编译
如果你的模块不需要任何外部的C库或者特殊的安装方式，你可以直接使用pyximport模块。这个模块由Paul Prescod开发，用来直接使用import来载入`.pyx`文件，而不需要在你每次更改代码的时候都重新跑一遍你的setup.py文件。它与Cython一起发布和安装，使用方法如下：
```Python
>>> import pyximport; pyximport.install()
>>> import helloworld
Hello World
```
Pyximport模块也支持对普通的Python模块实验性的编译。这可以让你在Python导入的每一个`.pyx`和`.py`模块上自动运行Cython，包括标准库和被安装的包。Cython在编译大量Python模块的时候也会失败，此时import机制将会回溯，转而去载入Python源模块。`.py`的import机制可以被这样安装：
```python
>>> pyximport.install(pyimport=True)
```
注意，现在已经不推荐在终端用户侧使用Pyximport的创建代码了，因为它会hook上import系统。对终端用户来说，最好的方法时提供wheel包形式的二进制预创建包。
## 斐波那契函数
Python的官方教程里有一个简单的斐波那契函数：
```python
from __future__ import print_function

def fib(n):
    """打印斐波那契数列到n"""
    a, b = 0, 1
    while b < n:
        print(b, end=' ')
        a, b = b, a + b

    print()
```
现在让我们跟着Hello World的例子依样画葫芦。首先，我们将文件的拓展名改为`.pyx`，比如`fib.pyx`；接下来，我们创建`setup.py`文件。我们使用Hello World例子中创建的文件，你需要做的只是将Cython的文件名和生成的模块名改掉。这样，我们得到：
```python
from distutils.core import setup
from Cython.Build import cythonize

setup(
    ext_modules=cythonize("fib.pyx"),
)
```
建立拓展的命令与`helloworld.pyx`的例子相同：
```
$ python setup.py build_ext --inplace
```
使用这个拓展：
```python
>>> import fib
>>> fib.fib(2000)
1 1 2 3 5 8 13 21 34 55 89 144 233 377 610 987 1597
```
## 质数
这里有一个小例子来展示Cython可以做到的一部分事情。
`primes.pyx:`
```cython
1    def primes(int nb_primes):
2        cdef int n, i, len_p
3        cdef int p[1000]
4        if nb_primes > 1000:
5        nb_primes = 1000
6   
7        len_p = 0  # p中当前元素的数量
8        n = 2
9        while len_p < nb_primes:
10           # 是否是质数？
11           for i in p[:len_p]:
12               if n % i == 0:
13                   break
14
15           # 如果循环中未发生break，我们就找到了一个质数
16           else:
17               p[len_p] = n
18               len_p += 1
19           n += 1
20
21       # 以python列表的形式返回结果
22       result_as_list  = [prime for prime in p[:len_p]]
23       return result_as_list
```
可以看到，函数的开始部分就像普通的Python函数定义，除了我们声明了参数`nb_primes`的int类型。这意味着这个对象被传入时会被转换为C的整数类型（或是一个`TypeError`，如果转换失败的话）。

现在让我们来仔细研究一下这个函数：
```cython
cdef int n, i, len_p
cdef int p[1000]
```
第2行和第3行使用了cdef语句来定义一些局部的C变量。在处理过程中，结果被保存在一个C数组`p`中，并且在最后会被复制到一个Python列表中（第22行）。
* 注意，在这个例子中，你不能创建太大的数组，因为它们是被分配在C函数调用的栈上，而这些栈资源是有限的。如果需要更大的数组，或者如果数组长度只有在程序运行时才能被确定，你可以去学习使用Cython对C内存分配、Python数组或Numpy数组进行更有效率的利用。
```cython
if nb_primes > 1000:
    nb_primes = 1000
```
在C中，声明一个静态数组需要在编译时知道数组的大小。我们需要确保使用者设置的值不大于1000（或者就像C一样生成一个段错误）。
```cython
len_p = 0  # p中当前元素的数量
n = 2
while len_p < nb_primes:
```
第7~9行建立了一个循环，用来判断数字是否是质数，直到达到所需的质数数量。
```cython
# 是否是质数？
for i in p[:len_p]:
    if n % i == 0:
        break
```
第11~12行使用候选数字依次除以已经找到的每一个质数。这两行代码值得我们深入研究一下。因为这里面没有Python对象被引用，循环被整体翻译成了C代码，因此跑得飞快。请注意我们迭代C数组`p`的方式：
```cython
for i in p[:len_p]:
```
循环被翻译为一段很快的C循环，但写程序时就像是在操作一个Python列表或是Numpy数组一样。如果你不使用`[:len_p]`来对C数组进行切片操作，Cython就会在数组中循环整整1000个元素。
```cython
# 如果循环中未发生break
else:
    p[len_p] = n
    len_p += 1
n += 1
```
如果没有发生break，也就是说我们找到了一个质数，那么从第16行<font color=green>else</font>之内的代码段就会被执行。我们向p中添加这个被找到的质数。在<font color=green>for</font>循环之后有一个<font color=green>else</font>是Python一个少为人知的语言特性，所以你可能会觉得有些奇怪，而Cython会以C的速度来执行它。如果你想更多了解这个少为人知的特性，请看[该博客](https://shahriar.svbtle.com/pythons-else-clause-in-loops)。
```cython
# 以python列表的形式返回结果
result_as_list  = [prime for prime in p[:len_p]]
return result_as_list
```
在第22行，在我们返回结果之前，我们需要将C数组复制到一个Python列表中，因为Python不能读取C数组。Cython可以自动将很多C类型转换为Python类型，所以我们可以用一个简单的列表来把C int类型的值以Python int对象的形式复制到Python列表。你也可以采用手动循环的方式使用`result_as_list.append(prime)`来完成这个操作，结果是一样的。

你会注意到我们采用了与Python完全相同的方式来声明一个Python列表。因为我们没有显式声明变量`result_as_list`的类型，它会被视为一个Python对象，而Cython也会从分配中知道它的确切类型为一个Python列表。

最后，我们返回了一个结果的列表。

使用Cython编译器，我们编译`primes.pyx`从而生成一个拓展模块。我们可以在交互的解释器里试试这个拓展模块：
```python
>>> import primes
>>> primes.primes(10)
[2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
```
成功了！你可能好奇Cython究竟帮我们节省了多少工作量，所以让我们来看看这个模块生成的C代码。

Cython为我们提供了一种方式，让我们可以看到Python对象和Python的C-API之间的交互。我们在`cythonize()`中传入参数`annotate=True`，它会生成一个HTML文件。

![image.png](http://docs.cython.org/en/latest/_images/htmlreport1.png)

白色的行意味着生成代码时没有和Python进行交互，所以这部分代码会运行得和正常的C代码一样快。黄色越深说明在这一行与Python进行的交互越多。这些黄色的行会操作Python对象、生成异常、或是做一些其他更高级的操作，而这些操作都不能被翻译为简单快速的C代码。函数声明和返回使用了Python的解释器，所以这些行也是黄色的。列表也是同样，因为这是创建了一个Python对象。但是为什么`if n % i == 0:`这一行也是黄色的呢？我们可以通过检查生成的C代码来帮助我们理解：

![image.png](http://docs.cython.org/en/latest/_images/python_division.png)

我们可以看到这里做了一些检查。因为Cython默认使用Python的行为，所以它会像Python一样在运行时做除法检查。你可以在使用[编译指令](http://docs.cython.org/en/latest/src/userguide/source_files_and_compilation.html#compiler-directives)时禁用这项检查。

现在，即使我们做除法检查，我们也可以看看程序运行的速度。我们写一个Python版的相同的程序：
```python
def primes_python(nb_primes):
    p = []
    n = 2
    while len(p) < nb_primes:
        # 是否是质数？
        for i in p:
            if n % i == 0:
                break

        # 如果循环中未发生break
        else:
            p.append(n)
        n += 1
    return p
```
我们也可以直接使用`.py`文件，但使用Cython来编译它。让我们使用`primes_python`，把它的函数名改为`primes_python_compiled`，在不改变任何代码的情况下使用Cython编译它。我们也可以将文件名修改为`example_py_cy.py`来区分这个文件。现在`setup.py`是这样：
```python
from distutils.core import setup
from Cython.Build import cythonize

setup(
    ext_modules=cythonize(['example.pyx',        # 含有primes()函数的Cython代码文件
                           'example_py_cy.py'],  # 含有primes_python_compiled()函数的Python代码文件 
                          annotate=True),        # 生成html注释文件
)
```
我们可以保证两个程序输出结果相同：
```python
>>> primes_python(1000) == primes(1000)
True
>>> primes_python_compiled(1000) == primes(1000)
True
```
比较一下速度：
```
python -m timeit -s 'from example_py import primes_python' 'primes_python(1000)'
10 loops, best of 3: 23 msec per loop

python -m timeit -s 'from example_py_cy import primes_python_compiled' 'primes_python_compiled(1000)'
100 loops, best of 3: 11.9 msec per loop

python -m timeit -s 'from example import primes' 'primes(1000)'
1000 loops, best of 3: 1.65 msec per loop
```
`primes_python`的cythonize版本比Python的版本速度快一倍，而我们并没有改代码。而Cython版本更是比Python版本的速度快了13倍！怎么解释？

多种因素：
* 这个程序中，每一行只有少量的计算，所以python解释器的总开销就非常重要了。如果需要做大量的计算情况就不一样了。比如Numpy。
* 数据的局部性。C比Python对CPU缓存的支持性更好。Python中一切皆对象，每个对象都以字典的形式实现，而这对缓存是不友好的。

一般来说，加速会在2x到1000x之间，具体取决于你调用Python解释器的多少。请在每次添加类型时先分析。添加类型会使代码的可读性变差，所以你需要做一个权衡。
## C++版本的质数
你也可以用Cython来调用C++。注意，一部分C++标准库可以在Cython代码中被直接导入。

让我们看看`primes.pyx`在使用C++标准库中的vector后会如何变化：

* 注意，C++中的vector是一种建立于可改变容量的C数组上的列表或栈的数据结构。它和数组标准库模块中Python的数组类型相似。如果事先知道你需要在vector中放入多少数据，你就可以使用方法`reserve`来避免复制操作。更多信息请看[此页面](https://en.cppreference.com/w/cpp/container/vector)。

```cython
# distutils: language=c++

from libcpp.vector cimport vector

def primes(unsigned int nb_primes):
    cdef int n, i
    cdef vector[int] p
    p.reserve(nb_primes)  # allocate memory for 'nb_primes' elements.

    n = 2
    while p.size() < nb_primes:  # vector的size()与len()类似
        for i in p:
            if n % i == 0:
                break
        else:
            p.push_back(n)  # push_back is similar to append()
        n += 1

    # 在转换到Python对象时vector被自动转换为Python列表
    return p
```
第一行是编译指令，它告诉Cython把代码编译到C++。这样你就可以使用C++语言的特性和C++标准库。注意不能使用`pyximport`来把Cython代码编译到C++。你需要使用`setup.py`者notebook来运行这个例子。

可以看到，vector的API和Python列表的API相似。在Cython中，有时我们也可以将它作为一个简易的替换。

关于更多在Cython中使用C++的细节，请看[在Cython中使用C++](http://docs.cython.org/en/latest/src/userguide/wrapping_CPlusPlus.html#wrapping-cplusplus)。

## 语言细节
更多关于Cython语言请看[语言基础](http://docs.cython.org/en/latest/src/userguide/language_basics.html#language-basics)。若要在数值计算环境中直接使用Cython，请看[Typed Memoryviews](http://docs.cython.org/en/latest/src/userguide/memoryviews.html#memoryviews).