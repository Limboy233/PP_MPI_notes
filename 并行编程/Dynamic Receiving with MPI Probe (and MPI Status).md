上一篇,我们使用 **MPI_Send** 和 **MPI_Recv** 执行标准的==Blocking Point-to-Point Communication==(**阻塞点对点通信**)

但是 我们那时候发送的是**接收者事先已知长度的消息**(==即约定消息长度的传输==)

==>现在我们学习如何Blocking Point-to-Point的进行==不约定消息长度(动态消息)的传输== (**接收者事先不知道长度的消息**)



我们先介绍**MPI\_Recv()** 的最后一个参数==MPI_Status* status
==,我们在上一节用*MPI_STATUS_IGNORE*忽略它了.

# MPI_Status 结构体中的主要信息

他的地址是MPI_Recv()函数的最后一个参数,如果我们将 **MPI_Status**结构体地址传递给 MPI_Recv() 函数，则操作完成后将在该结构体中填充有关接收操作的信息 :
## 主要信息有:
1. ==发送端秩(**MPI_SOURCE**)== :
 发送端的秩存储在结构体的 `MPI_SOURCE` 元素中。也就是说，如果我们声明一个 `MPI_Status stat` 变量，则可以通过 `stat.MPI_SOURCE` 访问秩
2. ==消息的标签(**MPI_TAG**)==:
消息的标签可以通过结构体的 `MPI_TAG` 元素访问（类似于 `MPI_SOURCE`）(即 stat.MPI_TAG)。

3. ==消息的长度== :**MPI_Status 中有它的信息,但是没有预定义的元素,故我们要使用另一个函数 MPI_Get_count 把消息从MPI_Status结构体中解析出来 **
### MPI_Get_count
```C
MPI_Get_count(
	MPI_Status* status,
	MPI_Datatype datatype,
	int* count
)
```
第一个参数:MPI_Status* status 传递的是MPI_status结构体的指针
第二个参数:MPI_Datatype datatype传递的是消息的数据类型
第三个参数:int* count是传递解析完MPI_Status* status中的关于消息长度的信息存放的地址
(即它指向的地方就存着 ==消息的长度==)

# 提出一个问题:
**我们MPI_Recv() 的参数中都包含了一些信息了,为什么我们还需要 MPI_Status 存下这三个信息?**
1. 事实证明，`MPI_Recv` 可以将 `MPI_ANY_SOURCE` 用作发送端的秩，将 `MPI_ANY_TAG` 用作消息的标签。 在这种情况下，`MPI_Status` 结构体是找出消息的实际发送端和标签的唯一方法。
2. 此外，并不能保证 `MPI_Recv` 能够接收函数调用参数的全部元素。 相反，它只接收已发送给它的元素数量（如果发送的元素多于所需的接收数量，则返回错误。） `MPI_Get_count` 函数用于确定实际的接收量。

# 应用: MPI_Status 结构体查询的示例
(还没到不约定消息长度(动态消息)的传输的时候)
==> 这个还是约定长度的,因为==**接收端缓存**和**接受数目(消息长度)**都是按可能出现的最大次数给的==

!!!!==重点 :==
不约定消息长度(动态消息)的传输,则应该**接收端缓存**和**接受数目(消息长度)** 都应该 **动态分配(依消息大小分配)**



