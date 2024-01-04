MPI的一组函数用于实现进程的集体通信，==这些函数可以实现不仅仅是两个单独进程之间的消息交换，而是所有在同一个通信组（communicator）中的进程之间的消息交换==。

通常，**MPI的集体函数是更好的选择，而不是多次调用点对点通信操作**，原因如下：

1. 高效性：MPI库实现了高效的算法，以实现集体通信操作。
2. 硬件支持：在超级计算机或集群系统中，集体通信操作可能会在硬件层面进行优化，这在MPI库中得到了考虑。

==集体通信操作是以阻塞模式执行的。要成功执行集体操作，需要确保在通信组（communicator）的所有进程中都调用了相应的集体函数==。

如果**集体操作涉及到一个特殊的进程扮演特殊的角色**，*那么在相应的MPI函数中会有一个称为`root`的参数*，
它指定了特殊角色的进程的排名。通常情况下，**只有在根进程的进程中使用的某些参数才会生效**。

如果MPI函数没有`root`参数，那么这意味着所有进程在集体操作中都是对等的。

1. MPI_Barrier(MPI_Comm comm) 是最简单的集体函数之一，它用于同步所有进程。它会阻塞调用它的进程，直到通信组（`comm`）中的所有进程都调用了MPI_Barrier。这可以用于确保所有进程在某个点同步，以便进行后续的操作。

3. 另一个具有根进程的基本集体函数是 MPI_Bcast(void* buf, int count, MPI_Datatype datatype, int root, MPI_Comm comm)。这个函数==从根进程将数据广播到通信组中的所有其他进程==。参数`buf`指定了广播消息的缓冲区，`count`表示要广播的元素数量，`datatype`表示元素的数据类型。根进程将数据发送到所有其他进程。这可以用于在集合中分发共享的数据或信息。在根进程中，`buf`参数用作输入，而在其他进程中，`buf`参数用作输出。
 ![[Pasted image 20231014161615.png]]

# MPI_Gather
MPI_Gather是一种MPI库中的集体通信函数，用于从通信器中的所有进程收集数据到根进程的缓冲区中。MPI_Gather的参数如下：

- `void* sbuf`：发送缓冲区，包含每个进程要发送的数据。
- `int scount`：每个进程发送的元素数量。
- `MPI_Datatype stype`：发送数据元素的数据类型。
- `void* rbuf`：接收缓冲区，数据将被收集到根进程中的这个缓冲区。这是输出参数，只有根进程使用它。
- `int rcount`：根进程接收的来自每个进程的元素数量。这个参数只在根进程中使用，通常不等于接收缓冲区`rbuf`的大小。
- `MPI_Datatype rtype`：接收数据元素的数据类型。
- `int root`：根进程的排名，它将收集来自其他进程的数据。
- `MPI_Comm comm`：通信器，指定了哪些进程参与数据收集操作。

MPI_Gather函数用于将每个进程的数据收集到根进程的缓冲区`rbuf`中，根进程的排名由`root`指定。

==所有进程必须参与MPI_Gather操作，否则会发生死锁。==

值得注意的是，根进程将收集所有进程的数据，包括自己的数据。

这个函数在需要将数据从多个进程收集到一个进程以进行进一步处理或分析时非常有用

## MPI_Gatherv
MPI_Gatherv是MPI库中的集体通信函数，它允许从通信器中的所有进程收集数据到根进程的缓冲区中，并且不同进程可以发送不同数量的元素。MPI_Gatherv函数的参数如下：

- `void* sbuf`：发送缓冲区，包含每个进程要发送的数据。
- `int scount`：每个进程发送的元素数量（可以不同）。
- `MPI_Datatype stype`：发送数据元素的数据类型。
- `void* rbuf`：接收缓冲区，数据将被收集到根进程中的这个缓冲区。这是输出参数，只有根进程使用它。
- `int* rcounts`：一个整数数组，指定根进程从每个进程接收的元素数量。
- `int* displs`：一个整数数组，指定每个进程的数据在接收缓冲区中的偏移量。
- `MPI_Datatype rtype`：接收数据元素的数据类型。
- `int root`：根进程的排名，它将收集来自其他进程的数据。
- `MPI_Comm comm`：通信器，指定了哪些进程参与数据收集操作。

