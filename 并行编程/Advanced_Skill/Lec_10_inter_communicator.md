这段内容描述了MPI-1标准中引入的互通子（inter-communicators）。与“普通”通信子（也称为内部通信子）不同，后者连接到一组进程并支持这些组中任何进程之间的各种交互，

互通子连接到两组进程，旨在支持来自不同组的进程之间的交互；这两组进程在其中的等级被使用。


互通子的交互模式非常方便，如果并行算法需要在多个进程组之间分发操作，并需要在这些不同组中的进程之间交换信息。

在MPI-2标准中，互通子的概念得到了进一步的发展：创建互通子的功能得到了扩展，允许在互通子内进行集体进程交互，并且最终，互通子成为了动态进程创建机制的基础工具。


每个互通子与两个平等的进程组相关联。

来自任一组的进程都可以在互通子中与来自另一组的进程交换信息。
1. 调用发送或接收消息的进程所属的组称为本地组（local group），
2. 而包含要建立联系的进程的组称为远程组（remote group）。


因此，**对于发送进程，接收进程所在的远程组是本地组，而对于接收进程，发送进程所在的远程组是本地组。**

==发送进程的目标进程（即远程组中的进程）的rank是指在远程组中的进程rank。==

组间通信子（inter-communicator）

组间通信与组内通信是两个相对的概念，参与组内通信的进程，都属于相同的进程组，并在相同的组内通信子对象上下文环境中执行。相应地，组间通信子把两个组绑定在一起，共享通信上下文，用于管理两者之间的通信。组间通信常用于解决采取模块化结构设计的，在多个空间中运行的复杂应用中的通信问题。

一个组内的进程要与另一个组的进程进行通信就要指定目标组和目标进程在目标组内的 rank 两个信息。

MPI 的进程总是属于某些进程组。我们称发起通信的进程所属进程组为本地组（local group），而称包含某次通信发起者所指定目标进程的组为远程组（remote group）。所谓远程组和本地组只是在一次通信过程中形成的相对的、临时的概念。

对点到点通信，通信双方所需指定的消息“信封”仍是（通信子，rank，tag），

==与组内通信不同的是，组间通信的 rank 总是远程组里的 rank。==
![[Pasted image 20231202012131.png]]
# 通信子与互通子两用函数
标准的 `MPI_Comm_size(MPI_Comm comm, int* size)` 函数用于确定通信子 `comm` 中的进程数量 `size`，它也可以用于互通子。在这种情况下，它返回本地组中进程的数量，即调用该函数的进程所在的互通子组。

`MPI_Comm_rank(MPI_Comm comm, int* rank)` 函数用于互通子，返回本地组中进程的等级 `rank`。

对于互通子 `comm`，`MPI_Comm_group(MPI_Comm comm, MPI_Group* group)` 函数返回其本地组 `group`，而 `MPI_Comm_remote_group` 函数也提供了类似的参数集以获取远程组。
# 单独互通子的函数
还有一个附加函数 `MPI_Comm_remote_size(MPI_Comm comm, int* size)`，仅适用于互通子；它返回远程组的大小 `size`，即不调用该函数的进程所在的互通子组。

# 检查互通子的函数 
使用 `MPI_Comm_test_inter(MPI_Comm comm, int *flag)` 函数可以检查通信子 `comm` 是否为互通子。如果是互通子，则此函数返回非零值，如果是内部通信子，则返回零


