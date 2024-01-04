MPI-2引入了对并行文件输入输出的支持，这是一个重要的新特性。在并行程序中，通常需要从文件中读取输入数据，并将结果写入文件。在MPI-1中，由于缺乏并行文件访问的功能，通常需要在一个特定的进程（通常是主进程）中读取数据，然后将数据传输到其他并行应用程序中的进程。类似地，为了保存结果，需要首先将结果传输到某个进程，然后在该进程中将结果写入文件。如果并行应用程序需要多个进程访问同一个文件，那么为了确保正确的文件访问，需要执行特殊的同步操作。

有了MPI-2标准，**现在可以在每个进程中读取或写入文件数据，而无需特别的文件访问同步操作**。这为并行文件输入输出提供了更方便的方式，从而减少了编写并行文件访问代码时的复杂性。MPI-2的这一特性使得每个进程都可以直接处理文件数据，而==无需额外的数据传输和同步操作==，大大简化了并行文件处理的流程。

MPI-2标准引入了灵活多样的文件输入输出（I/O）机制。
# I/O
这些机制包括
**本地**和**集体**文件访问函数，
并为每种类型的访问提供==三种==不同的定位方式。

这些定位方式可以是明确的，通过在函数的特殊参数中
1. **指定文件位置**，或者可以
2. 是隐式的，使用每个进程的
	1) **个别文件指针**或
	2) **共享文件指针**。

==这两种文件指针可以同时用于本地和集体文件访问。==

此外，所有描述的并行文件访问方式都有两种实现：
(blocking,unblocking)
==阻塞==和==非阻塞==，类似于数据传输操作的阻塞和非阻塞操作之间的差异。

通过结合这些不同的机制，可以获得24种（= 2 x 2 x 3 x 2）并行文件访问方式的组合：

- 读取或写入（2种方式），读取的函数名称中包含 "read"，写入的函数名称中包含 "write"；

- 本地或集体访问（2种方式），集体访问函数名称中添加 "all"（除了使用共享文件指针的函数，本地访问的名称中使用 "shared"，而集体访问的名称中使用 "ordered"）；

- 明确定位、个别文件指钘或共享文件指针（3种方式），明确定位的函数名称中添加 "at"，使用共享文件指针的函数名称中添加 "shared" 或 "ordered"；

- 阻塞或非阻塞访问（2种方式），对于本地函数，非阻塞访问的函数名称中添加 "i"（例如 "iread" 和 "iwrite"），集体非阻塞函数是成对出现的，第一个函数名称以 "begin" 结尾，第二个函数名称以 "end" 结尾。

每个文件访问选项都有相应的MPI函数，或者对于集体非阻塞访问，有一对函数，总共有30个函数。

通过函数的名称，可以轻松确定它们与文件访问的关联

。例如，MPI_File_read_at_all函数执行阻塞式读取（read），基于明确位置定位（at），并且是集体的（all）。

而MPI_File_iwrite_shared函数执行非阻塞式写入（iwrite），使用共享文件指针，并且是本地的（本地和集体访问的特性由 "shared" 字词确定）。

==最基本的文件读取和写入函数是**MPI_File_read和MPI_File_write**，它们是阻塞的、本地的，使用个别文件指针。==

表格1列出了与文件访问相关的所有函数名称。这些函数按照如下的分类进行分组： 
1) "Positioning"（定位方式）（明确位置定位、本地文件指针或共享文件指针）、 
2) "Access"（访问方式）（阻塞或非阻塞）、 
3) "Coordination"（协同方式）（本地或集体函数）。

![[Pasted image 20231101145809.png]]

# 有明确定位功能的阻塞函数
以下是具有明确定位功能的阻塞函数（MPI_File_read_at，MPI_File_write_at，MPI_File_read_at_all，MPI_File_write_at_all）的参数列表：

- MPI_File f - 文件变量（文件描述符）;
- MPI_Offset offset - 读取/写入数据的文件位置;
- void* buf - 用于存储已读取数据（对于读取功能）或要写入的数据（对于写入功能）的缓冲区；对于读取功能，这是一个输出参数;
- int count - 缓冲区buf的大小（以datatype类型的元素为单位）;
- MPI_Datatype datatype - 缓冲区buf中元素的类型;
- MPI_Status* status - 有关已执行读取/写入操作的附加信息（输出参数）。

MPI_File类型的文件描述符f是使用MPI_File_open函数打开文件时确定的。该函数是集体的，应该为某个通信器的所有进程调用；结果是该通信器的所有进程都能够访问该文件。