MPI_Gatherv函数允许不同进程发送不同数量的元素，其中`rcounts`参数指定了根进程从每个进程接收的元素数量，而`displs`参数则指定了每个进程的数据在接收缓冲区中的偏移量。这使得数据的收集更加灵活，可以处理不均匀的数据分布。

这个函数对于需要从多个进程收集数据，并且每个进程可能发送不同数量的元素的情况非常有用。通过指定`rcounts`和`displs`参数，您可以有效地管理数据的接收和存储。

## MPI_Scatter
MPI_Scatter是MPI库中的集体通信函数，用于从根进程向通信器中的所有其他进程散发数据。它的参数如下：

- `void* sbuf`：发送缓冲区，根进程中的数据将从这里发送给其他进程。这是输入参数，只有根进程使用它。
- `int scount`：每个进程接收的元素数量（可以不同）。
- `MPI_Datatype stype`：发送数据元素的数据类型。
- `void* rbuf`：接收缓冲区，接收到的数据将存储在这里。这是输出参数，每个非根进程都使用它。
- `int rcount`：接收缓冲区中每个进程应该接收的元素数量。
- `MPI_Datatype rtype`：接收数据元素的数据类型。
- `int root`：根进程的排名，它将向其他进程发送数据。
- `MPI_Comm comm`：通信器，指定了哪些进程参与数据散发操作。

MPI_Scatter函数用于将根进程中的数据分发给通信器中的其他进程。`scount`参数指定了每个接收进程应该接收的元素数量，而`rcount`参数指定了每个接收缓冲区中的元素数量。这允许不同的接收进程接收不同数量的元素。

MPI_Scatterv是其更灵活的变体，允许不同进程接收不同数量的元素，类似于MPI_Gatherv。这两个函数对于需要将数据从一个根进程分发到通信器中的其他进程的情况非常有用。

这些函数可以有效地在并行计算中进行数据分发，以实现各种分布式计算任务。

## MPI_Scatterv
MPI_Scatterv是MPI库中的集体通信函数，用于从根进程向通信器中的所有其他进程散发数据，允许每个接收进程接收不同数量的元素。它的参数如下：

- `void* sbuf`：发送缓冲区，根进程中的数据将从这里发送给其他进程。这是输入参数，只有根进程使用它。
- `int* scounts`：一个整数数组，指定每个进程接收的元素数量。数组的大小是通信器中的进程数量。
- `int* displs`：一个整数数组，表示从发送缓冲区的哪个位置开始发送数据给每个接收进程。这允许从发送缓冲区中选择不同的起始位置。数组的大小也是通信器中的进程数量。
- `MPI_Datatype stype`：发送数据元素的数据类型。
- `void* rbuf`：接收缓冲区，接收到的数据将存储在这里。这是输出参数，每个非根进程都使用它。
- `int rcount`：接收缓冲区中每个进程应该接收的元素数量。
- `MPI_Datatype rtype`：接收数据元素的数据类型。
- `int root`：根进程的排名，它将向其他进程发送数据。
- `MPI_Comm comm`：通信器，指定了哪些进程参与数据散发操作。

MPI_Scatterv函数非常灵活，允许您以不同的方式分发数据，每个接收进程可以接收不同数量的元素，并且数据可以来自不同的位置，具体取决于scounts和displs参数的设置。这使其适用于各种分布式计算任务，特别是当不同进程需要接收不同数量的数据时。

## MPI_Allgather
MPI_Allgather是MPI库中的集体通信函数，用于从每个进程收集数据，然后将结果广播给所有其他进程。所有进程都提供自己的数据，并且每个进程都会收到一个包含所有进程数据的结果。它的参数如下：