[更多例子](https://www.mcs.anl.gov/research/projects/mpi/mpi-standard/mpi-report-1.1/node114.htm)
例子:
![[Pasted image 20231201224942.png]]
```C
main(int argc, char **argv) 
   { 
     MPI_Comm   myComm;       /* intra-communicator of local sub-group */ 
     MPI_Comm   myFirstComm;  /* inter-communicator */ 
     MPI_Comm   mySecondComm; /* second inter-communicator (group 1 only) */ 
     int membershipKey; 
     int rank; 
 
     MPI_Init(&argc, &argv); 
     MPI_Comm_rank(MPI_COMM_WORLD, &rank); 
 
     /* User code must generate membershipKey in the range [0, 1, 2] */ 
     membershipKey = rank % 3; 
 
     /* Build intra-communicator for local sub-group */ 
     MPI_Comm_split(MPI_COMM_WORLD, membershipKey, rank, &myComm); 
 
     /* Build inter-communicators.  Tags are hard-coded. */ 
     if (membershipKey == 0) 
     {                     /* Group 0 communicates with group 1. */ 
       MPI_Intercomm_create( myComm, 0, MPI_COMM_WORLD, 1, 
                            1, &myFirstComm); 
     } 
     else if (membershipKey == 1) 
     {              /* Group 1 communicates with groups 0 and 2. */ 
       MPI_Intercomm_create( myComm, 0, MPI_COMM_WORLD, 0, 
                            1, &myFirstComm); 
       MPI_Intercomm_create( myComm, 0, MPI_COMM_WORLD, 2, 
                            12, &mySecondComm); 
     } 
     else if (membershipKey == 2) 
     {                     /* Group 2 communicates with group 1. */ 
       MPI_Intercomm_create( myComm, 0, MPI_COMM_WORLD, 1, 
                            12, &myFirstComm); 
     } 
 
     /* Do work ... */ 
 
     switch(membershipKey)  /* free communicators appropriately */ 
     { 
     case 1: 
        MPI_Comm_free(&mySecondComm); 
     case 0: 
     case 2: 
        MPI_Comm_free(&myFirstComm); 
        break; 
     } 
 
     MPI_Finalize(); 
   }
```

MPI_Intercomm_create函数用于创建一个互通子（inter-communicator），需要用到以下几个参数：

- MPI_Comm local：与将要创建的互通子相关联的本地组的通信子。
- int local_leader：本地组的代表等级。在参数local指定的通信子中，指定这个进程的等级。
- MPI_Comm peer_comm：中介通信子。这个参数只在充当本地组代表的进程中使用。
- int remote_leader：远程组的代表等级。在中介通信子peer_comm中指定这个进程的等级，同样也只在充当本地组代表的进程中使用。
- int tag：安全标签，要在调用MPI_Intercomm_create函数的所有进程中保持一致。当创建其他互通子时，应使用不同的标签值。
- MPI_Comm* intercomm：指向创建的互通子的指针（输出参数）。
- ![[Pasted image 20231202000954.png]]

文本提供了一种方法来确定local_leader和remote_leader参数的值。对于这两个参数，可以根据进程在MPI_COMM_WORLD通信子中的等级来确定。比如，如果进程在MPI_COMM_WORLD通信子中的等级是0（代表它是第一组的代表），则将参数remote_leader设为K/2；如果进程在MPI_COMM_WORLD通信子中的等级是K/2，则将参数remote_leader设为0。对于其他进程，可以选择任意值，例如也设为0。

该文本还提到了一种用C的值来确定remote_leader参数值的方法。对于C=1的进程，可以将remote_leader参数设为K/2；对于C=2的进程，可以将remote_leader参数设为0。

为了确保正确创建所需的互通子，可以进行简单的检查：调用MPI_Comm_remote_size函数来确定每个进程的互通子的远程组大小，并在调试部分输出这个值。这个大小在后续数据传输时可能会用到。

此外，这段文本还提到在解决问题的第二阶段时，如果尝试在一些进程退出Solve函数后创建MPI_COMM_WORLD的一个副本，会导致并行应用程序死锁。因此，在条件操作符中之前必须确定中介通信子

这个简单的最后阶段需要使用 MPI_Send 和 MPI_Recv 函数在两个进程之间交换消息。这些函数需要用于一个已经创建的互通子（inter-communicator）。在这个阶段，==你需要用 MPI_Send 来指定接收进程在远程组中的等级，而用 MPI_Recv 来指定发送进程在远程组中的等级。==

# MPI_Comm_spawn
![[Pasted image 20231202144146.png]]
使用模板:
```C
MPI_Comm inter; 
MPI_Comm_get_parent(&inter); 
if (inter == MPI_COMM_NULL) { 
	MPI_Comm_spawn("ptprj.exe", NULL, 1, MPI_INFO_NULL, 0, MPI_COMM_WORLD, &inter, MPI_ERRCODES_IGNORE); 
} 
Show(size); 
Show(rank);
```

==这个建立的子进程就是与原来的父进程形成 Intercommunicator  ==
所以在进行集体操作的时候要 注意 数字 ,MPI_ROOT,MPI_PRIC_NULL 这三个的使用
# 在 MPI_Comm_spawn 之前一般要使用MPI_Comm_get_parent 检查,本进程是子进程了,还是需要建立新的子进程
这段描述主要涉及在执行函数`Solve`时，需要区分当前代码运行的进程是原始进程还是新创建的子进程。为了实现这一区分，可以在 `Solve` 函数的开头==调用 `MPI_Comm_get_parent` 函数，该函数会返回当前进程的父通信器==。

如果返回值是 `MPI_COMM_NULL`，那么当前进程是原始进程，需要调用 `MPI_Comm_spawn` 函数生成新的子进程。

如果返回的不是 `MPI_COMM_NULL`，说明当前进程是子进程，可以使用该通信器与父进程进行通信。


这段描述的主要目的是确保在运行 `Solve` 函数时，只有原始进程调用了 `MPI_Comm_spawn` 函数来生成新的子进程，而子进程直接使用已存在的通信器与父进程通信。最后，为了验证新进程的创建是否成功，建议在调试部分输出每个并行应用程序进程的 `size` 和 `rank` 值。

例题 
![[Pasted image 20231202151016.png]]
```C
double a, sum; 
int root; 
MPI_Comm inter; 
MPI_Comm_get_parent(&inter); 
if (inter == MPI_COMM_NULL) { 
MPI_Comm_spawn("ptprj.exe", NULL, 1, MPI_INFO_NULL, 0, MPI_COMM_WORLD, &inter, MPI_ERRCODES_IGNORE); 
pt >> a; 
root = 0; } 
else root = MPI_ROOT; 

MPI_Reduce(&a, &sum, 1, MPI_DOUBLE, MPI_SUM, root, inter); 
MPI_Bcast(&sum, 1, MPI_DOUBLE, root, inter); 
if (root == MPI_ROOT) Show(sum); else pt << sum;
```
这段代码首先使用 `MPI_Comm_get_parent` 检查是否存在父进程，如果没有，则通过 `MPI_Comm_spawn` 生成一个新的进程组并创建了一个 intercommunicator。如果是父进程，它从输入中读取一个 double 类型的值 `a`，然后指定 `root = 0`。在这个例子中，新进程被用作根进程来接收归约操作的结果，所以设定 `root = MPI_ROOT`。

接着，使用 `MPI_Reduce` 在 intercommunicator 上进行归约操作，将各个进程的 `a` 值求和，并将结果存储在根进程的 `sum` 变量中。

最后，如果进程是根进程，即 `root == MPI_ROOT`，则使用 `Show(sum)` 输出归约后的结果。


# MPI 中的客户端和服务器模式
在MPI中使用 `MPI_Intercomm_merge` 和客户端-服务器端通信的机制来处理两个不同组的进程之间的通信问题。
1) 首先介绍了 `MPI_Intercomm_merge` 函数，该函数用于将两个不同组的进程合并成一个单一的内部通信组（intracommunicator），通过参数 `high` 来确定合并后的进程顺序。

