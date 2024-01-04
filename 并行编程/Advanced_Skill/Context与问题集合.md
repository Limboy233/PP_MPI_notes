1. 看视频 : [并行程序设计 谭光明](https://www.bilibili.com/video/BV1XM4y1S7wy/?spm_id_from=333.337.search-card.all.click&vd_source=92fc72eea66d0c63275fc8606897f5df)
中的MPI 部分----> 来补充MPI技术 和 openmp

2. 老师上课会讲一些API,和一些其他高级通信技巧!!!!!

3.看MPI 文档!!!!!   [[mpi40-report.pdf]] 

[MPI 微软文档](https://learn.microsoft.com/zh-cn/message-passing-interface/microsoft-mpi)

下方会列出一些上课学习的技术!!!!!

# 1  MPI_Alltoallv(题目)
1. (MPI_Alltoallv): 每个进程都有k(k+1)/2 个数 , 所有进程的第 1 个数发给0号进程 , ==接着==的两个数发给1号进程以此类推,.....
(Sends i items to processor i from all processors.)
重点: Allroallv 的使用

**int MPI_Alltoallv(**
  **void** *_sendbuf_**,**
  **int** *_sendcnts_**,**
  **int** *_sdispls_**,**
  **MPI_Datatype** _sendtype_**,**
  **void** *_recvbuf_**,**
  **int** *_recvcnts_**,**
  **int** *_rdispls_**,**
  **MPI_Datatype** _recvtype_**,**
  **MPI_Comm** _comm_
**);**

1.**参数数组的初始化**:------> ==是这道题的重点== 
int* sendcounts 和int *recvcounts。这两个参数你可以把它当做两个数组，数组中的元素代表往其他节点各发送（接收）多少数据。比如说，sendcounts[0]=3,sendcounts[1]=4,代表该节点要往0号节点发送3个sendtype的数据，往1号节点发送4个sendtype的数据。  

多了两个参数，int\*sdispls和int \*rdispls,这两个可以看做是数组，数组中的每个元素代表了要发送（接收）的那块数据相对于缓冲区起始位置的位移量。
------> **sdispls 和 rdispls 第一个 元素一般是 0 (一般都是从发送缓冲区第一个(第0号)元素开始发送的)**

(==重点: 思考的时候要以本进程来思考!!!!!!!==)

!!! 除了发送和接收存储点(缓冲区),其他可以不用动态内存分配来建立数组

```C
	int mem_size = (size*(size+1))/2;
	int* mem = (int*) malloc(sizeof(int)*mem_size);
	int* save_buffer = (int*)malloc(sizeof(int)*size*(rank+1));
	
	int sendcounts[size];
    int recvcounts[size];
    int rdispls[size];
    int sdispls[size];
    if (!sendcounts || !recvcounts || !rdispls || !sdispls) {
        fprintf( stderr, "Could not allocate arg items!\n" );fflush(stderr);
        MPI_Abort(MPI_COMM_WORLD, 1 );
    }
    for (int i=0; i<size; i++) {
        sendcounts[i] = i+1;
        recvcounts[i] = rank+1;//与i无关,与运行它的进程有关
        rdispls[i] = i*(rank+1);
        sdispls[i] = (i * (i+1))/2;
    }
    MPI_Alltoallv(mem,sendcounts,sdispls,MPI_INT,save_buffer,recvcounts,rdispls,MPI_INT,MPI_COMM_WORLD);
```

# 2 缓冲区辨析
2. MPI_Bsend 和 MPI_Recv 要求我们用MPI_Buffer_attach &  MPI_Buffer_detach 设置的缓冲区,和函数参数中的 **const void* buffer** 是==两个概念==,不是一个东西!!! 
		1) MPI_Bsend 和 MPI_Recv 中的 **const void* buffer** 是真实的发送(接收)数据存储的位置
		2) MPI_Buffer_attach &  MPI_Buffer_detach 设置的缓冲区 : 是为了执行 **缓冲模式发送** MPI_Bsend : 让==**发送程序**== 把要发送的东西 冲入  MPI_Buffer_attach &  MPI_Buffer_detach 设置的缓冲区即可往下继续执行程序代码,不用等待接收线程接收到了才能继续运行.
		3) MPI_Buffer_attach &  MPI_Buffer_detach 设置的缓冲区  一定要 ==尽量大==
	```C 
	int buff_size = 1000;
	int* buffer = (int*)malloc(sizeof(int)*buff_size);//缓冲区设置得尽量足够大!!!!
	MPI_Buffer_attach(buffer,buff_size);
	MPI_Buffer_detach(buffer,&buff_size);
	free(buffer);
    ```
		
