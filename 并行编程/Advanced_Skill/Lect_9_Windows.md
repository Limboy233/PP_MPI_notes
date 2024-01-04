![[Pasted image 20231201101216.png]]

MPI windows are created with [MPI_Win_create](https://rookiehpc.org/mpi/docs/mpi_win_create/index.html), [MPI_Win_create_dynamic](https://rookiehpc.org/mpi/docs/mpi_win_create_dynamic/index.html), [MPI_Win_allocate](https://rookiehpc.org/mpi/docs/mpi_win_allocate/index.html) or [MPI_Win_allocate_shared](https://rookiehpc.org/mpi/docs/mpi_win_allocate_shared/index.html). 



MPI windows are then destroyed with [MPI_Win_free](https://rookiehpc.org/mpi/docs/mpi_win_free/index.html)

==**Windows: 相当于建立了一个数据中转站!!!!!!!!**==

单边通信是 MPI-2 标准中出现的通信方式，允许在不需要两个交互方特定操作的情况下，在进程之间传输数据。在传统的两个进程交互中，通常需要在发送方（源进程或发送者）调用发送函数（例如 MPI_Send 或其各种阻塞和非阻塞变体），并在接收方（目标进程或接收者）调用接收函数（例如 MPI_Recv 或其非阻塞变体）。

==我们之前也实现过的,但是用的是==---> **下面这种方式实现(可变消息传输)**
然而，在某些情况下，发送方可能不知道其他进程需要它的哪些数据，或者接收方可能不知道其他进程将发送哪些数据。在后一种情况下，虽然存在 MPI_ANY_SOURCE 和 MPI_ANY_TAG 参数，允许从任意源接收数据，并根据传递的消息标签（msgtag）确定接收到的数据类型。但是，通常需要先对接收到的消息进行预先分析（通过调用 MPI_Probe 函数或其非阻塞变体），从而增加了接收方算法的复杂性。

===> **极端的是:
如果发送方不知道哪些进程可能需要它的数据**，那么传统的 MPI "发送-接收" 机制就不太可能实现。这为一方通信提供了空间，它允许发送数据而不需要等待接收方显式地准备好接收.

在多线程编程技术中，单向通信是信息交换的标准方式：一个线程可以将数据写入共享内存的某个区域，随后任何其他线程都可以访问该内存并获取所需的数据。为了在 MPI 技术中实现这种类型的通信，需要让并行 MPI 应用程序的进程能够确定其 "共享区域"，其他同一应用程序的进程可以直接访问这些区域。因此，==MPI 中的一种单向通信机制（one-sided communications）也被称为远程内存访问（Remote Memory Access，RMA）==。某个进程的一块内存区域，其他任何并行应用程序的进程都可以访问，被称为**窗口（window）**。

# 为了创建窗口
MPI 提供了一个集体函数 `MPI_Win_create(void* base, MPI_Aint size, int disp_unit, MPI_Info info, MPI_Comm comm, MPI_Win* win)`。

此函数需要在通信器 `comm` 中的所有进程上调用。调用此函数一次可以在所有进程中创建共享内存区域（即窗口），所有这些区域都将与返回的类型为 `MPI_Win` 的窗口描述符 `win` 相关联。==利用此描述符，任何进程都可以访问任何其他进程的窗口==。
函数 `MPI_Win_create` 的前两个参数为此进程的窗口基地址 `base` 和窗口大小 `size`（以字节为单位）。在不需要为某些进程创建窗口的情况下，可以将此进程的 `size` 参数设置为零。

# 在不同进程中定义的窗口可以具有不同的大小。

如果不需要为某些进程创建窗口，则在调用 `MPI_Win_create` 函数时，==将对应进程的 `size` 参数设为零即可==。
`MPI_Win_create` 函数的第三个参数 `disp_unit` 用于简化访问窗口时所使用的地址运算。在访问函数中，从窗口开头的偏移量会自动乘以 `disp_unit` 的值。

如果窗口用于存储某个数组的元素，则将数组元素的大小设置为 `disp_unit` 参数的值会更方便。这样，之后在访问函数中可以指定数组元素的索引。如果将 `disp_unit` 参数设置为 1，则在访问函数中需要==以字节为单位==指定偏移量。

第四个参数 `info` 允许关联额外的信息到创建的窗口。通常情况下，该参数不被使用，并且可以设为常量 `MPI_INFO_NULL`。

# MPI 库中提供了三个用于访问窗口的函数：

`MPI_Get`（读取访问）、`MPI_Put`（写入访问）和 `MPI_Accumulate`（累积访问）。
调用这些函数的进程被称为发起进程（origin process），包含要访问窗口的进程被称为目标进程（target process）。
==在单向通信中，发起进程提供数据，而目标进程扮演被动角色，无需执行任何特殊操作。==

发起进程和目标进程都可以是数据的来源和接收者。
::::
1) 如果使用 `MPI_Get` 操作，则发起进程是数据的接收者，而目标进程是数据的提供者；
2) 如果使用 `MPI_Put` 或 `MPI_Accumulate` 操作，则发起进程是数据的提供者，而目标进程是数据的接收者。