- `void* sbuf`：发送缓冲区，包含每个进程提供的数据。这是输入参数，每个进程都提供它自己的数据。
- `int scount`：发送缓冲区中每个进程提供的元素数量。
- `MPI_Datatype stype`：发送数据元素的数据类型。
- `void* rbuf`：接收缓冲区，包含所有进程提供的数据。这是输出参数，每个进程都会接收到包含所有数据的结果。
- `int rcount`：接收缓冲区中每个进程接收的元素数量。
- `MPI_Datatype rtype`：接收数据元素的数据类型。
- `MPI_Comm comm`：通信器，指定了哪些进程参与数据收集和广播操作。

MPI_Allgather函数用于收集来自每个进程的数据，然后将这些数据分发给每个进程，以使每个进程都有来自所有其他进程的数据。这对于需要每个进程具有完整信息的情况非常有用，例如并行排序算法。

请注意，与MPI_Gather不同，MPI_Allgather不需要指定根进程，因为每个进程都提供数据并接收结果。这使其非常适合在一组进程之间共享数据。

MPI_Allgatherv(void* sbuf, int scount, MPI_Datatype stype, void* rbuf, int* rcounts, int* displs, MPI_Datatype rtype, MPI_Comm comm)

## MPI_Alltoall
`MPI_Alltoall` 是一个高级的集体通信操作，用于在 MPI 程序中执行 All-to-All 通信模式。在 `MPI_Alltoall` 中，每个进程向通信组中的所有其他进程发送不同的数据，并从每个进程接收相同数量的数据。

以下是 `MPI_Alltoall` 的参数：

- `void* sbuf`：发送缓冲区，包含每个进程要发送的数据。这是输入参数，每个进程都有自己的 `sbuf`。
- `int scount`：发送缓冲区中每个进程发送的数据元素数量。
- `MPI_Datatype stype`：发送数据元素的数据类型。
- `void* rbuf`：接收缓冲区，包含从每个进程接收的数据。这是输出参数，每个进程都会从其他进程接收相同数量的数据，这些数据存储在 `rbuf` 中。
- `int rcount`：接收缓冲区中每个进程接收的数据元素数量。
- `MPI_Datatype rtype`：接收数据元素的数据类型。
- `MPI_Comm comm`：通信器，指定了哪些进程参与 All-to-All 操作。

`MPI_Alltoall` 对于需要在通信组中的每个进程之间交换相同数量的数据时非常有用。这种通信模式可以用于许多并行算法和应用程序中，例如数据分发和数据收集。

在 `MPI_Alltoall` 中，每个进程发送数据给所有其他进程，并从所有其他进程接收数据。这种通信模式通常需要高效的通信拓扑和算法，以确保性能良好。

## MPI_Alltoallv

`MPI_Alltoallv` 是一个 MPI 集体通信操作，用于在 MPI 程序中执行 All-to-All 通信模式。在 `MPI_Alltoallv` 中，每个进程向通信组中的所有其他进程发送不同数量的数据，并从每个进程接收不同数量的数据。此函数提供了更大的灵活性，因为您可以指定每个进程发送和接收的数据量，以及相对于缓冲区的偏移量。

以下是 `MPI_Alltoallv` 的参数：

- `void* sbuf`：发送缓冲区，包含每个进程要发送的数据。这是输入参数，每个进程都有自己的 `sbuf`。
- `int* scounts`：一个整数数组，指定了每个进程要发送的数据元素数量。数组的长度等于通信组中的进程数。
- `int* sdispls`：一个整数数组，指定了每个进程发送数据时在 `sbuf` 中的偏移量。它指定了数据从哪里开始。
- `MPI_Datatype stype`：发送数据元素的数据类型。
- `void* rbuf`：接收缓冲区，包含从每个进程接收的数据。这是输出参数，每个进程都会从其他进程接收数据，这些数据存储在 `rbuf` 中。
- `int* rcounts`：一个整数数组，指定了每个进程要接收的数据元素数量。数组的长度等于通信组中的进程数。
- `int* rdispls`：一个整数数组，指定了每个进程接收数据时在 `rbuf` 中的偏移量。它指定了数据应该放在 `rbuf` 的哪个位置。
- `MPI_Datatype rtype`：接收数据元素的数据类型。
- `MPI_Comm comm`：通信器，指定了哪些进程参与 All-to-All 操作。

