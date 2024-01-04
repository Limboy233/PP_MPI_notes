在MPI库中，有一系列函数用于定义新的数据类型（派生数据类型，derived datatypes），这些类型可用于简化和加速复杂数据的传输。复杂数据的例子包括由不同数据类型的字段组成的结构以及多维数组的部分（例如矩阵的某些列）。为了能够同时考虑这两个特征，新类型与一系列基本类型和一系列偏移量（displacements）相关联。因此，新类型可以包含不同基本类型的元素，并且这些元素可能不是紧凑排列的，它们之间可能有一些偏移（可以是正偏移或负偏移）。

用户可以使用以下MPI函数定义新数据类型，它们对**派生数据类型的定义**提供了不同的方式。

1. **MPI_Type_contiguous**: 这是最简单的MPI函数之一，用于创建一个由`count`个==紧凑排列==的`oldtype`类型元素组成的新类型`newtype`。----> 内部应该是用**联合体实现的union**
```C
    MPI_Type_contiguous(int count, MPI_Datatype oldtype, MPI_Datatype* newtype)
```
![[Pasted image 20231018173611.png]]

1. **MPI_Type_vector**: 这个函数允许您创建一个新类型，
- `count`：块的数量。
- `blocklength`：每个块中的元素数量。
- `stride`：以 `oldtype` 为单位的每个块的起始之间的距离。
- `oldtype`：块中元素的基本数据类型。
- `newtype`：输出参数，新创建的数据类型。

==重点:== block 的意思是空掉 , 比如在通信传输中,你用了如下数据类型,那么它会检索sendbuf,检索长度为下面我们构造的总长度(块+空格),然后传输不是空格的块,空格就跳过不传,例子:
![[Pasted image 20231229170254.png]]
==然后,对于通讯函数,比如以我们上文自定义类型 MPI_BAND_B 为例子==
**MPI_Send(B,1,MPI_BAND_B,1,0,MPI_COMM_WORLD);**
====
==我们很容易在MPI_Send 的count 里 填P ,     但是不是这样的==
**这代表 我要传输 P * sizeof(MPI_BAND_B) 个数据 , 但是我们已经有了空格 所以这代表我们要传 P*Q 个 ,,,,,, 但实际上 只用\[Q/size\]*P 个===> 即 1 * sizeof(MPI_BAND_B) 个***

MPI_Recv(Band,P*N_b,MPI_INT,0,0,MPI_COMM_WORLD,MPI_STATUS_IGNORE);
====
在接收时一样,因为发送的是一个个int值,所以我们不能用MPI_BAND_B 要写MPI_INT 加上总共的数字

```C
MPI_Type_vector(int count, int blocklen, int stride, MPI_Datatype oldtype, MPI_Datatype* newtype)
```
![[Pasted image 20231018173736.png]]

1. **MPI_Type_indexed**: 这个函数允许您创建一个新类型，其中包含`count`个块，每个块可以包含不同数量的`oldtype`类型元素，并且每个块相对于新类型的起始位置具有不同的偏移量。块大小存储在`blocklens`数组中，偏移量存储在`displs`数组中。
```C
	MPI_Type_indexed(int count, int* blocklens, int* displs, MPI_Datatype oldtype, MPI_Datatype* newtype)
```
![[Pasted image 20231018173804.png]]
4. **MPI_Type_struct**: 这是最灵活的派生数据类型定义函数之一。它允许您创建一个新类型，其中包含`count`个块，每个块可以包含`oldtype`类型的不同元素数量，并且每个块的基本类型可以不同。每个块的偏移量相对于新类型的起始位置存储在`displs`数组中，而基本类型存储在`oldtypes`数组中。这使您可以表示复杂的数据结构。
```C
MPI_Type_struct(int count, int* blocklens, MPI_Aint* displs, MPI_Datatype* oldtypes, MPI_Datatype* newtype)
```
![[Pasted image 20231018173834.png]]

5. (在MPI-2标准中)，新增了一系列用于定义新数据类型的函数。尽管无法详尽列举所有新增的函数，但有一个函数在MPI_Type_vector和MPI_Type_indexed之间扮演了重要的角色，
它就是**MPI_Type_create_indexed_block**
:
```C
MPI_Type_create_indexed_block(int count, int blocklen, int* displs, MPI_Datatype oldtype, MPI_Datatype* newtype)
```