一定不要看着数据数量精确设置 !!!!!!!!!

# 3 MPI_ANY_SOURCE 使用(!!!)
3. MPI_ANY_SOURCE 
是 MPI（Message Passing Interface）中的一个特殊常量，用于接收消息时指定可以接受来自任何发送者的消息。**它的主要用途是在接收操作中不关心消息的发送者，从而实现更灵活的消息接收**。
以下是 `MPI_ANY_SOURCE` 的一些用途和示例：

1. **灵活接收**: `MPI_ANY_SOURCE` 允许接收进程从任何其他进程接收消息。这在不需要确定特定发送者的情况下非常有用。==而且我还可以知道发送者的信息==: 通过status!!!!!!!!!!!
```C    
int source; // 用于存储实际消息的发送者
MPI_Recv(receive_buffer, count, datatype, MPI_ANY_SOURCE, tag, comm, &status); 
source = status.MPI_SOURCE; // 获取实际消息的发送者`
```
这使得接收进程能够接受来自任何进程的消息，而不仅仅是一个特定的进程。
==而且我还可以知道发送者的信息==
2. **动态任务分配**: 当任务数在运行时动态分配给不同的进程时，`MPI_ANY_SOURCE` 允许接收进程处理来自任何发送者的消息，而不需要事先知道哪些进程会发送消息。
    
3. **广播**: 在广播操作中，广播源使用 `MPI_Bcast` 向所有其他进程广播消息，而其他进程可以使用 `MPI_ANY_SOURCE` 接收来自广播源的消息。  
    ```C
    MPI_Bcast(buffer, count, datatype, broadcast_root, comm); 
    if (my_rank != broadcast_root) {     
	    MPI_Recv(buffer, count, datatype, MPI_ANY_SOURCE, tag, comm, &status); 
    }
    ```

    这允许广播源向所有其他进程广播相同的消息，而其他进程可以使用 `MPI_ANY_SOURCE` 接收广播消息。

`MPI_ANY_SOURCE` 的使用使得编写更通用和灵活的 MPI 程序成为可能，因为接收进程不需要提前知道消息的发送者是谁。这对于动态系统和任务分配非常有用，因为任务可以在运行时动态地创建和销毁

# 4 死锁
如果一直卡在:
![[Pasted image 20231017211637.png]]
==是有八九就是死锁了!!!!!!!!!!==
4.MPI（Message Passing Interface）死锁

是在并行计算中常见的问题，通常是由于不正确的进程间通信方式导致的。MPI死锁通常发生在以下情况下：

1. **相互等待**: 死锁通常是由于进程之间相互等待对方释放资源或完成通信而导致的。这可能发生在两个或多个进程之间的相互依赖通信上。
    
2. **缺少匹配的发送和接收操作**: 如果发送操作的数量不匹配接收操作的数量，可能会导致死锁。例如，如果一个进程尝试接收更多的消息，而没有足够的发送操作来匹配，死锁可能会发生。
    
3. **循环依赖**: 如果一组进程形成循环依赖，每个进程等待下一个进程释放资源，这可能导致死锁。
    
4. **消息排队问题**: MPI消息在队列中等待发送或接收。如果这些队列满了，可能会导致死锁。
    

避免MPI死锁的一些常见方法包括：

1. **正确匹配发送和接收操作**: 确保每个发送操作都有对应的接收操作，以便消息能够被正确地传递。
    
2. **不要过度使用阻塞操作**: 尽量避免使用阻塞操作，特别是在复杂的通信模式中。非阻塞通信和异步通信操作可以减少死锁的风险。
    
3. **使用MPI_Waitall或MPI_Testall**: 这些函数允许您在一次调用中等待多个通信请求，确保它们都已完成。
    
4. **避免循环依赖**: 尽量设计算法，以避免进程之间的循环依赖。
    
5. **使用适当的同步**: 使用MPI_Barrier等同步机制，以确保所有进程都已完成其工作，然后再继续。
    
6. **使用调试工具**: 使用MPI调试工具（如MPICH或Open MPI的调试选项）来检测和解决潜在的死锁问题。

下面就是一个典型的**死锁**例子: ==**相互等待**== 
![[Pasted image 20231017201540.png]]
```C
int rank, size;
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    int token;pt >> token;
    int save;
    MPI_Ssend(&token,1,MPI_INT,(rank+size-1)%size,0,MPI_COMM_WORLD);
    MPI_Recv(&save,1,MPI_INT,(rank+1)%size,0,MPI_COMM_WORLD,MPI_STATUS_IGNORE);
    pt << save;
