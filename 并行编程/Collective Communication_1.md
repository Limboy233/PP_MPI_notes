我们之前讲了,blocking point-to-point communication,这种通讯只会涉及两个不同的线程.
而现在的Collective Communication (MPI _集体通信_) 指的是一个通信行为涉及 communicator 里面所有进程的一个方法。


# 集体通信概念 与 同步点

## synchronization point
这意味着所有的进程在执行代码的时候必须首先_都_到达一个同步点才能继续执行后面的代码。
==也即先完成同步点前代码的进程将被阻塞,等到所有进程都到同步点后再进行同步点下方代码==

## 为了同步而进行的阻塞 MPI_Barrier()

```C
MPI_Barrier(MPI_Comm communicator)
```

（Barrier，屏障）- 这个方法会构建一个屏障，任何该通讯器(Commuincator)进程都没法跨越屏障，直到所有的进程都到达屏障,才可进行下一步.

![[Pasted image 20230915162511.png]]

`MPI_Barrier` 在很多时候很有用。其中一个用途是用来同步一个程序，使得分布式代码中的某一部分可以被精确的计时

# 那MPI_Barrier()是如何实现的呢?
我们之前 在 [[Blocking point-to-pointCommunicator]]中用MPI_Recv() 和MPI_Send() 实现的 ==ring.c== 程序就是它的一个简单实现 :**我们当时写了一个在所有进程里以环的形式传递一个令牌（token）的程序,这种形式的程序是最简单的一种实现屏障的方式，==因为令牌只有在所有程序都完成之后才能被传递回第一个进程==**

关于同步最后一个要注意的地方是：**始终记得每一个你调用的集体通信方法都是同步的**

# ==集体方法均是会同步的==
如果你没法让所有进程都完成 `MPI_Barrier`，那么你也没法完成任何集体调用。如果你在没有确保所有进程都调用 `MPI_Barrier` 的情况下调用了它，那么程序会空闲下来。
# broadcasting (广播)(MPI_Bcast())

_广播_ (broadcast) 是标准的==集体通信技术==之一
==> Broadcast 肯定是**会同步**的
一个广播发生的时候，一个进程会把同样一份数据传递给一个 communicator 里的所有其他进程
## 广播的用途
1. 用户输入传递给一个分布式程序
2. 把一些配置参数传递给所有的进程。


![[Pasted image 20230915163442.png]]
==进程0==是我们的**根**进程，它持有一开始的数据。其他所有的进程都会从它这里接受到一份数据的副本
![[Pasted image 20230915163904.png]]

```C
MPI_Bcast(
    void* data,
    int count,
    MPI_Datatype datatype,
    int root,
    MPI_Comm communicator)
```

**尽管根节点和接收节点在广播这个操作中做的事情是不一样的,但他们调用的函数及该函数的参数都是一样的(唯一的不同就是执行该函数的地方(进程)不同)**   :

**当根节点**(在我们的例子是节点0)调用 `MPI_Bcast` 函数的时候，==`data` 变量里的值会被发送到其他的节点上。==

**当其他的节点**调用 `MPI_Bcast` 的时候，==`data` 变量会被赋值成从根节点接受到的数据。==

---->所以变量data的赋值应该在进程0中(声明在外面),这样其他进程也调用他时就可以也把接收到的值存在data中不会浪费空间 ( **当然我们也可以重新申请空间,不存在data变量中 :)))))))**)
# Broadcast使用例子
```C
#include<mpi.h>
#include<stdio.h>

int main(int argc,char**argv){
    MPI_Init(&argc,&argv);

    int data;
    int world_rank ,world_size;
    MPI_Comm_rank(MPI_COMM_WORLD,&world_rank);
    MPI_Comm_size(MPI_COMM_WORLD,&world_size);

    //我们想让 进程0 成为广播发起者(根进程)
    if(world_rank ==0){
        data = 666;
        printf("Process 0 broadcasting data %d\n", data);
        MPI_Bcast(&data,1,MPI_INT,0,MPI_COMM_WORLD);
    }else{
        MPI_Bcast(&data,1,MPI_INT,0,MPI_COMM_WORLD);
        printf("Process %d get the data %d from 0\n",world_rank,data);
    }
    MPI_Finalize();
}
```

```OUTPUT
mpirun -n 14 ./bcast
Process 0 broadcasting data 666
Process 8 get the data2 666 from 0
Process 9 get the data2 666 from 0
Process 10 get the data2 666 from 0
Process 2 get the data2 666 from 0
Process 3 get the data2 666 from 0
Process 4 get the data2 666 from 0
Process 5 get the data2 666 from 0
Process 11 get the data2 666 from 0
Process 6 get the data2 666 from 0
Process 1 get the data2 666 from 0
Process 12 get the data2 666 from 0
Process 13 get the data2 666 from 0
Process 7 get the data2 666 from 0
```
# 用MPI_Send()与MPI_Recv() 实现Broadcast

