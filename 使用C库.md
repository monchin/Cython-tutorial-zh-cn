# 使用C库
除了写快速代码外，Cython的一个主要用例是从Python代码调用外部C库。当Cython代码编译为C代码本身时，在代码中直接调用C函数实际上非常简单。下面给出了在Cython代码中使用(和包装)外部C库的完整示例，包括适当的错误处理和为Python和Cython代码设计合适的API时的注意事项。

假设你需要一种有效率的方法来在先进先出队列中存储整数值。由于内存真的很重要，而且值实际上来自C代码，所以你不能在列表或双向队列中创建和存储Python int对象。所以你要寻找C语言中的队列实现。

在网上找了一圈之后，你找到了C-algorithms library [CAlg]，并决定使用它的双端队列实现。但是，为了简化处理，你决定将其封装在一个Python扩展类型中，该扩展类型可以封装所有内存管理。

[CAlg]	Simon Howard, C Algorithms library, http://c-algorithms.sourceforge.net/
## 定义外部声明
你可以在[这里](https://codeload.github.com/fragglet/c-algorithms/zip/master)下载CAlg。

队列实现的C API被定义在头文件`c-algorithms/src/queue.h`中，看上去像这样：
```C
/* queue.h */

typedef struct _Queue Queue;
typedef void *QueueValue;

Queue *queue_new(void);
void queue_free(Queue *queue);

int queue_push_head(Queue *queue, QueueValue data);
QueueValue queue_pop_head(Queue *queue);
QueueValue queue_peek_head(Queue *queue);

int queue_push_tail(Queue *queue, QueueValue data);
QueueValue queue_pop_tail(Queue *queue);
QueueValue queue_peek_tail(Queue *queue);

int queue_is_empty(Queue *queue);
```
首先，第一步是在`.pxd`文件中重新定义C API，比如`cqueue.pxd`：
```python
# cqueue.pxd

cdef extern from "c-algorithms/src/queue.h":
    ctypedef struct Queue:
        pass
    ctypedef void* QueueValue

    Queue* queue_new()
    void queue_free(Queue* queue)

    int queue_push_head(Queue* queue, QueueValue data)
    QueueValue  queue_pop_head(Queue* queue)
    QueueValue queue_peek_head(Queue* queue)

    int queue_push_tail(Queue* queue, QueueValue data)
    QueueValue queue_pop_tail(Queue* queue)
    QueueValue queue_peek_tail(Queue* queue)

    bint queue_is_empty(Queue* queue)
```
注意，这些声明与头文件声明几乎完全相同，因此你常常可以将它们复制过来。但是，你不需要像上面那样提供所有声明，只需要那些在代码或其他声明中使用的声明，这样Cython就可以看到它们的一个充分且一致的子集。然后，考虑对它们进行一些调整，使它们更适合在Cython中工作。

具体来说，你应该注意为C函数选择合适的参数名，因为Cython允许你将它们作为关键字参数进行传递。稍后更改它们是一种向后不兼容的API修改。立即选择好名称将使Cython代码中的这些函数更易于使用。

我们上面使用的头文件的一个值得注意的区别是第一行中队列结构的声明。在本例中，队列用作不透明句柄;只有被调用的库才知道其中真正的内容。由于Cython代码不需要知道结构体的内容，我们不需要声明它的内容，因而我们只提供一个空定义(因为我们不想声明在C头文件中引用的`_Queue`类型)[1]。
    
    [1] `cdef struct Queue: pass`和`ctypedef struct Queue: pass`之间有一个细微的区别。前者声明在C代码中引用的类型为`struct Queue`，而后者在C代码中引用为`Queue`。这是`Cython`无法隐藏的C语言怪癖。大多数现代C库使用`ctypedef`类型的结构。
另一个例外是最后一行。`queue_is_empty()`函数的整数返回值实际上是一个C布尔值，也就是说，我们只关心它是非零还是零，表示队列是否为空。这最好用Cython的`bint`类型来表示，在C中使用`bint`类型时是一种普通的`int`类型，但是在转换为Python对象时映射到Python的`boolean`值`True`和`False`。这种压缩`.pxd`文件中的声明的方法通常可以简化使用它们的代码。

为你使用的每个库定义一个.pxd文件是一个很好的习惯。如果API很大，有时甚至为每个头文件(或功能组)定义一个.pxd文件。这简化了它们在其他项目中的重用。有时，你可能需要使用标准C库中的C函数，或者希望直接从CPython调用C-API函数。对于这样的的常见需求，Cython附带了一组标准`.pxd`文件，这些文件以一种易于使用的方式提供这些声明，并对其在Cython中的使用进行了调整。主要的包是`cpython`、`libc`和`libcpp`。NumPy库还有一个标准的`.pxd`文件`numpy`，因为它在Cython代码中很常用。请查阅Cython的`Cython/Includes/`源包来看被提供的`.pxd`文件的完整名单。