```
我们想让每个进程向上传值:
按找我们上述代码,会导致发送完的MPI_Ssend线程等待接收线程完成接收,
==但是他们又不可能完成接收==,因为MPI_Ssend 无法完成,大家都没有人可以执行到下一步的MPI_Recv操作,以至于大家都卡在了MPI_Ssend,产生了死锁.

!!!!!   ==(rank+size-1)%size== ---> 逆向!!!!

解决方法: 1) 使用缓冲模式: MPI_Bsend

2)使用我们在MPI_tutorial中写过的 ring程序的例子,打破这个僵局
```C
 if(rank != 0){
        MPI_Recv(&save,1,MPI_INT,(rank+1)%size,0,MPI_COMM_WORLD,MPI_STATUS_IGNORE);
        pt << save;
    }else{}

    MPI_Ssend(&token,1,MPI_INT,(rank+size-1)%size,0,MPI_COMM_WORLD);

    if(rank==0){
        MPI_Recv(&save,1,MPI_INT,(rank+1)%size,0,MPI_COMM_WORLD,MPI_STATUS_IGNORE);
        pt << save;
    }
```
这个程序由rank=\=0  打头阵 , 先让所有rank 不是 0 的程序等待接收 ,MPI_Ssend 此时只有 rank=\=0 可以发出-----> 然后依次打破死局, 最后 rank=\=0 再接收到它的!!!!

下方是等价程序----> 但上方少写一个 MPI_Ssend
```C
if(rank==0){
        MPI_Ssend(&token,1,MPI_INT,(rank+size-1)%size,0,MPI_COMM_WORLD);
        MPI_Recv(&save,1,MPI_INT,(rank+1)%size,0,MPI_COMM_WORLD,MPI_STATUS_IGNORE);
        pt <<save;

    }else{
        MPI_Recv(&save,1,MPI_INT,(rank+1)%size,0,MPI_COMM_WORLD,MPI_STATUS_IGNORE);
        MPI_Ssend(&token,1,MPI_INT,(rank+size-1)%size,0,MPI_COMM_WORLD);
        pt << save;

    }