2) 接下来描述了客户端-服务器端通信的机制。在这种情况下，一个组的进程充当服务器角色，另一个组的进程充当客户端角色。服务器端创建一个端口并发布其公共名称，然后等待客户端连接。客户端通过公共名称查找服务器端的端口，并通过该端口连接到服务器端。

最后介绍了释放相关资源的函数，包括 `MPI_Close_port` 用于释放端口、`MPI_Unpublish_name` 用于释放公共名称、以及 `MPI_Comm_disconnect` 用于销毁客户端和服务器端之间的通信。

MPI_Intercomm_merge是MPI中的一个特殊函数，用于合并两个不同组的进程成为一个单一的内部通信子。它有三个参数：原始的互通子comm、high参数用于确定在创建的新互通子中进程的顺序、以及输出参数newcomm，指向新创建的互通子。MPI_Intercomm_merge必须在原始互通子的所有进程中调用。high参数是一个整数标志；原始互通子中每个组的所有进程必须指定相同的high值。如果在一个组中high的值为0，在另一个组中为1，那么在创建的互通子中首先排列来自第一个组的进程（high=0），然后是来自第二个组的进程（high=1），它们在各自组中的顺序保持不变。如果在原始互通子的所有进程中high参数的值相同，那么组的排列顺序是不确定的，但是每个组内进程的顺序与原始组中的顺序相同。

