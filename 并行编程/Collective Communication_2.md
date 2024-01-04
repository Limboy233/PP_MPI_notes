==MPI Scatter, Gather, and Allgather==

讲述两个额外的机制来补充集体通信的知识 - `MPI_Scatter` 以及 `MPI_Gather`。我们还会讲一个 `MPI_Gather` 的变体：`MPI_Allgather`。

# MPI_Scatter
MPI_Scatter 是一个和 MPI_Bcast 类似的集体通讯机制:
`MPI_Scatter` 的操作会设计一个指定的根进程，根进程会将数据发送到 communicator 里面的所有进程.
![[Pasted image 20230916103926.png]]
==重点区别==:
`MPI_Bcast` 给每个进程发送的是_同样_的数据，然而 `MPI_Scatter` 给每个进程发送的是_一个数组的一部分数据_。

在图中我们可以看到,
MPI_Bcast在跟进程上接收一个单独的元素(红色的方块),然后把他们复制到所有的进程.(=>信息拷贝同步)

MPI_Scatter 接收一个数组 , **并把元素按照==进程的秩==分发出去**: 第一个元素(红色方块发往进程0),第二个元素(绿色方块发往进程1),....... (=> 实现计算图计算分离)

重要:(==特殊==)------> 即使自己是根进程还是会发给自己的自己应该得到的元素.!!!!!!-->比如上文:
尽管根进程（进程0）拥有整个数组的所有元素，`MPI_Scatter` 还是会把正确的属于进程0的元素放到这个进程的接收缓存中.

## MPI_Scatter() 函数原型

```C
MPI_Scatter(
	void* send_data,
	int send_count,
	MPI_Datatype send_datatype,
	void* recv_data,
	int recv_count,
	MPI_Datatype recv_datatype,
	int root,
	MPI_Comm communicator)
```

1. void* send_data : 是在根进程上的一个数据数组
2.  int send_count : 描述了发送给每个进程的数据数量
3. MPI_Datatype send_datatype : 发送给每个进程的数据类型
==> (send_count * send_datatype = 发送给每个进程的数据量)
e.g.如果 `send_count` 是1，`send_datatype` 是 `MPI_INT`的话，进程0会得到数据里的第一个整数，以此类推。
如果`send_count`是2的话，`send_datatype` 是 `MPI_INT`的话,进程0会得到前两个整数，进程1会得到第三个和第四个整数，以此类推。
==在实践中，一般来说`send_count`会等于数组的长度除以进程的数量==
	**如果除不尽** ,-----> 后面会讲 :)))))

函数定义里面接收数据的参数跟发送的参数几乎相同: 
4. 4 5 6 :  `recv_data` 参数是一个==缓存==，它里面存了`recv_count`个`recv_datatype`数据类型的元素。

7. int root : 指定开始分发数组的根进程
8. MPI_Comm communicator : 以及对应的communicator。

===>(==接受和发送都是调用同一个API(参数除了接收缓存可以不同其他要基本一致!!!!!!)==)

# MPI_Gather

==是MPI_Scatter的逆操作==
: 从很多个进程中收集数据到一个进程上面而不是从一个进程分发数据到多个进程.
**多用于: 平行算法 e.g. 并行排序与搜索**
![[Pasted image 20230917175202.png]]
跟`MPI_Scatter`类似，`MPI_Gather`从其他进程收集元素到根进程上面,元素是根据接收到的进程的秩排序的。
`MPI_Gather`的函数原型跟`MPI_Scatter`长的一样。
```C
MPI_Gather(
    void* send_data,
    int send_count,
    MPI_Datatype send_datatype,
    void* recv_data,
    int recv_count,
    MPI_Datatype recv_datatype,
    int root,
    MPI_Comm communicator)
```
根进程 就是那个收集的
在`MPI_Gather`中，只有根进程需要一个有效的接收缓存。

**所有其他的调用进程可以传递`NULL`给`recv_data`**
==别忘记_recv_count_参数是从_每个进程_接收到的数据数量，而不是所有进程的数据总量之和==