`MPI_Alltoallv` 对于需要在通信组中的每个进程之间交换不同数量的数据时非常有用。这种通信模式可以用于许多并行算法和应用程序中，其中每个进程需要与其他进程进行数据交换，而且每个进程的通信需求不同。

# 集体通讯总结
以下是其中一些集体操作函数的简要描述：

1. **MPI_Gather**：它用于将所有进程的数据收集到一个特定的进程中，通常称为“根”进程。每个进程可以贡献一个数据块，这些块在根进程中按照一定顺序组合。
    
2. **MPI_Scatter**：与MPI_Gather相反，它从一个特定的根进程中将数据散发给所有其他进程。每个进程将收到来自根进程的数据。
    
3. **MPI_Allgather**：这个操作允许每个进程将自己的数据发送给其他所有进程，并从每个进程接收其他进程的数据。这实现了全互连的数据交换。
    
4. **MPI_Alltoall**：每个进程将数据发送给所有其他进程，并从每个其他进程接收数据。这也是全互连的数据通信操作。
    
5. **MPI_Gatherv**和**MPI_Scatterv**：它们类似于MPI_Gather和MPI_Scatter，但允许不同进程发送和接收不同数量的数据块。
    
6. **MPI_Alltoallv**：类似于MPI_Alltoall，但允许不同进程发送和接收不同数量的数据块。
    
7. **MPI_Alltoallw**：这是MPI_Alltoallv的扩展，允许以字节为单位指定偏移量，并处理不同数据类型的数据。
    

这些集体操作函数提供了处理并行计算中不同类型通信需求的工具，包括数据聚集、数据分发、数据同步等操作。在MPI程序中，它们用于实现不同进程之间的数据交换和协同工作，是高性能并行计算的重要组成部分。


# 归约
这段文本讨论了MPI中用于执行归约操作的一组集体函数。

归约操作是一种将多个进程的处理结果进行合并的操作，通常涉及诸如求和（MPI_SUM）、乘积（MPI_PROD）、最大值（MPI_MAX）或最小值（MPI_MIN）等操作。此外，还包括逻辑运算，如与（MPI_LAND）、或（MPI_LOR）、异或（MPI_LXOR）以及位运算，如按位与（MPI_BAND）、按位或（MPI_BOR）和按位异或（MPI_BXOR）。其中，MPI_MAXLOC和MPI_MINLOC操作用于查找最大或最小元素，同时还提供了该元素的进程编号。

用户可以自定义归约操作（op），对此用到了==**MPI_Op_create**==函数。自定义操作必须是可结合的（associative）。
如果操作也是可交换的（commutative），则需要将commute参数设置为非零。
用户定义的操作通过一个函数MPI_User_function实现，该函数采用输入数组invec和输出数组inoutvec，以及元素的数量和数据类型等参数。

用户可以使用`MPI_Op_create`函数来创建新的归约操作（reduction operation）。此操作必须满足结合律（associative），如果还满足交换律（commutative），则需要将`commute`参数设置为非零值。`MPI_Op_create`函数的原型如下：

```C
MPI_Op_create(MPI_User_function* function, int commute, MPI_Op* op)
```

参数`function`是一个指向用户定义的函数的指针，该函数用于定义新的操作。这个用户定义的操作需要对输入缓冲区`invec`、输出缓冲区`inoutvec`、元素数量`len`和数据类型`datatype`进行操作。操作应用于`invec[i]`和`inoutvec[i]`的元素，结果应该被保存在`inoutvec[i]`中。
```C
typedef void MPI_User_function(void* invec, void* inoutvec, int* len, MPI_Datatype* datatype);
```
如果用户定义的操作只用于特定类型的数据，那么在定义时可以假设`invec`和`inoutvec`具有所需的数据类型，并且无需分析`datatype`参数。