有可能出现两个进程组之间没有共同的互通子的情况（例如，如果每个组都是使用单独的MPI_Comm_spawn调用创建的）。对于这样的进程组之间建立连接，可以使用MPI-2中实现的一种特殊的客户端-服务器交互机制。其中一个进程组扮演服务器的角色，另一个扮演客户端的角色。服务器组的进程创建用于通信的端口（使用MPI_Open_port函数），并定义此端口的公共名称（使用MPI_Publish_name函数），随后开始侦听此端口，以等待客户端连接（使用MPI_Comm_accept函数）。这个函数是集体的，必须对comm中的所有进程调用，但是端口名port_name只需要在根进程中指定

这段描述了在MPI中如何使用客户端-服务器通信模式进行进程间的连接。在这种情况下，一个进程组扮演服务器角色，另一个进程组扮演客户端角色。

客户端进程首先通过使用`MPI_Lookup_name`函数，利用服务器进程组提供的公共名称`service_name`来获取服务器创建的端口信息，并将其存储在`port_name`中。`MPI_Lookup_name`函数是集体操作，需要对comm中的所有进程进行调用。然后，客户端进程使用`MPI_Comm_connect`函数连接到服务器的端口，传入端口名称`port_name`，这也是一个集体操作，需要对comm中的所有进程进行调用，而端口名称`port_name`只需要在rank为root的进程中指定。

如果`MPI_Comm_accept`和`MPI_Comm_connect`成功执行，它们将返回一个新的互通子`newcomm`，用于连接客户端和服务器进程组。端口`port_name`是一个文本字符串，在调用`MPI_Open_port`函数时由MPI环境生成，其最大长度不超过常量`MPI_MAX_PORT_NAME`。端口的创建只需在服务器进程组的一个进程中进行，而在客户端进程组的一个进程中获取即可。与`port_name`不同，公共端口名称`service_name`必须提前为服务器和客户端进程组的进程所知，它类似于两组之间交换的“密码”。

在所有这些函数中都存在一个额外的参数`info`，可以简单地将其设为`MPI_INFO_NULL`。

在服务器端的标准操作序列如下所示：
```C
char port[MPI_MAX_PORT_NAME];
MPI_Open_port(MPI_INFO_NULL, port);
MPI_Publish_name("password", MPI_INFO_NULL, port);
MPI_Comm_accept(port, MPI_INFO_NULL, root, comm, &inter);

```
在客户端的操作序列如下所示:
```C
char port[MPI_MAX_PORT_NAME];
MPI_Lookup_name("password", MPI_INFO_NULL, port);
MPI_Comm_connect(port, MPI_INFO_NULL, root, comm, &inter);

```
这段描述了在客户端-服务器模式下的一些重要概念和操作。以下是相关内容的翻译：

"重要的是协调函数调用的顺序，确保在服务器组的根进程中，`MPI_Publish_name`函数在时间上早于客户端组的根进程中的`MPI_Lookup_name`函数。为释放客户端-服务器连接所分配的资源，有三个函数：
- `MPI_Close_port(char* port_name)`：释放由`MPI_Open_port`分配的端口`port_name`。
- `MPI_Unpublish_name(char* service_name, MPI_Info info, char* port_name)`：释放之前与端口`port_name`相关联的公共名称`service_name`（在释放端口`port_name`之前必须先释放公共名称）。
- `MPI_Comm_disconnect(MPI_Comm* comm)`：销毁用于连接客户端和服务器的通信器`comm`，并将其设置为`MPI_COMM_NULL`。与标准的`MPI_Comm_free`不同，`MPI_Comm_disconnect`函数会等待使用通信器`comm`执行的所有通信操作完成后才会销毁该通信器。