大多数访问函数（MPI_Get、MPI_Put、MPI_Accumulate）的参数是相同的。前三个参数确定了发起进程端的数据：void *origin_addr、int origin_count、MPI_Datatype origin_datatype。
其中，第一个参数是数据缓冲区的起始地址，
第二个参数是缓冲区中的元素数量，
第三个参数是缓冲区中元素的类型。

接下来的四个参数确定了目标进程端的数据：int target_rank、MPI_Aint target_disp、int target_count、MPI_Datatype target_datatype。
第一个参数表示目标进程的等级，
第二个参数是从目标窗口开始的偏移量（以 `MPI_Win_create` 函数中指定的 `disp_unit` 单位为准，请参阅前面关于 `MPI_Win_create` 函数的描述），
第三个和第四个参数确定了访问窗口的数量和类型（值的数量以指定类型的元素为单位，而不是与 `disp_unit` 单位有关）。如果发起进程和目标窗口的元素类型相匹配（通常情况下是这样），则起始数量（origin_count）和目标数量（target_count）应该相匹配。所有访问函数的最后一个参数都是 MPI_Win 类型的窗口描述符。

MPI_Accumulate 函数与前面的参数相同，并具有一个==额外的参数 MPI_Op==，在窗口参数之前。

MPI_Op 参数确定要用于更改窗口内容的操作。
此操作将应用于发起进程缓冲区中的元素和目标进程窗口中相应的元素，并将操作结果写入窗口的相应元素。可以使用 MPI 库中定义的任何**标准归约操作**（请参阅第 1.2.5 节）。不允许使用用户定义的操作。


MPI_Accumulate 函数的**设计允许在一个窗口中通过多个发起进程进行安全使用**（MPI_Put 函数不具备这种特性：如果两个发起进程在一个访问周期中尝试将其数据写入同一个目标进程的窗口，那么该窗口将仅保留其中一个发起进程的数据，而无法预先知道保留哪一个）。
调用 MPI_Get 函数会更改发起进程缓冲区的内容；
而调用 MPI_Put 和 MPI_Accumulate 函数会更改目标进程窗口中指定区域的内容。


# 单向通信机制中的一个重要方面是对窗口访问的同步。
与标准的双向 "发送-接收" 交换方式不同，**目标进程**在单向交换时不执行任何特殊操作，因此需要额外的工作来协调位于窗口中的数据访问。

特别是，如果目标进程扮演数据接收者的角色（在这种情况下，它被称为主动目标进程（active target）），则它必须知道何时可以访问窗口以读取收到的数据；

如果数据接收者是发起进程，则它必须知道何时可以访问从目标进程获取的数据。

在这两种情况下都需要同步。

唯一的例外是，对于被称为被动目标进程（passive target）的单向交互方式，目标进程不需要同步。
被动目标进程的窗口被用作其他并行应用程序进程访问数据的存储区域（类似于多线程编程中使用的共享内存模型），在这种情况下，不需要目标进程访问其自身的窗口

# 访问时期(access epoch)

函数同步允许在初始化进程的一侧设置所谓的访问时期(access epoch)，以及在活动目标进程的一侧设置提供时期(exposure epoch)。

在访问时期（与相应的提供时期相协调）执行的所有单向交互的结果将只在该时期结束后对进程可用。

换句话说，只有在当前访问时期结束之前，初始化进程才不应该使用MPI_Get函数获得的数据，而在提供时期结束之前，活动目标进程才不应该使用MPI_Put或MPI_Accumulate函数读取已通过这些函数放入其窗口的数据。

MPI_Win_fence(int assert, MPI_Win win)提供了最简单的同步方式（“fence”一词可以译为“栅栏”或“围栏”）。

这是一个集体函数，需要在定义了窗口win的所有进程中调用。

除了确定时期访问的窗口参数win外，此函数还包括assert参数，该参数可以包含一组常量，用于说明确定时期访问的性质（例如，MPI_MODE_NOPUT常量表示在此期间不执行与MPI_Put或MPI_Accumulate函数修改窗口内容有关的操作）。