程序将随机数量的数字发送给接收端，然后接收端找出发送了多少个数字:
```C
#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
int main(int argc,char**argv){
MPI_Init(NULL, NULL);

  int world_size;
  MPI_Comm_size(MPI_COMM_WORLD, &world_size);
  if (world_size != 2) {
    fprintf(stderr, "Must use two processes for this example\n");
    MPI_Abort(MPI_COMM_WORLD, 1);
  }
  int world_rank;
  MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);

const int MAX_NUMBER = 100
int numbers[MAX_NUMBERS];
int number_amount;//我们用它来存:
//进程2(发送进程): 随机发送数字的个数
//进程1(接收进程): MPI_Status 解析出来的消息长度

if (world_rank == 0){//进程 0
//Pick a random amount of integers to send to process 1
	srand(time(NULL));//初始化随机数构造器
	number_amount = (rand() / (float)RAND_MAX) * MAX_NUMBERS;//通过这个方法将随机数量的数字发送给接收端

	MPI_Send(numbers,number_amount,MPI_INT
			,1,0,MPI_COMM_WORL);
	printf("0 sent %d numbers to 1\n",number_amount);
}else if(world_rank == 1){
	MPI_Status status;
	// Receive at most MAX_NUMBERS from process 0
	MPI_Recv(numbers, MAX_NUMBERS, MPI_INT, 0, 
			0,MPI_COMM_WORLD,&status)			
	// After receiving the message, check the status to determine
    // how many numbers were actually received
    MPI_Get_count(&status, MPI_INT, &number_amount);
    
	// Print off the amount of numbers, and also print additional
    // information in the status object
    printf("1 received %d numbers from 0. Message source = %d, ""tag = %d\n",number_amount,status.MPI_SOURCE,status.MPI_TAG);
}
  MPI_Barrier(MPI_COMM_WORLD);
  MPI_Finalize();
}
```
==为什么不初始化numbers[MAX_NUMBERS]就发送? ==
__因为不初始化它里面就是随机值  :)__


如我们所见，进程 0 将最多 `MAX_NUMBERS` 个整数以随机数量发送到进程 1。 
进程 1 然后调用 `MPI_Recv` 以获取总计 `MAX_NUMBERS` 个整数。 
尽管进程 1 以 `MAX_NUMBERS` 作为 `MPI_Recv` 函数参数，但进程 1 将最多接收到此数量的数字(即**接收数目<=MAX_NUMBERS**)。 
在代码中，进程 1 使用 `MPI_INT` 作为数据类型的参数，调用 `MPI_Get_count`，以找出==实际==接收了多少个整数。
除了打印出接收到的消息的大小外，进程 1 还通过访问 status 结构体的 `MPI_SOURCE` 和 `MPI_TAG` 元素来打印消息的来源和标签。

==TIPS: MPI_Get_count 解析消息长度时,的数据类型要和接收到的类型一致==:
`MPI_Get_count` 的返回值是相对于传递的数据类型而言的。 如果用户使用 `MPI_CHAR` 作为数据类型，则返回的数量将是原来的四倍（假设整数是四个字节，而 char 是一个字节）

编译和运行: (-np 时number processes :))))
```shell
mpicc check_status.c -o check_status

mpirun -np 2 ./check_status
```

输出的例子:
```
0 sent 92 numbers to 1
1 received 92 numbers from 0. Message source = 0, tag = 0
```
正如预期的那样，进程 0 将随机数目的整数发送给进程 1，进程 1 将打印出接收到的消息的有关信息

下文,我们开始讲进行==不约定消息长度(动态消息)的传输== (**接收者事先不知道长度的消息**):

#  使用 `MPI_Probe` 找出消息大小

除了传递接收消息并简易地配备一个很大的缓冲区来为所有可能的大小的消息提供处理（就像我们在上一个示例中所做的那样）（==上一个例子的不聪明的方法==）

现在有MPI_Probe()我们有**聪明**的方法了：
您可以**使用 `MPI_Probe` 在实际接收消息之前查询消息大小**

```C
MPI_Probe(
    int source,
    int tag,
    MPI_Comm comm,
    MPI_Status* status)
```
	第一个参数 int source:发送端的rank--->因为发送端叫Source
	第二个参数 int tag : 数据标签
	第三个参数：MPI_Comm comm： communicator 类型
	最后一个参数 MPI_Status* status： 把打探到的信息放在 status 结构体里 :)))))))

MPI_Probe() 看起来与 MPI_Recv() 非常相似。 
1. 实际上，您可以将 `MPI_Probe` 视为 `MPI_Recv`，==除了不接收消息外，它们执行相同的功能==。
2. 与 `MPI_Recv` 类似，`MPI_Probe` 将==阻塞==具有匹配标签和发送端的消息。
3. **当消息可用时，它将填充 status 结构体**。 
----->由此我们可以通过这个MPI_Probe()函数来了解将要接收消息的信息(存在一个MPI_Status中)--->由此我们可以在接收时根据这个信息来知道将要接收的信息大小与生成合适大小的缓存空间---->实现==**动态消息传输**==