```

# MPI_Sendrecv_replace
==replace 表示原地API== : 即使用同一块区域来发送与接收, 即接收到覆盖掉发送区域
**`MPI_Sendrecv_replace` 是一个 MPI 函数，用于同时发送和接收数据，并用接收到的数据覆盖发送缓冲区**。
这是一个==组合操作==，它允许进程通过发送数据并接收其他进程的数据来实现互通。
```C
int MPI_Sendrecv_replace(void *buf, int count, MPI_Datatype datatype, int dest, int sendtag, int source, int recvtag, MPI_Comm comm, MPI_Status *status)
```

`MPI_Sendrecv_replace` 是一个**非阻塞**操作，它执行以下操作：

1. 进程将数据从自己的缓冲区 `buf` 发送到目标进程 `dest`。
2. 同时，它接收来自源进程 `source` 的数据，并将其覆盖到相同的缓冲区 `buf` 中。
3. 在发送和接收数据后，函数会返回有关接收操作的状态信息（如果指定了 `status` 参数）。

这个函数在某些特定的通信模式中非常有用，其中进程需要交换数据。它简化了这种操作，因为它可以确保在交换数据时不会出现数据冗余。

请注意，使用 `MPI_Sendrecv_replace` 时需要小心，确保发送和接收操作之间没有死锁。通常，这是在进程之间协调好的，以确保互相等待对方的数据。
![[Pasted image 20231026012851.png]]

MPI_Cart_sub 的 remind 数组要多调整这样才能实现我们在笛卡尔拓扑中取到正确子集实现 环传输token
==而且在构建笛卡尔拓扑中的 periodic 要开启 不然MPI_Cart_Shift 得到的 source和destion 没有办法成环, 那你也没有办法完成这个任务==
**periods={1,1,1}**---> 三个维度环周期访问都开启!!!!
```C
#include "pt4.h"

  

#include "mpi.h"

  

void Solve()