这些常量允许MPI环境优化执行单向通信时的操作。如果不需要进一步确定访问时期的性质，则assert参数应设置为0。

值得注意的是，assert参数也包含在其他同步函数中（稍后会描述）。

第一次调用MPI_Win_fence函数开始了给定窗口的第一个访问时期（以及相应的提供时期）。==每次对此函数的后续调用都会结束之前的访问时期（以及相应的提供时期），同时开始新的访问时期（以及相应的提供时期）==。

因此，使用此同步函数时，每个带有窗口的进程都需要**至少执行两次此函数调用**。

当多个进程在访问时期充当活动目标进程时，会使用这种同步方式。它的主要限制在于它是全局性的：为定义了给定窗口的所有进程设置了一个同步时期

Tips: 
如果MPI_Win_fence函数只被调用来结束最后一个访问时期，那么可以通过在assert参数中指定特殊值MPI_MODE_NOSUCCEED来标记这一事实
```C
MPI_Win_fence(MPI_MODE_NOSUCCEED, w);
```



# 以下是关于另一种同步方法的描述，它允许仅针对某些定义了窗口的进程进行本地定义访问期间（以及相关的提供访问期间）。

但是，为了实现这种灵活性，需要使用更复杂的设置来管理访问和提供访问期间，其中开始和结束访问以及提供访问的周期都需要特殊函数：

- 函数 MPI_Win_start(MPI_Group group, int assert, MPI_Win win) 开始访问期间，适用于调用此函数的所有进程。其中指定了组 group 的进程，这些进程可以充当此期间的活动目标进程。
    
- 函数 MPI_Win_complete(MPI_Win win) 终止由 MPI_Win_start 开始的访问期间。
    
- 函数 MPI_Win_post(MPI_Group group, int assert, MPI_Win win) 开始提供访问期间，适用于调用此函数的所有进程。其中指定了组 group 的进程，这些进程可以充当此期间的初始化进程。
    
- 函数 MPI_Win_wait(MPI_Win win) 终止由 MPI_Win_post 开始的提供访问期间。从 MPI_Win_wait 函数中返回表示所有初始化进程都已通过调用 MPI_Win_complete 终止了访问期间

Tips: 
函数MPI_Win_test是MPI_Win_wait的非阻塞版本，它包含额外的输出参数flag。函数MPI_Win_test(MPI_Win win, int* flag) 如果返回非零的flag值，表示所有的初始化进程都已经完成了访问期间，这也意味着提供访问期间的结束。在当前提供访问期间内，只有在前一个MPI_Win_test调用返回值为0时才应该再次调用MPI_Win_test函数。需要注意的是，在教学任务中很少使用MPI_Win_test函数。

具体使用看(康奈尔大学的这个):
https://cvw.cac.cornell.edu/mpionesided/synchronization-calls/post-start-complete-wait

和这个官网的图![[Pasted image 20231201185151.png]]
https://www.mpi-forum.org/docs/mpi-3.1/mpi31-report/node281.htm
Tips: 其中的put 表示 MPI_Put 函数,是对窗口的操作

# 我们现在来解释原因

POST-WAIT ----->对应的GROUP(==括号里的数字==): 是说我这个进程会暴露给GROUP中的进程访问我的Windows中的资源

START-complete----->对应的GROUP: 表示我要去搞(访问)哪个进程的资源
----> 叫一个epoch

POST-WAIT 曝露期间,他gruop中所有process的epoch访问完该进程后,结束Blocking(==应该是调用完post后该进程就直接开始expose了, 继续执行下方代码,遇到wait后开始等待(Blocking)他gruop中所有process的epoch访问完该进程 , 完成之后wait函数完成==)----->下方MPI_Win_wait函数功能写得也是等待epoch结束(这个是kornel大学的网站上的例子)
![[Pasted image 20231201190456.png]]