这个函数允许您定义一个派生类型newtype，其中包含count个块，每个块都包含blocklen个基本类型oldtype的元素，且每个块相对于新类型的起始位置都有不同的偏移量（偏移量以基本类型的元素数量为单位，并存储在大小为count的displs数组中）。这个函数的区别在于新类型中的所有块都具有相同的大小，因此使用一个整数类型的标量参数blocklen来定义它（与MPI_Type_vector中一样）。这个函数为处理块大小相同但偏移不同的数据结构提供了便利。

==(在定义完类型后的重要步骤)==:
**COMMIT**

用户可以使用这些新类型在MPI中有效地表示和传输复杂的数据。在定义新类型后，您需要使用`MPI_Type_commit`函数注册它，以便在MPI通信操作中使用。
```C
MPI_Type_commit(MPI_Datatype* datatype)
```


# MPI自定义类型销毁

MPI (Message Passing Interface)中的用户自定义数据类型可以通过释放与其相关的描述符（如MPI_Datatype类型）来销毁。
这可以通过MPI_Type_free函数来实现，其中参数datatype同时是输入和输出参数。
```C
MPI_Type_free(MPI_Datatype* datatype)
```
调用此函数后，参数datatype将被设置为MPI_DATATYPE_NULL。

==需要强调的是，即使销毁了用户定义的数据类型，使用这些数据类型派生的数据类型仍然存在。==


# MPI数据类型的性质

MPI数据类型具有两个主要属性：
1. extent（范围）和
2. size（大小）。

Extent表示该数据类型在内存中占用的字节数，包括所有块之间的空白(blank)。

Size表示数据类型的大小，即所有块的大小之和，不包括块之间的空白(blank)。


对于标准MPI数据类型（如MPI_INT、MPI_DOUBLE、MPI_CHAR等），extent和size是相同的。

## 获取extent 和 size 的函数
在MPI-1标准中，有两个函数用于获取MPI数据类型的extent和size：
1. MPI_Type_extent（获取extent）
 ```C
MPI_Type_extent(MPI_Datatype datatype, MPI_Aint* extent)
 ```
1. MPI_Type_size(获取size)
```C
MPI_Type_size(MPI_Datatype datatype, int* size)
```
第一个函数返回数据类型datatype的extent，而第二个函数返回它的size。

# Hole(初始和结束的空白位置) 的指定-->指定数据类型的开始位置与结束位置

有时，在定义新的数据类型时，可能希望为其指定初始或结束的空白区域（“hole”）。

在MPI-1标准中，可以使用特殊的基本数据类型（伪类型）==MPI_LB和MPI_UB==来实现这一点，

这些类型与任何实际数据都无关，它们的大小和extent都为零。

## 法一 : MPI_Type_struct
它们可以用作**MPI_Type_struct函数**中的“标记”，以指定新数据类型的起始和结束位置。
```C
MPI_Type_struct(int count, int* blocklens, MPI_Aint* displs, MPI_Datatype* oldtypes, MPI_Datatype* newtype)
```
例如，如果在定义类型type1时，
在**oldtypes**数组中包括了三个元素MPI_LB、MPI_INT和MPI_UB，并且定义了**blocklens和displs数组**如下：

blocklens = {1, 1, 1} displs = {-3, 0, 6}

那么类型type1将包含一个整数元素，在它之前有一个大小为3字节的初始空白，而类型的上限将位于离第一个整数元素6字节处。

因此，创建的数据类型的大小将等于MPI_INT类型的大小（通常为4字节），而extent将为9字节（不包括第6字节，因为MPI_UB类型的extent为零）。

请注意，==在组合具有明确定义标记的多个数据类型时，所有标记都会被移除，**除了最左边的MPI_LB标记和最右边的MPI_UB标记**==
![[Pasted image 20231018215353.png]]

# 法二 : MPI_Type_contiguous
如果您使用MPI_Type_contiguous函数并提供参数 **(2, type1, &type2)** 来定义新数据类型type2，那么type2将包含两个整数元素，但它的extent将是18字节。
```C
MPI_Type_contiguous(int count, MPI_Datatype oldtype, MPI_Datatype* newtype)
```
这是因为extent包括所有块之间的空白区域。根据之前的定义，type1的extent是9字节，包括初始3字节的空白和两个4字节的整数元素。
当您使用MPI_Type_contiguous来创建type2时，它将包括两个type1的副本，因此总的extent将是2倍的type1的extent，即18字节。
这意味着type2将包含两个连续的整数元素，且在内存中占用18字节的空间。
![[Pasted image 20231018233241.png]]