有关客户端-服务器交互机制的任务可以在MPI8Inter21至MPI8Inter22中找到。这些任务的注释中描述了如何使用`MPI_Barrier`函数来协调`MPI_Publish_name`和`MPI_Lookup_name`函数的调用顺序的不同方法。"

# 第2题
![[Pasted image 20231202022432.png]]
我们通过这道题彻底搞懂intercommuicator
他是组间通信--> 用i
nt MPIAPI MPI_Intercomm_create(
        MPI_Comm local_comm,
        int      local_leader,
        MPI_Comm peer_comm,
        int      remote_leader,
        int      tag,
  _Out_ MPI_Comm *newintercomm
);
建立
local_comm ---> 为发起通信者所在的Comm
local_leader----> 一般是零  (这个是local_comm的头头---> 那当然是零了)
==重点==
peer_comm----> 他是一个包含着我们要链接的另一个Communacator的Communicator
---->一般我们用 MPI_COMM_WORLD
remote_leader--->是我们要链接的那个Communicator 的头头(就是在这个Communicator中rank/=/=0 的进程)在peer_comm中的秩
tag---> 过去过来两个对应即可(看上面那个例子)
newintercomm  ----> OUTOUT : intercomm

## 在intercommunicator 中的Recv 和Send
```C
 if(C==0){

            MPI_Send(&X, 1, MPI_DOUBLE, R, 0, inter);

            MPI_Recv(&Save, 1, MPI_DOUBLE, R, 0, inter, &s);

        }else{

            MPI_Recv(&Save, 1, MPI_DOUBLE, R, 0, inter, &s);

             MPI_Send(&X, 1, MPI_DOUBLE, R, 0, inter);

        }
```

==他的秩是 远程COmm 的秩==:
我们: ![[Pasted image 20231202023817.png]]

会发现:
![[Pasted image 20231202023841.png]]
remote_size ==== 5  , 因为两个通信子互为 remote 
所以这个Send和Recv 的秩其实就是另一个通讯子的秩
(TipS: Send  ----> 是发到另一组通讯子那个秩上  ;Recv------> 是从另一组通讯子的那个秩上接收   )

而本题
本质上是在 交换对应秩的元素 : 就是 R---->new_rank (因为我们在构造另一个通信子的时候已经逆序了)
![[Pasted image 20231202024531.png]]
又因为我们是用C 作为Color 来划分的 , 所以我们要进行上述处理,让C/=/=0的通讯子先SEND ,C/=/=1的先接收(==这样就不会出现死锁==),
然后两个操作的秩均为 R , 因为 本题
本质上是在 交换对应秩的元素 : 就是 R---->new_rank (因为我们在构造另一个通信子的时候已经逆序了)

```C
#include "pt4.h"
#include "mpi.h"

void Solve()

{

    Task("MPI8Inter2");

    int flag;

    MPI_Initialized(&flag);

    if (flag == 0)

        return;

    int rank, size;

    MPI_Comm_size(MPI_COMM_WORLD, &size);

    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

  

    MPI_Comm peer;

    MPI_Comm_dup(MPI_COMM_WORLD, &peer);

    MPI_Comm local;

    int C;pt >> C;

    double X; pt >> X;

    if(C==0){

        MPI_Comm_split(MPI_COMM_WORLD, C, rank, &local);

    }

    if(C==1){

        MPI_Comm_split(MPI_COMM_WORLD,C,size-rank-1,&local);

    }

    if (local == MPI_COMM_NULL)

    { pt << -1; return; }

    int R;//new_rank

    MPI_Comm_rank(local,&R);

    pt << R;

    MPI_Comm inter;

    int lead=0;

    if(C==0)

        lead=size-1;

    MPI_Intercomm_create(local, 0, peer, lead, 1, &inter);

    MPI_Status s;

    double Save;

  

    int remote_size;

    MPI_Comm_remote_size(inter, &remote_size);

    Show(remote_size);

        if(C==0){

            MPI_Send(&X, 1, MPI_DOUBLE, R, 0, inter);

            MPI_Recv(&Save, 1, MPI_DOUBLE, R, 0, inter, &s);

        }else{

            MPI_Recv(&Save, 1, MPI_DOUBLE, R, 0, inter, &s);

             MPI_Send(&X, 1, MPI_DOUBLE, R, 0, inter);

        }

    pt << Save;
  }
```

