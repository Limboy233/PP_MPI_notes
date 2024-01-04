==非阻塞消息传递涉及发送和接收操作，这些操作只会启动相应的操作，然后立即返回，返回一个MPI特殊对象——通信请求（request）类型的MPI_Request==

以后可以使用这个对象MPI_Request来查询操作的状态，
1) 要么使用Wait组的阻塞函数，等待操作完全结束，
2) 要么使用Test组的非阻塞函数来查询。

**当非阻塞操作完成时，与它关联的通信请求将被"释放"，其值将变为MPI_REQUEST_NULL，这通常是通过Wait组的函数返回，或通过返回已完成操作信息的Test组函数调用来实现的**

# Immediate
与阻塞传输一样，MPI提供了**四个非阻塞发送消息函数和一个非阻塞接收消息函数**。这些函数的名称与相应的阻塞传输函数的名称相同，但前面加了一个"I"前缀，代表immediate。

## 发送
==MPI_Isend、MPI_Ibsend、MPI_Issend、MPI_Irsend==函数在四种不同的模式下启动非阻塞传输操作，并返回与该传输相关的请求对象（MPI_Request）。请求对象是这些函数的最后一个参数，前面的参数与阻塞传输函数的参数相同。在非阻塞操作结束之前，不能重新使用缓冲区buf。

## 接收
**MPI_Irecv**函数启动非阻塞接收操作，并返回与之相关的请求对象（MPI_Request）。它的参数与MPI_Recv函数中的参数相同，但MPI_Irecv的请求参数是最后一个参数。在非阻塞操作结束之前，不能使用buf来读取接收到的数据

# Wait

1. MPI_Wait(MPI_Request* request, MPI_Status* status),
2. MPI_Waitall(int count, MPI_Request* requests, MPI_Status* statuses),
3. MPI_Waitany(int count, MPI_Request* requests, int* index, MPI_Status* status),
4. MPI_Waitsome(int count, MPI_Request* requests, int* outcount, int* indices, MPI_Status* statuses).
在Wait组中，有四个阻塞函数：MPI_Wait、MPI_Waitall、MPI_Waitany和MPI_Waitsome。

1) MPI_Wait函数等待与请求相关的非阻塞传输或接收操作完成，并返回可选的状态参数，通常仅在非阻塞接收时使用。

2) 其他函数接受一个请求数组requests，其大小为count。

3) MPI_Waitall函数会阻塞程序，直到与指定请求相关的所有操作完成（在参数statuses中返回每个操作的额外信息）。
   
4) MPI_Waitany函数会阻塞程序，直到与指定的请求中的任何一个操作完成。
5) 
6) MPI_Waitsome函数与MPI_Waitany类似，但可以返回多个完成操作的信息。

# Test

1. MPI_Test(MPI_Request* request, int* flag, MPI_Status* status),
2. MPI_Testall(int count, MPI_Request* requests, int* flag, MPI_Status* statuses),
3. MPI_Testany(int count, MPI_Request* requests, int* index, int* flag, MPI_Status* status), 
4.MPI_Testsome(int count, MPI_Request* requests, int* outcount, int* indices, MPI_Status* statuses).

在Test组中也有四个函数：MPI_Test、MPI_Testall、MPI_Testany和MPI_Testsome。

1) MPI_Test函数检查与请求相关的非阻塞传输或接收操作是否已经完成，并立即返回，将检查结果存储在标志参数（flag）中。如果操作已完成，flag参数将返回非零值，此时与请求关联的对象将变为MPI_REQUEST_NULL，并在状态参数中返回操作的附加信息。
2) 其他函数的行为与MPI_Test类似，检查与请求数组requests相关的非阻塞操作，并立即返回结果。

# MPI_Iprobe<---MPI_Probe
此外，还有MPI_probe函数的非阻塞版本MPI_Iprobe，它不会等待接收操作完成，而是立即返回是否有消息可供接收。

如果没有消息可供接收，flag参数将返回0。

它还可以返回与消息有关的信息，类似于MPI_Iprobe的阻塞版本，但它是非阻塞的。

==这个函数通常被用在一种情况下，即在发送的消息中元素的数量是未知的==


==一个十分重要的高级技巧----> 在性能优化方面==
# "延迟请求" (是持久性的)

"延迟请求"是MPI中一种特殊的非阻塞操作。

这些请求是通过使用函数MPI_Send_init、MPI_Bsend_init、MPI_Ssend_init、MPI_Rsend_init和MPI_Recv_init来生成的，
(它们具有与之前讨论的非阻塞函数MPI_Isend、MPI_Ibsend、MPI_Issend、MPI_Irsend、MPI_Irecv相同的参数。)

==不同之处在于==，"延迟请求"并不会立即执行相应的操作，它们只返回一个"延迟请求"，即MPI_Request类型的参数，包含了所需操作的所有设置。
**然后让操作者在合适的时候发出请求**
## 启动延时请求
要启动使用Init函数生成的"延迟请求"，
可以使用**MPI_Start和MPI_Startall**函数。
1) MPI_Start函数用于以非阻塞模式启动与请求相关的操作，
2) MPI_Startall函数用于以非阻塞模式启动与请求数组（大小为count）相关的所有操作。

**这些已经启动的操作可以使用之前讨论的Wait和Test函数来检查其完成状态。**



## 持久性的请求"（persistent）
值得注意的是，通过Init函数生成的请求是"持久性的"（persistent）。
这意味着一旦相关的非阻塞操作完成，与之相关的请求不会自动重置为MPI_REQUEST_NULL，它们仍然有效。

==因此，可以在以后多次重复使用"延迟请求"==，只需再次调用MPI_Start或MPI_Startall函数（在此之前可以更改包含要发送数据的缓冲区buf）。

要释放通过Init函数生成的请求，可以使用MPI_Request_free函数




# 时间测量

MPI还提供了两个特殊的用于测量时间的函数，用于性能分析和优化。

1) MPI_Wtime函数返回自某一时间点以来经过的时间（以秒为单位），可以用于测量程序中某个部分的执行时间。

2) MPI_Wtick函数返回两个连续时钟滴答之间的时间差，用于测量时间测量的精度。这些函数对于MPI程序中的时间度量非常有用。