## 法三 : MPI_Type_create_resized
在标准MPI-1中，使用显式标记的方法来定义开始和结束的空白区域具有一个限制，即不能减小原始类型的显式边界，只能通过指定新的MPI_LB和MPI_UB标记来增加它们。因此，MPI-2标准引入了一种新的更灵活和方便的方式，用于定义新类型时指定开始和结束空白区域的方法，该方法基于MPI_Type_create_resized函数。

使用MPI_Type_create_resized函数，您可以定义一个新的类型newtype，其中包括基本类型oldtype，新的左边界位置lb和新的extent。以下是使用MPI_Type_create_resized来定义之前提到的type1类型的示例：
```C
`MPI_Datatype oldtype = MPI_INT; // 基本类型（例如MPI_INT） 
MPI_Aint lb = -3; // 新的左边界位置 
MPI_Aint extent = 9; // 新的extent 
MPI_Datatype newtype; 
MPI_Type_create_resized(oldtype, lb, extent, &newtype);
```
在此示例中，newtype包括了oldtype的定义，但左边界位置被设置为-3，extent设置为9，这与之前定义的type1相同。

MPI_Type_create_resized方法提供了更大的灵活性，因为您可以==精确地定义新类型的开始和结束空白区域==，而不受原始类型的限制。这使得在创建新的派生数据类型时更容易进行边界调整和对齐。

## 法四 : MPI_Type_get_extent
`MPI_Type_get_extent` 函数是 MPI-2 标准的一个新功能，允许同时获取 MPI 数据类型 `datatype` 的左界限 `lb` 和长度 `extent`。先前的 `MPI_Type_extent` 函数已经被标记为不建议使用，因为 `MPI_Type_get_extent` 更全面、更灵活地描述了数据的特性。
```C
MPI_Type_get_extent(MPI_Datatype datatype, MPI_Aint* lb, MPI_Aint* extent).
```
- `lb`（低边界）表示数据的左边界，单位为字节。它指示数据在数据类型 `datatype` 内部的起始字节。
- `extent` 表示数据的长度，单位为字节。它表示数据在内存中占用的字节数，包括元素之间的空白。

这些值在需要准确了解给定数据类型内部数据布局时非常有用。例如，在手动处理内存中的数据或定义复杂用户自定义数据类型时，它们可以非常有用。

因此，`MPI_Type_get_extent` 提供了获取有关 MPI 数据类型内部数据布局的信息的更加灵活和全面的方法，相比已不推荐使用的 `MPI_Type_extent` 函数。

如果你只希望指定结束空隙，可以将 `lb` 参数设置为0。这样，新类型将保留原始类型的开始空隙，但结束空隙将被设置为 `extent`。这允许你精确地控制新类型的数据布局，特别是当需要在新类型中排除开始空隙时非常有用

# MPI 的打包和解包-->(一种无需定义新的数据的方法)

MPI通信标准提供了一种称为"数据打包（pack）"的方式，==它允许在发送进程中打包数据，然后将其发送给接收进程，最后在接收进程中进行解包==(unpack)。这种方法的好处是**无需定义新的数据类型**，但缺点是==需要额外的缓冲区来存储打包的数据==

## pack
以下是MPI_Pack函数的参数及功能：
```C
int MPIAPI MPI_Pack( _In_ void *inbuf,      
					      int incount,  
					      MPI_Datatype datatype,      
     _Out_bytecap_(outsize)void *outbuf,      
					       int  outsize,     
				_Inout_    int *position,
				           MPI_Comm comm );