MPI_File的文件描述符f是在通过MPI_File_open函数打开文件时定义的。该函数是一个集体操作，必须由某个通信器中的所有进程调用。调用后，该通信器中的所有进程都能够访问文件。有关MPI_File_open函数的详细描述请参见第1.3.3节。

MPI_Offset类型与之前讨论的MPI_Aint类型类似（参见第1.2.6节）。它用于存储不同文件位置之间的偏移量，并实现为具有足够大小以存储磁盘地址空间中任何可能偏移的有符号整数类型。通过status输出参数，您可以确定实际读取/写入的元素数量（通常需要使用MPI_Get_count函数，参见第1.2.1节），并获取错误代码（使用MPI_ERROR字段）。文件访问函数中不使用MPI_Status结构的其他字段。调用具有显式定位的函数不会影响本地和全局文件指针的当前位置。
# 文件描述符f
MPI_File的文件描述符f是在通过**MPI_File_open函数打开文件时定义的**。该函数是一个集体操作，必须由某个通信器中的所有进程调用。调用后，该通信器中的所有进程都能够访问文件。

有关MPI_File_open函数的详细描述请参见第1.3.3节。

MPI_Offset类型与之前讨论的MPI_Aint类型类似（参见第1.2.6节）。它用于存储不同文件位置之间的偏移量，并实现为具有足够大小以存储磁盘地址空间中任何可能偏移的有符号整数类型。通过status输出参数，您可以确定实际读取/写入的元素数量（通常需要使用MPI_Get_count函数，参见第1.2.1节），并获取错误代码（使用MPI_ERROR字段）。文件访问函数中不使用MPI_Status结构的其他字段。调用具有显式定位的函数不会影响本地和全局文件指针的当前位置。


==MPI_Offest 是用bit 为单位的--->表示bit的个数(可以强制转换为int类型)
MPI_File_get_position 得到的文件位置,是与文件开头的距离 ---> 一定大于零==

==均已 bit 为文件位置统计单位==
# MPI_File_seek
此外，您还可以使用**MPI_File_seek**（对于本地指针）和**MPI_File_seek_shared**（对于共享指针）函数显式设置文件指针的位置。这两个函数具有相同的参数集：

1. MPI_File f - 文件变量;
2. MPI_Offset offset - 所需的文件指针偏移量（可以为正数或负数）;
3. int whence - 所使用的偏移模式。

为指定偏移模式whence，有三个常量可供使用：

1. MPI_SEEK_SET - 从文件数据视图的起始位置开始偏移;
2. MPI_SEEK_CUR - 从当前文件指针位置开始偏移;
3. MPI_SEEK_END - 从文件末尾标记开始偏移。在MPI_SEEK_SET模式下，只允许正偏移值，在MPI_SEEK_END模式下只允许负偏移值。

# MPI\_File\_get\_position(\_share\_)
用于确定本地和共享文件指针当前位置的函数是**MPI_File_get_position**和**MPI_File_get_position_shared**，它们具有相同的参数集：

1. MPI_File f - 文件变量;
2. MPI_Offset* offset - 文件指针的当前位置（输出参数）。
 
# MPI_File_read_shared и MPI_File_write_shared
 
 需要注意的是，当使用与共享文件指针相关的本地函数（MPI_File_read_shared和MPI_File_write_shared）时，不会执行额外的同步，也就是说，数据读取的顺序将取决于在不同进程中调用这些函数的时间点。
# MPI_File_read_ordered и MPI_File_write_ordered
与==共享文件指针==相关的集体函数（MPI_File_read_ordered和MPI_File_write_ordered）提供所需的同步。
在调用它们时（与任何集体函数一样，必须**在为其打开文件的通信器中的所有进程中调用**），使用共享指针执行的读取/写入数据的操作是==按照在该通信器中进程的等级递增==的顺序依次执行的。

对于前面讨论的所有函数，offset位置都以文件etype的基本类型的元素为单位，同时考虑到在设置文件数据视图时指定的初始位移disp（请参阅第1.3.3节中MPI_File_set_view函数的描述）。
 
# MPI_File_get_byte_offset 

 要将偏移量offset（以基本类型的元素为单位）转换为从文件开头算起的以字节为单位的位移disp，可以使用辅助函数MPI_File_get_byte_offset(MPI_File f, MPI_Offset offset, MPI_Offset* disp)。

在某些情况下，MPI_File_get_size(MPI_File f, MPI_Offset* size)函数也很有用，它会将文件f的大小以字节为单位存储在参数size中。
# MPI_File_set_size
   与其成对的是MPI_File_set_size(MPI_File f, MPI_Offset size)函数，它==允许更改文件f的大小，将其设置为size字节==。
   1) 缩小文件的大小（在这种情况下，删除文件的末尾部分），也可以
   2) 增加文件的大小（在这种情况下，添加到末尾的内容是未定义的）。
   
   MPI_File_set_size函数是一个==集体操作==，必须由打开文件的通信子中的所有进程调用，并且**它们都必须传递相同的size参数值**。
   
	