在MPI-1标准中，有四个执行归约操作的函数：
MPI标准提供了四个执行归约操作的函数：`MPI_Reduce`、`MPI_Allreduce`、`MPI_Reduce_scatter`和`MPI_Scan`。

1. `MPI_Reduce`函数执行全局归约操作，并将结果发送到指定的接收进程（root）。
2. `MPI_Allreduce`函数执行全局归约操作，并将结果广播到所有进程。
3. `MPI_Reduce_scatter`函数执行全局归约操作，并将结果分散到各个进程。它不需要指定接收进程，而是根据`rcounts`数组来决定每个进程接收的元素数量。
4.  `MPI_Scan`函数执行一系列部分归约操作，并将结果依次发送给各个进程

- MPI_Reduce函数执行全局归约操作，并将结果返回给指定的根进程root。该函数的参数包括sbuf（输入缓冲区）、rbuf（输出缓冲区，仅在根进程中使用）、count（每个进程的参数数量）、datatype（数据类型）、op（操作类型）、root（执行接收数据的进程的排名）和comm（通信器）。
    
- MPI_Allreduce函数是MPI_Reduce的变体，将全局操作的结果返回给所有进程，因此它没有根参数root，且rbuf在所有进程中使用。
    
- MPI_Reduce_scatter函数也不需要根参数，它执行归约操作并将结果分散到各个进程，每个进程可以接收不同数量的结果。
    
- MPI_Scan函数执行一系列部分全局归约操作，每个进程将全局操作的结果发送给从零到自己的进程。与MPI_Allreduce具有相同的参数。

这些集体函数用于处理不同类型的归约操作，如全局求和、平均值、最大值、最小值等。用户还可以通过MPI_Op_create来定义自定义操作以满足特定需求。这些函数在并行计算中广泛应用，用于处理多个进程的数据合并和协作。

代刷了六道题目:
MPI3Coll23 , MPI5Comm33,36,39,46,47

# 非阻塞集体函数

- [**MPI_Iallgather**](https://learn.microsoft.com/zh-cn/message-passing-interface/mpi-iallgather-function)  
    从组的所有成员收集数据，并以非阻止方式将数据发送到组的所有成员。
    
- [**MPI_Iallreduce**](https://learn.microsoft.com/zh-cn/message-passing-interface/mpi-iallreduce-function)  
    将所有进程中的值组合在一起，并将结果以非阻塞方式分发回所有进程。
    
- [**MPI_Ibarrier**](https://learn.microsoft.com/zh-cn/message-passing-interface/mpi-ibarrier-function)  
    以非阻塞方式跨组的所有成员执行屏障同步。
    
- [**MPI_Ibcast**](https://learn.microsoft.com/zh-cn/message-passing-interface/mpi-ibcast-function)  
    以非阻塞方式将消息从进程（排名为“root”）广播到通信器所有其他进程。
    
- [**MPI_Igather**](https://learn.microsoft.com/zh-cn/message-passing-interface/mpi-igather-function)  
    以非阻塞方式将数据从组的所有成员收集到一个成员。
    
- [**MPI_Igatherv**](https://learn.microsoft.com/zh-cn/message-passing-interface/mpi-igatherv-function)  
    以非阻塞方式将组的所有成员中的变量数据收集到一个成员。
    
- [**MPI_Ireduce**](https://learn.microsoft.com/zh-cn/message-passing-interface/mpi-ireduce-function)  
    执行全局化简操作 (，例如，以非阻塞方式跨组的所有成员执行总和、最大值或逻辑和) 。
    
- [**MPI_Iscatter**](https://learn.microsoft.com/zh-cn/message-passing-interface/mpi-iscatter-function)  
    以非阻塞方式将一个成员中的数据分散在组的所有成员中。 此函数执行 [**由MPI_Igather**](https://learn.microsoft.com/zh-cn/message-passing-interface/mpi-igather-function)函数执行的操作的反函数。
    
- [**MPI_Iscatterv**](https://learn.microsoft.com/zh-cn/message-passing-interface/mpi-iscatterv-function)

==微软文档  中都有介绍!!!!!!==