我们可以用Blocking point-to-point 来实现Brodcast,因为似乎 `MPI_Bcast` 仅仅是在 `MPI_Send` 和 `MPI_Recv` 基础上进行了一层包装.但是==直接包装的话函数效率非常低==
因为Broadcast是通过树型广播实现的,`MPI_Bcast` 的实现使用了一个类似的树形广播算法来获得比较好的网络利用率.
![[Pasted image 20230916100729.png]]
实现这个算法有点超出我们这个课的主要目的了，如果你觉得你足够勇敢的话，可以去看这本超酷的书：[Parallel Programming with MPI](http://www.amazon.com/gp/product/1558603395/ref=as_li_qf_sp_asin_tl?ie=UTF8&tag=softengiintet-20&linkCode=as2&camp=217145&creative=399377&creativeASIN=1558603395)

我们的比较愚蠢的实现:(基于阻塞点对点通讯的)
```C
void my_bcast(void* data, int count, MPI_Datatype datatype, int root,
			 MPI_Comm communicator){
	int world_rank;
	MPI_Comm_rank(communicator,&world_rank);
	int world_size;
	MPI_Comm_size(communicator,&world_size);

	if (world_rank == root){
		//我们是根节点我们要发送信息给所有其他节点
		int i;
		for(i=0;i< world_size;i++){
			if(i != world_rank){
				MPI_Send(data, count, datatype, i, 0, communicator);
			}
		}
	}else{
		//其他节点要接收其数据
		MPI_Recv(data, count, datatype, root, 0,communicator, MPI_STATUS_IGNORE);
	}
}
```
==> 在调用我们的my_bcast()时,下方要加一个同步,来避免根进程发送完了就不管其他人继续啊执行其他东东去了:))))

## 两者比较:

先让我们看一个 MPI 跟时间相关的函数 - `MPI_Wtime`。`MPI_Wtime` 不接收参数，它仅仅返回以浮点数形式展示的从1970-01-01到现在为止进过的秒数，跟 C 语言的 `time` 函数类似。我们可以多次调用 `MPI_Wtime` 函数，并去差值，来计算我们的代码运行的时间。
```C
#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>
#include <assert.h>

void my_bcast(void* data, int count, MPI_Datatype datatype, int root,
              MPI_Comm communicator) {
  int world_rank;
  MPI_Comm_rank(communicator, &world_rank);
  int world_size;
  MPI_Comm_size(communicator, &world_size);

  if (world_rank == root) {
    // If we are the root process, send our data to everyone
    int i;
    for (i = 0; i < world_size; i++) {
      if (i != world_rank) {
        MPI_Send(data, count, datatype, i, 0, communicator);
      }
    }
  } else {
    // If we are a receiver process, receive the data from the root
    MPI_Recv(data, count, datatype, root, 0, communicator, MPI_STATUS_IGNORE);
  }
}

int main(int argc, char** argv) {
  if (argc != 3) {
    fprintf(stderr, "Usage: compare_bcast num_elements num_trials\n");
    exit(1);
  }

  int num_elements = atoi(argv[1]);
  int num_trials = atoi(argv[2]);

  MPI_Init(NULL, NULL);

  int world_rank;
  MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);

  double total_my_bcast_time = 0.0;
  double total_mpi_bcast_time = 0.0;
  int i;
  int* data = (int*)malloc(sizeof(int) * num_elements);
  assert(data != NULL);

  for (i = 0; i < num_trials; i++) {
    // Time my_bcast
    // Synchronize before starting timing
    MPI_Barrier(MPI_COMM_WORLD);
    total_my_bcast_time -= MPI_Wtime();
    my_bcast(data, num_elements, MPI_INT, 0, MPI_COMM_WORLD);
    // Synchronize again before obtaining final time
    MPI_Barrier(MPI_COMM_WORLD);
    total_my_bcast_time += MPI_Wtime();

    // Time MPI_Bcast
    MPI_Barrier(MPI_COMM_WORLD);
    total_mpi_bcast_time -= MPI_Wtime();
    MPI_Bcast(data, num_elements, MPI_INT, 0, MPI_COMM_WORLD);
    MPI_Barrier(MPI_COMM_WORLD);
    total_mpi_bcast_time += MPI_Wtime();
  }

  // Print off timing information
  if (world_rank == 0) {
    printf("Data size = %d, Trials = %d\n", num_elements * (int)sizeof(int),
           num_trials);
    printf("Avg my_bcast time = %lf\n", total_my_bcast_time / num_trials);
    printf("Avg MPI_Bcast time = %lf\n", total_mpi_bcast_time / num_trials);
  }

  free(data);
  MPI_Finalize();
}
```

```OUTPUT
mpirun -n 16 -machinefile hosts ./compare_bcast 100000 10
Data size = 400000, Trials = 10
Avg my_bcast time = 0.510873
Avg MPI_Bcast time = 0.12683
```
![[Pasted image 20230916102424.png]]