# Intercommunicator 的集体通信
![[Pasted image 20231202123532.png]]
他是把左边的X 汇集到 右边 等于 R2 的新秩上的process 上去

Comm1 (C=1) 与 Comm(C=2) : 我们建立inter通讯子

然后使用集体函数 MPI_Gather 来汇集
1. intercommunicator 中的每个process 都必须调用 MPI_Gather
2. (==重点==) --> 这个 root_rank----> 是有特别标识符的 
	1)  MPI_PROC_NULL
	2) MPI_ROOT
====> 我们要在发送组写的是发送到的那个接收组的那个进程的rank (==因为在intercommmunicator 中所有rank 都是指 remot 的rank==)
那个收集的只有我们发送给他的那个进程需要接收(MPI_ROOT),其他都是MPI_PORC_NULL

2. 其实还可以再细一点: GROUP(C=1)可以不需要接收,接收那一块写 NULL,0,MPI_REC_NULL
, GROUP(C=2)可以不要发送,发送那一块写 , NULL,0,MPI_ESNF_NULL, 而且只要一个特定的根进程接收,所以在C=2时,if (R2 != R)

            R2 = MPI_PROC_NULL;

        else

            R2 = MPI_ROOT;
 把秩R(本秩)!=R2(要接收进程的秩)---->均设为 MPI_PROC_NULL
 把秩R(本秩)/=/=R2 -------> 设为 MPI_ROOT(为唯一接收根进程)!!!!!!!!
 (==为什么要这么设置?==)
 **因为 Intercommunicator 只能看见 对面那个组的秩--->我们要让他知道自己是接收汇集的中心,所以要用这两个特别的标识符!!!!!!!!!!!!!!!!!!!!!!!!1**

```C
pt >> R2 //
if(C!=0){

        int remote_size;

        MPI_Comm_remote_size(inter,&remote_size);

        int save[remote_size]={0};

        if (C == 2)

    {

        if (R2 != R)

            R2 = MPI_PROC_NULL;

        else

            R2 = MPI_ROOT;

    }

  

    MPI_Gather(&X, 1, MPI_INT, save, 1, MPI_INT, R2, inter);

    if (R2 == MPI_ROOT)

        for (int i = 0; i < remote_size; i++)

            pt << save[i];

  

    }
```

Whole Code:
```C
#include "pt4.h"

  

#include "mpi.h"

  

void Solve()

{

    Task("MPI8Inter12");

    int flag;

    MPI_Initialized(&flag);

    if (flag == 0)

        return;

    int rank, size;

    MPI_Comm_size(MPI_COMM_WORLD, &size);

    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

    MPI_Comm Comm,peer,inter;

    MPI_Comm_dup(MPI_COMM_WORLD,&peer);

    int C;pt>>C;

    int R=-1;

    int R2,X;

    //Like Task 3

    //0   1 <-----> 2

    //0 & 1 do not have the intercommunicator

    if(C==0){

        MPI_Comm_split(MPI_COMM_WORLD, C, rank, &Comm);

        pt << R;

    }else if(C==1){

            pt >> R2 >> X;

            MPI_Comm_split(MPI_COMM_WORLD, C, rank, &Comm);

            MPI_Comm_rank(Comm,&R);pt <<R;

            MPI_Intercomm_create(Comm,0,peer,size/2,66,&inter);

  

  

        }else if(C==2){

                pt >> R2;

                MPI_Comm_split(MPI_COMM_WORLD, C, rank, &Comm);

                MPI_Comm_rank(Comm,&R);pt <<R;

                MPI_Intercomm_create(Comm,0,peer,0,66,&inter);

            }

  

    if(C!=0){

        int remote_size;

        MPI_Comm_remote_size(inter,&remote_size);

        int save[remote_size]={0};

        if (C == 2)

    {

        if (R2 != R)

            R2 = MPI_PROC_NULL;

        else

            R2 = MPI_ROOT;

    }

  

    MPI_Gather(&X, 1, MPI_INT, save, 1, MPI_INT, R2, inter);

    if (R2 == MPI_ROOT)

        for (int i = 0; i < remote_size; i++)

            pt << save[i];

  

    }

  

}
```