除了MPI-2中不同的文件读写组织方式外，还提供了一种灵活的文件数据视图（file view）配置方式，有时也称为“**文件数据图像**”，

它从简单的将文件视为一系列连续字节的情况，一直到复杂的情况，其中文件视图可以包括元素组，这些元素可以不必连续存在（允许在元素之间和开头末尾有空白）。此外，每个进程可以定义自己的文件数据视图。

这些功能将在下一节详细讨论。----> ==WIN==

在MPI-6File任务组（参见第2.6节）的执行中，您可以了解并行文件输入和输出的大多数功能。

在这个任务组的第一个子组（任务MPI6File1-MPI6File8）中

，研究本地文件操作，第二个子组（MPI6File9-MPI6File16）中研究集体文件操作

，最后一个子组（MPI6File17-MPI6File30）涉及不同方法来定义复杂的文件数据视图。

在前两个子组的任务中，使用了所有三种定位机制，它们基于明确指定位置或使用本地或全局文件指针。

在第三个子组中，主要使用了集体文件操作，这些操作主要使用本地文件指针。

MPI6File任务组之外还有一些额外的功能，涉及非阻塞文件访问，这种情况相对较少见

![[Pasted image 20231125165254.png]]

# **MPI_File_set_view**
https://cvw.cac.cornell.edu/parallel-io/mpi-io/file-view-examples
![[Pasted image 20231201023841.png]]![[Pasted image 20231201024125.png]]
![[Pasted image 20231201024150.png]]
![[Pasted image 20231201024221.png]]
![[Pasted image 20231201024513.png]]

```C
#include "pt4.h"

  

#include "mpi.h"

  

void Solve()

{

    Task("MPI6File24");

    int flag;

    MPI_Initialized(&flag);

    if (flag == 0)

        return;

    int rank, size;

    MPI_Comm_size(MPI_COMM_WORLD, &size);

    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

    //string name;

    char name[20];

    if (rank == 0) pt >> name;

    //然后用一个广播把文件名播给slave process

    MPI_Bcast(name,20,MPI_CHAR, 0, MPI_COMM_WORLD);

    MPI_File F ;

    ShowLine(name);

    MPI_File_open(MPI_COMM_WORLD, name, MPI_MODE_CREATE | MPI_MODE_RDWR, MPI_INFO_NULL, &F);

  

    int size_arr=4;

    double *arr = new double[size_arr];

    for(int i=0;i<size_arr;i++){

        pt >> arr[i];

    }

    int site;

    MPI_Type_size(MPI_DOUBLE, &site);

  

    MPI_Datatype data;

    MPI_Type_vector(4, 2, 2*size, MPI_DOUBLE, &data);

  

    MPI_File_set_view(F, (size-rank-1)*2*sizeof(double), MPI_DOUBLE, data, "native", MPI_INFO_NULL);

    MPI_File_write_all(F, arr, 4, MPI_DOUBLE, MPI_STATUS_IGNORE);

    MPI_File_close(&F);

}
```
![[Pasted image 20231201024705.png]]
```C
#include "pt4.h"

  

#include "mpi.h"

  

void Solve()

{

    Task("MPI6File22");

    int flag;

    MPI_Initialized(&flag);

    if (flag == 0)

        return;

    int rank, size;

    MPI_Comm_size(MPI_COMM_WORLD, &size);

    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

    //string name;

    char name[20];

    if (rank == 0) pt >> name;

    //然后用一个广播把文件名播给slave process

    MPI_Bcast(name,20,MPI_CHAR, 0, MPI_COMM_WORLD);

    MPI_File F ;

    ShowLine(name);

    MPI_File_open(MPI_COMM_WORLD, name, MPI_MODE_CREATE | MPI_MODE_RDWR, MPI_INFO_NULL, &F);

    int size_arr=3;

    int *arr = new int[size_arr];

    for(int i=0;i<size_arr;i++){

        pt >> arr[i];

    }

    int site;

    MPI_Type_size(MPI_INT, &site);

  

    MPI_Datatype data;

    MPI_Type_vector(size, 1, size, MPI_INT, &data);

  

    MPI_File_set_view(F, (size - rank - 1)*site, MPI_INT, data, "native", MPI_INFO_NULL);

    MPI_File_write_all(F, arr, 3, MPI_INT, MPI_STATUS_IGNORE);

    MPI_File_close(&F);

  

}
```