{

    Task("MPI5Comm27");

    int flag;

    MPI_Initialized(&flag);

    if (flag == 0)

        return;

    int rank, size;

    MPI_Comm_size(MPI_COMM_WORLD, &size);

    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

  

    double num;pt >> num;

  

    MPI_Comm newcomm;

    int dims[3]={2,2,size/4};

    int periods[3]={1,1,1};

    MPI_Cart_create(MPI_COMM_WORLD,3,dims,periods,0,&newcomm);

  

    MPI_Comm sub_Cart;

    int remind[3]={0,1,1};

    MPI_Cart_sub(newcomm,remind,&sub_Cart);

  

    MPI_Comm sub_sub_Cart;

    int rem[2]={0,1};

    MPI_Cart_sub(sub_Cart,rem,&sub_sub_Cart);

  
  

    int source, dest;//dest --> 下一个

    // 查找当前进程在x方向上的左侧和右侧相邻进程的rank-----> 0 : x方向

    MPI_Cart_shift(sub_sub_Cart, 0, 1, &source, &dest);

  

    int new_rank;

    MPI_Comm_rank(sub_sub_Cart,&new_rank);

    Show(new_rank);

  

    MPI_Sendrecv_replace(&num, 1,MPI_DOUBLE,dest,0,source,0,sub_sub_Cart,MPI_STATUSES_IGNORE);

    pt << num;

  

}
```
# tag 这个参数的使用场景!!!

可以让我们**不用数组** 就 完成顺序接收!!!!!!!!!
---------------->MPI_ANY_SOURCE + tag参数

![[Pasted image 20231018152846.png]]
![[Pasted image 20231018155600.png]]
# 自定义数据类型中有blank , blank在数据类型开头或者结尾叫 Hole

其实我们定义一个类型,其实-->他有用法是:
==是在定义一个新的 发送一次数据的大小==

**这也是为什么我们要往里面加 blank---> 其实blank本无用,但是其如此填充让新数据的大小变化有用**!!!!
!!!!
!例子:
```C
// 在主进程中给出了两个整数序列：大小为3K的序列A和大小为K的序列N，其中K是从进程的数量。
    // 序列的元素从1开始编号。将序列A的N_R个元素发送到每个从进程R（R＝1，2，…，K），从A_R开始并将序号增加2（R，R+2，R+4，…）。
    // 例如，如果N2等于3，则过程2应当接收元素A2、A4、A6。输出每个从进程中接收到的所有数据。
    // 使用MPI_Send、MPI_Probe和MPI_Recv函数的一个调用向每个从属进程发送号码；MPI_Recv函数应该返回一个数组，该数组只包含应该输出的元素。
    // 要做到这一点，请定义一个新的数据类型，该数据类型包含一个整数和一个大小等于整数数据类型大小的额外空格（一个HOLE）。
    // 使用以下数据作为MPI_Send函数的参数：具有适当位移的给定数组A、发送元素的数量N_R、新的数据类型。
    // 在MPI_Recv函数中使用大小为NR、数据类型为MPI_INT的整数数组。要确定接收元素的数量N_R，请在从属进程中使用MPI_Get_count函数。
    // NOTE:  使用MPI_Type_create_resize函数定义新数据类型的Hole大小（此函数应应用于MPI_INT数据类型）。
    // 在MPI-1中，零大小的上限标记MPI_UB应和MPI_Type_struct一起使用（在MPI-2中，不赞成使用MPI_UB伪数据类型）。
    if (rank == 0)
    {
        int k = size - 1;
        int *a = new int[3 * k];
        int *n = new int[k];
        for (int i = 0; i < 3 * k; ++i)
            pt >> a[i];
        for (int i = 0; i < k; ++i)
            pt >> n[i];
        //上面在接收
        MPI_Datatype t;//新的数据类型
        int int_sz;
        MPI_Type_size(MPI_INT, &int_sz);//给出数据类型的size大小 (size :  不包括空白((blank)))
        MPI_Type_create_resized(MPI_INT, 0, 2 * int_sz, &t);
        //搞了一个新的类型,他包含了oldetype:MPI_INT , 还是从 0 开始 ,但是 extent 变成了 2 * int_sz:SIZE_MPI_INT---> 即尾部有一个MPI_INT 大小的Hole
        MPI_Type_commit(&t);//提交 新类型 t----> 使这个类型真的变得可用
        //其实我们定义一个类型,其实是在定义一个新的 发送一次数据的大小
        // 序列的元素从1开始编号。将序列A的N_R个元素发送到每个从进程R（R＝1，2，…，K），从A_R开始并将序号增加2（R，R+2，R+4，…）。
        // 例如，如果N2等于3，则过程2应当接收元素A2、A4、A6。输出每个从进程中接收到的所有数
        for (int i = 1; i < size; ++i)
            MPI_Send(&a[i - 1], n[i - 1], t, i, 0, MPI_COMM_WORLD);
        delete[] a;
        delete[] n;
    }else{
        MPI_Status s;
        MPI_Probe(0, 0, MPI_COMM_WORLD, &s);
        int n;
        MPI_Get_count(&s, MPI_INT, &n);
        //目的是提前知道数据大小,先分配好接收区域 (动态消息传输)
        int *a = new int[n];
        MPI_Recv(a, n, MPI_INT, 0, 0, MPI_COMM_WORLD, &s);
        //你看,我传一个 2*MPI_INT ----> 但我读取的时候按题目要求只用读取第一个MPI_INT
        //这样就完成了题目
        //好处: 是不用手动 偏移 发送数组的地址
        for (int i = 0; i < n; ++i)
            pt << a[i];
        delete[] a;
    }