![[Pasted image 20231202142923.png]]
```C
#include "pt4.h"

  

#include "mpi.h"

  

void Solve()

{

    Task("MPI8Inter13");

    int flag;

    MPI_Initialized(&flag);

    if (flag == 0)

        return;

    int rank, size;

    MPI_Comm_size(MPI_COMM_WORLD, &size);

    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

  

    MPI_Comm peer,Comm,inter;

    MPI_Comm_dup(MPI_COMM_WORLD,&peer);

    int C;pt >> C;int R=-1;

    double Val,Save;

  

    if(C==0){

        pt << R;

        MPI_Comm_split(MPI_COMM_WORLD, C, rank, &Comm);

    }else if(C==1){

            pt >> Val;

            MPI_Comm_split(MPI_COMM_WORLD, C, rank, &Comm);

            MPI_Comm_rank(Comm,&R);pt <<R;

            MPI_Intercomm_create(Comm,0,peer,size/2,66,&inter);

  

  

        }else if(C==2){

                pt >> Val;

                MPI_Comm_split(MPI_COMM_WORLD, C, rank, &Comm);

                MPI_Comm_rank(Comm,&R);pt <<R;

                MPI_Intercomm_create(Comm,0,peer,0,66,&inter);

            }

    int num =1;

    if (C == 1)

        //我们约归的是对方的 MAX 因为Intercommunicator看见的是对方的rank

        MPI_Allreduce(&Val, &Save, num, MPI_DOUBLE, MPI_MAX, inter);

    else

    {

        MPI_Allreduce(&Val, &Save, num, MPI_DOUBLE, MPI_MIN, inter);

    }

    if(C!=0)

        pt << Save;

  

}
```

# 另一个集体操作的例子
这段代码的主要目标是使用 MPI_Reduce 在 intercommunicator 上执行归约操作。在这个例子中，`MPI_Comm_spawn` 用于生成新的进程组，并创建一个 intercommunicator。
```C
double a, sum;
int root;
MPI_Comm inter;
MPI_Comm_get_parent(&inter);

if (inter == MPI_COMM_NULL) {
    MPI_Comm_spawn("ptprj.exe", NULL, 1, MPI_INFO_NULL, 0, MPI_COMM_WORLD, &inter, MPI_ERRCODES_IGNORE);
    pt >> a;
    root = 0;
} else {
    root = MPI_ROOT;
}

MPI_Reduce(&a, &sum, 1, MPI_DOUBLE, MPI_SUM, root, inter);

if (root == MPI_ROOT) {
    Show(sum); // 在新进程中输出归约后的结果
}

```

这段代码首先使用 `MPI_Comm_get_parent` 检查是否存在父进程，如果没有，则通过 `MPI_Comm_spawn` 生成一个新的进程组并创建了一个 intercommunicator。

!!!!!==Tips==如果是父进程，它从输入中读取一个 double 类型的值 `a`，然后指定 `root = 0`。在这个例子中，新进程被用作根进程来接收归约操作的结果，所以设定 `root = MPI_ROOT`。

接着，使用 `MPI_Reduce` 在 intercommunicator 上进行归约操作，将各个进程的 `a` 值求和，并将结果存储在根进程的 `sum` 变量中。

最后，如果进程是根进程，即 `root == MPI_ROOT`，则使用 `Show(sum)` 输出归约后的结果