:)))))))))))
然后，用户可以使用 `MPI_Recv` 接收实际的消息。


# 动态消息传递代码 ：

和我们上一个例子类似，但我们用了更聪明的==动态消息传递==，而不是简易地配备一个很大的缓冲区来为所有可能的大小的消息提供处理   :((((

==重点区别是： 我们在接收线程1 中采用了MPI_Probe() 来在实际接收消息之前查询消息信息，从而得到消息大小，分配适当大小的缓冲区并接收数字（而不是简易地配备一个很大的缓冲区来为所有可能的大小的消息提供处理:))))))) ）==

```C
...}
else if(world_rank == 1){
	MPI_Status status;
	//Probe for an incoming message from process 0
	MPI_Probe(0, 0, MPI_COMM_WORLD, &status);

	MPI_Get_count(&status, MPI_INIT, &number_amount);
	//MPI_Get_count 把从MPI_Probe得到的MPI_Staus结构体中的信息长度解析出来了

	//通过MPI_Get_count解析出的信息长度，来构型接收缓存
	int* number_buf=(int*)malloc(sizeof(int) * number_amount);

	MPI_Recv(number_buf, number_amount, MPI_INT,0,0,MPI_COMM_WORLD, MPI_STATUS_IGNORE);
	//MPI_STATUS_IGNORE: 这里就不用再接收一边MPI_Status了，前面我们已经用MPI_Probe() 接收一次啦
printf("1 dynamically received %d numbers from 0.\n",
           number_amount);
    free(number_buf);//别忘了free掉
}
```
与上一个示例类似，进程 0 选择随机数量的数字发送给进程 1。
==不同之处在于==，进程 1 现在调用 `MPI_Probe`，以找出进程 0 试图发送多少个元素（利用 `MPI_Get_count`）。 然后，进程 1 分配适当大小的缓冲区并接收数字。

完整程序：
```C
#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

int main(int argc, char** argv) {
  MPI_Init(NULL, NULL);

  int world_size;
  MPI_Comm_size(MPI_COMM_WORLD, &world_size);
  if (world_size != 2) {
    fprintf(stderr, "Must use two processes for this example\n");
    MPI_Abort(MPI_COMM_WORLD, 1);
  }
  int world_rank;
  MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);

  int number_amount;
  if (world_rank == 0) {
    const int MAX_NUMBERS = 100;
    int numbers[MAX_NUMBERS];
    // Pick a random amont of integers to send to process one
    srand(time(NULL));
    number_amount = (rand() / (float)RAND_MAX) * MAX_NUMBERS;
    // Send the amount of integers to process one
    MPI_Send(numbers, number_amount, MPI_INT, 1, 0, MPI_COMM_WORLD);
    printf("0 sent %d numbers to 1\n", number_amount);
  } else if (world_rank == 1) {
    MPI_Status status;
    // Probe for an incoming message from process zero
    MPI_Probe(0, 0, MPI_COMM_WORLD, &status);
    // When probe returns, the status object has the size and other
    // attributes of the incoming message. Get the size of the message.
    MPI_Get_count(&status, MPI_INT, &number_amount);
    // Allocate a buffer just big enough to hold the incoming numbers
    int* number_buf = (int*)malloc(sizeof(int) * number_amount);
    // Now receive the message with the allocated buffer
    MPI_Recv(number_buf, number_amount, MPI_INT, 0, 0, MPI_COMM_WORLD,
             MPI_STATUS_IGNORE);
    printf("1 dynamically received %d numbers from 0.\n",
           number_amount);
    free(number_buf);
  }
  MPI_Finalize();
}
```
尽管这个例子很简单，**但是 `MPI_Probe` 构成了许多动态 MPI 应用程序的基础**。 

e.g.  控制端/执行子程序在==交换变量大小的消息==时通常会大量使用 `MPI_Probe`。 

作为练习，对 `MPI_Recv` 进行包装，将 `MPI_Probe` 用于您可能编写的任何动态应用程序。 它将使代码看起来更美好：-)