例题:
![[Pasted image 20231201191337.png]]
```C
#include "pt4.h"

  

#include "mpi.h"

  

void Solve()

{

    Task("MPI7Win19");

    int flag;

    MPI_Initialized(&flag);

    if (flag == 0)

        return;

    int rank, size;

    MPI_Comm_size(MPI_COMM_WORLD, &size);

    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

  

    double source[size-1] = {0};

    double target[1] = {0};

    MPI_Win win;

    MPI_Group world_group;

    MPI_Comm_group(MPI_COMM_WORLD, &world_group);

    MPI_Group targets_group;

    MPI_Group sources_group;

    int ranks[1]={0};

    MPI_Group_excl(world_group, 1,ranks, &sources_group);

  

    MPI_Group_incl(world_group,1,ranks,&targets_group);

  

    if(rank == 0){

        for(int i=0;i< size-1;i++){

            pt >> source[i];

        }

    }  

  
  

    MPI_Win_create(source,(size-1)*sizeof(double),sizeof(double),MPI_INFO_NULL,MPI_COMM_WORLD,&win);

    //MPI_Win_fence(0,win);

    if(rank==0){

    MPI_Win_post(sources_group, 0, win);

    MPI_Win_wait(win);

    }

    if(rank!=0){

        MPI_Win_start(targets_group, 0, win);

        MPI_Get(target,1,MPI_DOUBLE,0,size - rank - 1,1,MPI_DOUBLE,win);

        MPI_Win_complete(win);

    }

  

    MPI_Win_fence(0,win);

    if(rank!=0)

        pt << target[0];

  

  

    MPI_Group_free(&world_group);

    MPI_Group_free(&targets_group);

    MPI_Group_free(&sources_group);

    MPI_Win_free(&win);

}
```













讨论了三种不同的MPI同步方式，针对被动进程（不访问其窗口）提供了一种

# 基于锁定的同步方法。

基于锁定的同步方式主要针对被动进程，使其不需要调用任何同步函数（也就是没有提供访问期间），而同步函数专门用于初始化进程开始和结束访问期间。这种方式包含以下函数：

- `MPI_Win_lock(int lock_type, int rank, int assert, MPI_Win win)`：用于初始化进程开始一个阻塞的访问期间。
- `MPI_Win_unlock(int rank, MPI_Win win)`：用于结束一个阻塞的访问期间。

参数`lock_type`可以是`MPI_LOCK_EXCLUSIVE`（独占锁）或`MPI_LOCK_SHARED`（共享锁）。

1) 如果某个进程尝试使用独占锁访问某个目标进程，而该目标进程当前已经使用==独占锁==，那么访问将被推迟，直到先前的独占锁被释放。
2) 与独占锁不同，多个进程可以同时使用共享锁访问相同的目标进程（例如，如果初始化进程仅需要读取目标进程的窗口，则使用==共享锁==）。

详细的看:
https://cvw.cac.cornell.edu/mpionesided/synchronization-calls/lock-unlock

==如果下一个步骤依赖于上一个步骤,但可能上一个步骤中的进程还在等待锁而没有执行操作让下一个步骤拿到没有更新的数-----> **我们要使用MPI_Barrier 来Blocking让上一个步骤全部都完成!!!**==
![[Pasted image 20231201202729.png]]
```C
#include "pt4.h"

  

#include "mpi.h"

  

void Solve()

{

    Task("MPI7Win25");

    int flag;

    MPI_Initialized(&flag);

    if (flag == 0)

        return;

    int rank, size;

    MPI_Comm_size(MPI_COMM_WORLD, &size);

    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

  

    double* source = new double[5];

    int M;double B;

  

    //ues the rest ring

    if(rank % 3 ==0){

        for(int i=0;i<5;i++){

            pt >> source[i];

        }

    }

    if((rank)%3==1){

        pt >> M >> B;

    }

    MPI_Win win;

    MPI_Win_create(source,5*sizeof(double),sizeof(double),MPI_INFO_NULL,MPI_COMM_WORLD,&win);

    MPI_Barrier(MPI_COMM_WORLD);

  

    if ((rank % 3) == 1){

        MPI_Win_lock(MPI_LOCK_EXCLUSIVE,rank-1,0,win);

        MPI_Accumulate(&B, 1, MPI_DOUBLE, rank - 1, M, 1, MPI_DOUBLE, MPI_MIN, win);

        MPI_Win_unlock(rank-1,win);

    }

  

    MPI_Barrier(MPI_COMM_WORLD);

    if ((rank % 3) == 2){

        MPI_Win_lock(MPI_LOCK_EXCLUSIVE,rank-2,0,win);

        MPI_Get(source, 5, MPI_DOUBLE, rank - 2, 0, 5, MPI_DOUBLE, win);

        MPI_Win_unlock(rank-2,win);

    }

  

    MPI_Barrier(MPI_COMM_WORLD);

    if ((rank % 3) == 2){

        for (int i = 0; i < 5; ++i){

            pt << source[i];

        }

    }

}
```