```
- `void* inbuf`：输入缓冲区，包含原始数据。
- `int incount`：输入缓冲区中的元素数量。
- `MPI_Datatype datatype`：输入缓冲区中元素的数据类型。
- `void* outbuf`：输出缓冲区，包含打包后的数据（这是一个输出参数）。
- `int outsize`：输出缓冲区的大小（以字节为单位）。
- `int* position`：**输出和输入参数**，表示输出缓冲区中的当前位置（以字节为单位）。
- `MPI_Comm comm`：通信子（communicator），数据将为该通信子进行打包。

!!!: 输入输出参数--->==即该API 要读取它的值,而且之后还有给它赋值==

MPI_Pack函数将来自输入缓冲区的`incount`个`datatype`类型的元素打包到输出缓冲区`outbuf`中，从指定的`position`位置开始。在执行操作后，`position`参数的值将递增，以表示输出缓冲区中的新位置。在第一次调用该函数时，`position`参数应设置为0。在为特定输出缓冲区的最后一次调用之后，`position`参数的值将等于已填充部分的大小（以字节为单位）。

请注意，==必须确保输出缓冲区`outbuf`的大小`outsize`足够大，以容纳所有打包的数据==，以使`position`参数的最终值不超过`outsize`。这是因为MPI_Pack函数不会检查缓冲区溢出，您需要自行确保缓冲区足够大。

MPI_Pack函数通常用于将数据打包为一个连续的字节流，以便通过MPI消息传递传送。
在接收端，可以使用MPI_Unpack函数来解包数据。

这种方法**适用于在不定义新的MPI数据类型的情况下发送和接收各种类型的数据。**

请注意，在打包（和随后的解包）时，必须==指定用于发送的通信器==来打包数据。

在数据传输时，打包的数据使用==特殊的MPI数据类型MPI_PACKED==，并**指定了数据的大小（以字节为单位**

## unpack
接收端可以使用MPI_Unpack函数来解包数据。MPI_Unpack函数的参数如下：
```C

int MPIAPI MPI_Unpack(
        _In_bytecount_(insize) void *inbuf,
        int                         insize,
        _Inout_ int                 *position,
  _Out_ void                        *outbuf,
        int                         outcount,
        MPI_Datatype                datatype,
        MPI_Comm                    comm
);
```

- `void* inbuf`：输入缓冲区，包含打包的数据。
- `int insize`：输入缓冲区的大小（以字节为单位）。
- `int* position`：当前在输入缓冲区中的位置（以字节为单位）（输入和输出参数）。
- `void* outbuf`：输出缓冲区，包含解包后的数据（这是一个输出参数）。
- `int outcount`：从输入缓冲区中提取的元素数量。
- `MPI_Datatype datatype`：从输入缓冲区中提取的元素的数据类型。
- `MPI_Comm comm`：与接收数据相关的通信子。


解包操作从输入缓冲区的指定位置`position`开始。在执行操作后，`position`参数的值将增加，以表示输入缓冲区中的新位置。在第一次调用MPI_Unpack函数时，应将`position`参数设置为0。
## Pack_size
此外，还有一个名为MPI_Pack_size的函数，它允许您确定存储给定数量的打包数据（`incount`个元素，数据类型为`datatype`）所需的内存大小（以字节为单位）。但要注意，返回的大小可能大于实际需要的大小，因此需要确保分配足够的内存。
```C
int MPIAPI MPI_Pack_size(
        int          incount,
        MPI_Datatype datatype,
        MPI_Comm     comm,
  _Out_ int          *size
);
```

MPI_Pack和MPI_Unpack函数通常用于在不定义新MPI数据类型的情况下发送和接收各种类型的数据，通过将数据打包成连续的字节流，然后在接收端将其解包。这个方法非常灵活，因为它不需要事先定义新的数据类型，使其非常适合各种数据传输需求。

```C
#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);

    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    // 数据准备
    int data[4] = {1, 2, 3, 4};
    int sendbuf[10];
    int recvbuf[10];

    int position = 0;
    MPI_Pack(data, 4, MPI_INT, sendbuf, 40, &position, MPI_COMM_WORLD);

    if (rank == 0) {
        // 进程0将数据打包后发送给进程1
        MPI_Send(sendbuf, position, MPI_PACKED, 1, 0, MPI_COMM_WORLD);
        printf("进程0发送数据到进程1\n");
    } else if (rank == 1) {
        // 进程1接收从进程0发送的数据
        MPI_Recv(recvbuf, 10, MPI_INT, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);

        int recv_count;
        position = 0;
        MPI_Unpack(recvbuf, 10, &position, &recv_count, 1, MPI_COMM_WORLD);
        printf("进程1接收到的数据：");
        for (int i = 0; i < recv_count; i++) {
            printf("%d ", recvbuf[i]);
        }
        printf("\n");
    }

    MPI_Finalize();
    return 0;
}

```