```
# 打包
![[Pasted image 20231022212008.png]]
三个关键点:
1) 正确的操作是通过两次打包 , 且让两次打包输出的位置在一块连续内存中
    // 第一个包是double + 第二个是后面的所有int
    // 都输出到这一块连续地址中====> (通过position参数)
  
2) 重点:  接收类型也是 MPI_PACKED ----> 然后我们再执行一步,解包操作

3) 注意: 要空掉第一个, 因为我们用的是 MPI_Gather 不是 MPI_Gatherv 0号进程也是会接收的,位置在第一个
        //如果用 MPI_Gatherv 我们可以设置---> 让0号进程接收0个元素



```C
	int rank, size;
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    int* get_arr = new int[size-1];
    //打包是为了避免定义类型,所以不应该定义类型
    //正确的操作是通过两次打包 , 且让两次打包输出的位置在一块连续内存中
    // 第一个包是double + 第二个是后面的所有int
    // 都输出到这一块连续地址中
    int buff_size = (size-1)*sizeof(int) + sizeof(double);
    char* send_buff = new char[buff_size];
    double R_number;

    if(rank > 0)
    {
        pt >> R_number;
        for(int i=0;i<rank ;++i){
            pt >> get_arr[i];
        }
        //输入输出参数 position 是我们能够执行这种操作前的关键
        int position=0;
        MPI_Pack(&R_number , 1,MPI_DOUBLE,send_buff,buff_size,&position,MPI_COMM_WORLD);

        MPI_Pack(get_arr,size-1,MPI_INT,send_buff,buff_size,&position,MPI_COMM_WORLD);
    }

    //给 0 号进程一个 大空间 , 但是因为我们要求要用集体通信所以我们不可以单独给0号进程开一个空间
    char* save_buff = new char[(size-1)*buff_size];MPI_Gather(send_buff,buff_size,MPI_PACKED,save_buff,buff_size,MPI_PACKED,0,MPI_COMM_WORLD);

    //重点:  接收类型也是 MPI_PACKED ----> 然后我们再执行一步,解包操作
    if(rank == 0)
    {
        //从头开始一个个解包
        //注意: 要空掉第一个, 因为我们用的是 MPI_Gather 不是 MPI_Gatherv 0号进程也是会接收的,位置在第一个
        //如果用 MPI_Gatherv 我们可以设置---> 让0号进程接收0个元素
        //还可以利用 之前分配给 0号进程的空间 : R_number , send_buff
        int pos = buff_size;
        for(int i=1;i<size;++i){
            MPI_Unpack(save_buff,buff_size,&pos,&R_number,1,MPI_DOUBLE,MPI_COMM_WORLD);           MPI_Unpack(save_buff,buff_size,&pos,get_arr,size-1,MPI_INT,MPI_COMM_WORLD);


            pt << R_number;  
            for(int j=0;j<i;++j){
               pt << get_arr[j];
            }
        }  
    }
```

# 构造接收数据结构(MPI4Type16)--> 不会
![[Pasted image 20231022223344.png]]
```C
int K =size-1;

  

    if(rank>0){

        double* send_arr = new double[K];

        for(int i=0;i<K;++i){

            pt >> send_arr[i];

        }

  

        for(int i=0;i<K;i++){

            MPI_Send(&send_arr[i],1,MPI_DOUBLE,0,0,MPI_COMM_WORLD);

            MPI_Barrier(MPI_COMM_WORLD);

        }

    }

  

    if(rank ==0){

  

        MPI_Datatype newtype;

        int doub_sz;

        MPI_Type_size(MPI_DOUBLE,&doub_sz);

        MPI_Type_contiguous(K,MPI_DOUBLE,&newtype);

        MPI_Type_commit(&newtype);

  

        double* Buff = new double[K*K];

  

        //因为一次接收要接收到满足 MPI_Recv 的数量 :  此时 K个MPI_Send 对应 1 个MPI_Recv

        for(int j=0;j<K;j++){

            for(int i=1;i<=K;i++)

            MPI_Recv(&Buff[j*K+(i-1)],1,newtype,i,0,MPI_COMM_WORLD,MPI_STATUS_IGNORE);

            MPI_Barrier(MPI_COMM_WORLD);

        }

        for(int i=0;i<K*K;i++){

            pt << Buff[i];

        }

    }
```

# MPI_Allgather(num_arr,2,MPI_DOUBLE,save,2,MPI_DOUBLE,newcomm);

MPI_Allgather ==的接收数目参数== : **是从每个进程中接收的数目,不是总共的数目(即不是接收缓冲区长度)**