# 应用: 计算平均数
提供了一个用来计算数组里面所有数字的平均数的样例程序（[avg.c](https://github.com/mpitutorial/mpitutorial/tree/gh-pages/tutorials/mpi-scatter-gather-and-allgather/code/avg.c)）。尽管这个程序十分简单，但是它展示了我们如何使用MPI来把工作拆分到不同的进程上，每个进程对一部分数据进行计算，然后再把每个部分计算出来的结果汇集成最终的答案。
Step:
1. 在根进程（进程0）上生成一个充满随机数字的数组。
2. 把所有数字用`MPI_Scatter`分发给每个进程，每个进程得到的同样多的数字。
3. 每个进程计算它们各自得到的数字的平均数。
4. 根进程收集所有的平均数，然后计算这个平均数的平均数，得出最后结果。

```C
#include <stdio.h>
#include <mpi.h>
#include <stdlib.h>
#include <time.h>
#include <assert.h>
float* create_rand_nums(int num_element) {

    float* rand_nums = (float*) malloc(sizeof(float)* num_element);

    assert(rand_nums != NULL);

    int i;

    for (i=0;i<num_element;i++){

        rand_nums[i] = (rand() /(float)RAND_MAX);

    }

    return rand_nums;

}
float compute_avg (float* array , int num_element){

    float sum =0.f;

    int i;

    for (i=0;i<num_element;i++){

        sum += array[i];

    }

    return sum / num_element;

}
int main (int argc,char**argv){

    if (argc !=2){
        fprintf(stderr, "Usage : avg num_element_per_proc\n");
        exit(1);
    }
    int num_elements_per_proc = atoi(argv[1]);
    srand(time(NULL));
    
    MPI_Init(&argc,&argv);
    
    int world_rank;
    MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);
    int world_size;
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);

    float* rand_nums =NULL;
    //根进程 0 中 建立那个数组
    if (world_rank == 0) {
    rand_nums = create_rand_nums(num_elements_per_proc * world_size);
    }
    //括号不用扩到下面,因为根进程也会要接收到自己的那一份!!!!!!


    float *sub_rand_nums = (float *)malloc(sizeof(float) * num_elements_per_proc);
    assert(sub_rand_nums != NULL);

  

  // Scatter the random numbers from the root process to all processes in

  // the MPI world

    MPI_Scatter(rand_nums, num_elements_per_proc, MPI_FLOAT, sub_rand_nums,
             num_elements_per_proc, MPI_FLOAT, 0, MPI_COMM_WORLD);
  // Compute the average of your subset
    float sub_avg = compute_avg(sub_rand_nums, num_elements_per_proc);
  // Gather all partial averages down to the root process
 
    float *sub_avgs = NULL;//其他进程调用的时候 send_data ---> 为NULL 即可!!!!
    // Gather 的根进程要分配好,收集所有进程信息的空间
    if (world_rank == 0) {

        sub_avgs = (float *)malloc(sizeof(float) * world_size);
        assert(sub_avgs != NULL);
    }

    //其他进程调用的时候 send_data ---> 为NULL 即可!!!!

    MPI_Gather(&sub_avg, 1, MPI_FLOAT, sub_avgs, 1, MPI_FLOAT, 0, MPI_COMM_WORLD);

  // Now that we have all of the partial averages on the root, compute the

  // total average of all numbers. Since we are assuming each process computed

  // an average across an equal amount of elements, this computation will

  // produce the correct answer.

 //用Gather的根进程报告结果

  if (world_rank == 0) {
    float avg = compute_avg(sub_avgs, world_size);
    printf("Avg of all elements is %f\n", avg);
    // Compute the average across the original data for comparison
    float original_data_avg =
      compute_avg(rand_nums, num_elements_per_proc * world_size);
    printf("Avg computed across original data is %f\n", original_data_avg);
  }

  // Clean up
  if (world_rank == 0) {
    free(rand_nums);
    free(sub_avgs);
  }
  free(sub_rand_nums);
  MPI_Barrier(MPI_COMM_WORLD);

  MPI_Finalize();
}
```
代码开头根进程创建里一个随机数的数组。当`MPI_Scatter`被调用的时候，每个进程现在都持有`elements_per_proc`个原始数据里面的元素。每个进程计算子数组的平均数，然后根进程收集这些平均数。然后总的平均数就可以在这个小的多的平均数数组里面被计算出来。

```OUTPUT
mpirun -n 14 ./avg 100
Avg of all elements is 0.491956
Avg computed across original data is 0.491955
```

# MPI_Allgather 
讲解了两个用来操作**多对一**或者**一对多**通信模式的MPI方法，也就是说多个进程要么向一个进程发送数据，要么从一个进程接收数据
------> 现在我们讲 **多对多**---->==MPI_Allgather==

对于分发在所有进程上的一组数据来说，`MPI_Allgather`会收集所有数据到所有进程上。

==从最基础的角度来看，`MPI_Allgather`相当于一个`MPI_Gather`操作之后跟着一个`MPI_Bcast`操作。==

下面的示意图显示了`MPI_Allgather`调用之后数据是如何分布的。
![[Pasted image 20230917184929.png]]

就跟`MPI_Gather`一样，每个进程上的元素是==根据他们的秩为顺序被收集起来的==，只不过这次是**收集到了所有进程上面**。

!!!**这里的recv_count 是从每个进程中接收的数目,不是从全部进程中一起接收的数目**

`MPI_Allgather`的方法定义跟`MPI_Gather`几乎一样，==只不过`MPI_Allgather`不需要root这个参数来指定根节点==。
```C
MPI_Allgather(
    void* send_data,
    int send_count,
    MPI_Datatype send_datatype,
    void* recv_data,
    int recv_count,
    MPI_Datatype recv_datatype,
    MPI_Comm communicator)
```

## 用 MPI_Allgather 稍微更改一下 
![[Pasted image 20230917185853.png]]

此处不需要单独只给 Gather 的根进程单独开辟空间,都一起开,一起算
```C
float* sub_avgs = (float*) malloc(sizeof(float) * eorld_size);
MPI_Allgather(&sub_avg, 1, MPI_FLOAT, sub_avgs, 1 MPI_FLOAT,MPI_COMM_WORLD);

float avg = compute_avg(sub_avgs, world_size);
printf("```
Avg of all elements from proc %d is %d\n",world_rank,avg);   
```

```OUTPUT
mpirun -n 4 ./all_avg 100
Avg of all elements from proc 1 is 0.479736
Avg of all elements from proc 3 is 0.479736
Avg of all elements from proc 0 is 0.479736
Avg of all elements from proc 2 is 0.479736
```
现在每个子平均数被`MPI_Allgather`收集到了所有进程上面。最终平均数在每个进程上面都打印出来了。样例运行之后应该跟下面的输出结果类似

跟你注意到的一样，all_avg.c 和 avg.c 之间的唯一的区别就是 all_avg.c 使用`MPI_Allgather`把平均数在每个进程上都打印出来了.