MPI7Win组的任务可以让你了解单边通信机制的方方面面。在该组的第一子组（MPI7Win1–MPI7Win17）中，使用了基于`MPI_Win_fence`的最简单同步，讨论了所有三种类型的单边通信（读取、写入和修改）。在该子组的初始任务（MPI7Win1–MPI7Win6）中，窗口访问中的共享内存仅在一个进程中创建，而其他任务则在进程组或整个应用程序中创建共享内存。第二个子组（参见2.7.2）研究了更复杂的同步类型：基于四个函数`MPI_Win_start`、`MPI_Win_complete`、`MPI_Win_post`和`MPI_Win_wait`的本地同步（MPI7Win18–MPI7Win23），以及基于`MPI_Win_lock`和`MPI_Win_unlock`的锁同步（MPI7Win24–MPI7Win27, MPI7Win29）。在任务MPI7Win28和MPI7Win30中，需要同时使用第二个子组中讨论的两种同步方式。

![[Pasted image 20231201135156.png]]
```C
#include "pt4.h"

  

#include "mpi.h"

  

void Solve()

{

    Task("MPI7Win2");

    int flag;

    MPI_Initialized(&flag);

    if (flag == 0)

        return;

    int rank, size;

    MPI_Comm_size(MPI_COMM_WORLD, &size);

    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

  

    int capacity = size*(size-1)/2;

    int key = rank * (rank-1)/2;

  

    ShowLine(capacity);

    double* save = new double[rank]; // Sources

  

    double* mem_win = new double[capacity]; //Target

    for(int i=0;i<capacity;i++){

        mem_win[i]=0;

    }

    MPI_Win win;

    MPI_Win_create(mem_win,capacity* sizeof(double) , sizeof(double),MPI_INFO_NULL,MPI_COMM_WORLD,&win);

  

    if (rank != 0){

        //Init the value in the slave process

        for(int i =0;i<rank;i++){

            pt >>save[i];

            //ShowLine(save[i]);

        }

  

    }

    //synchrornization of all process

    MPI_Win_fence(0, win);

  

    //I want to put the data into the 0 from all slave processes

    if(rank!=0)//Tips : disp== rank !!!!!!! It is vital.

        MPI_Put(save, rank, MPI_DOUBLE, 0,  key, rank, MPI_DOUBLE, win);

        //key : 是每个slave process 往win of targe_process 写入东西时的偏移量

    //synchrornization of all process

    MPI_Win_fence(0, win);

  

    if (rank == 0) {

        for (int i = 0 ; i < capacity ; i++)

            pt << mem_win[i];

    }

     MPI_Win_free(&win);

}
```


当使用 MPI 的 `MPI_Put` 函数时，你可以用以下示例演示如何将数据从一个进程放置到另一个进程的数组中：

在这个示例中，进程0初始化了一个数组`sourceArray`，包含了1到5。然后，进程0使用 `MPI_Put` 将 `sourceArray` 的数据放置到 `targetArray` 中。接下来，所有进程通过 `MPI_Get` 读取了 `targetArray` 的内容，并输出了结果
```C
#include <stdio.h>
#include <mpi.h>

#define ARRAY_SIZE 5

int main(int argc, char** argv) {
    MPI_Init(&argc, &argv);

    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    int sourceArray[ARRAY_SIZE] = {0};
    int targetArray[ARRAY_SIZE] = {0};
    MPI_Win win;

    // 创建窗口
    MPI_Win_create(targetArray, ARRAY_SIZE * sizeof(int), sizeof(int), MPI_INFO_NULL, MPI_COMM_WORLD, &win);

    // 在进程0中初始化源数组
    if (rank == 0) {
        for (int i = 0; i < ARRAY_SIZE; ++i) {
            sourceArray[i] = i + 1;
        }
        printf("Process %d initialized source array\n", rank);
    }

    // 同步所有进程
    MPI_Win_fence(0, win);

    // 进程0将数组数据放置到进程1的目标数组中
    if (rank == 0) {
        MPI_Win_lock(MPI_LOCK_SHARED, 1, 0, win);
        MPI_Put(sourceArray, ARRAY_SIZE, MPI_INT, 1, 0, ARRAY_SIZE, MPI_INT, win);
        MPI_Win_unlock(1, win);
        printf("Process %d put data into target array\n", rank);
    }

    // 同步所有进程
    MPI_Win_fence(0, win);

    // 所有进程读取目标数组的内容
    MPI_Win_lock(MPI_LOCK_SHARED, rank, 0, win);
    MPI_Get(targetArray, ARRAY_SIZE, MPI_INT, rank, 0, ARRAY_SIZE, MPI_INT, win);
    MPI_Win_unlock(rank, win);

    // 输出目标数组内容
    printf("Process %d received target array: ", rank);
    for (int i = 0; i < ARRAY_SIZE; ++i) {
        printf("%d ", targetArray[i]);
    }
    printf("\n");

    MPI_Win_free(&win);
    MPI_Finalize();
    return 0;
}

```