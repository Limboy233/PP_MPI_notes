先介绍归约的概念:-----> 我们用python讲一下 :)
# 归约(Reduce)
_归约_ 是函数式编程中的经典概念。
**数据归约包括通过函数将一组数字归约为较小的一组数字。**
例如，假设我们有一个数字列表 `[1,2,3,4,5]`。
1. 用 sum 函数归约此数字列表将产生 `sum([1、2、3、4、5]) = 15`。 类似地，
2. 乘法归约将产生 `multiply([1、2、3、4、5]) = 120`。

## Python
在python2中，reduce 是内置函数，但是在 Python 3 中放到 functools 模块里了。
```Python
from functools import reduce
from operator import add

list_num = [1,2,3,4,5]
reduce(add ,list_num)
# <==> 
sum(list_num)
```
这两个是一样的
这就是归约.

那在分布式的进程上应用归约函数就很麻烦(随之而来的是，难以有效地实现非可交换的归约，即必须以设定顺序发生的归约。)
===> MPI 有一个方便的函数，`MPI_Reduce`，它将处理程序员在并行程序中需要执行的几乎所有常见的归约操作

# MPI_Reduce
`MPI_Reduce` 在每个进程上获取一个输入元素数组，并将输出元素数组返回给根进程。 输出元素包含归约的结果
```C
MPI_Reduce(
	void* send_data,
	void* recv_data,
	int count,
	MPI_Datatype datatype,
	MPI_Op op,
	int root,
	MPI_Comm communicator
)
```
1. `send_data` 参数是每个进程都希望归约的 `datatype` 类型元素的数组。
2. `recv_data` 仅与具有 `root` 秩的进程相关
3. `recv_data` 数组包含归约的结果，大小为`sizeof（datatype）* count`。

`op` 参数是您希望应用于数据的操作。
MPI 包含一组可以使用的常见归约运算。 尽管可以定义自定义归约操作--->见==advanced==

- `MPI_MAX` - 返回最大元素。
- `MPI_MIN` - 返回最小元素。
- `MPI_SUM` - 对元素求和。
- `MPI_PROD` - 将所有元素相乘。
- `MPI_LAND` - 对元素执行逻辑_与_运算。
- `MPI_LOR` - 对元素执行逻辑_或_运算。
- `MPI_BAND` - 对元素的各个位按位_与_执行。
- `MPI_BOR` - 对元素的位执行按位_或_运算。
- `MPI_MAXLOC` - 返回最大值和所在的进程的秩。
- `MPI_MINLOC` - 返回最小值和所在的进程的秩。

![[Pasted image 20230918003809.png]]

多个数字归约!!!
上图中的每个进程都有两个元素。 结果求和基于每个元素进行。 
==换句话说，**不是将所有数组中的所有元素累加到一个元素中**，而是将每个数组中的第 i 个元素累加到进程 0 结果数组中的第 i 个元素中==
![[Pasted image 20230918003942.png]]

我们再次提升, 上一次用MPI_Satter和MPI_Gather 的均值计算代码
我们用进程归约实现:
# 使用 MPI_Reduce 计算均值
原始程序 [reduc_arg](https://github.com/mpitutorial/mpitutorial/blob/gh-pages/tutorials/mpi-reduce-and-allreduce/code/reduce_avg.c)
!!! 我们在这里用了 ring.c 的结构让进程按顺序输出!!!!!!!
==ring.c 是一个很重要的程序==

```C
#include <stdio.h>
#include <mpi.h>
#include <time.h>
#include <assert.h>
#include <stdlib.h>
// Creates an array of random numbers. Each number has a value from 0 - 1
float *create_rand_nums(int num_elements) {
  float *rand_nums = (float *)malloc(sizeof(float) * num_elements);
  assert(rand_nums != NULL);
  int i;
  for (i = 0; i < num_elements; i++) {
    rand_nums[i] = (rand() / (float)RAND_MAX);
  }
  return rand_nums;
}
  
int main (int argc,char** argv){
    if(argc !=2){
        fprintf(stderr, "Usage : avg num_elements_per_proc\n");
        exit(1);
    }
  
    int num_elements_per_proc = atoi(argv[1]);
  
    MPI_Init(&argc, &argv);
  
    int world_rank , world_size;
    MPI_Comm_rank(MPI_COMM_WORLD,&world_rank);
    MPI_Comm_size(MPI_COMM_WORLD,&world_size);
  
    //Create a random arrary on all process
    srand(time(NULL) * world_rank);
    float* rand_nums = NULL;
    rand_nums = create_rand_nums(num_elements_per_proc);
  
    //Sum the numbers locally(在每个进程上)!!!!
    float local_sum =0;
    int i;
    for (i=0; i< num_elements_per_proc;i++){
        local_sum +=rand_nums[i];
    }
  
    // Reduce all of the local sums into the global sum
    float global_sum; // 接收缓存------>只有根进程的可以接收
    MPI_Reduce(&local_sum,&global_sum,1,MPI_FLOAT,MPI_SUM,0,MPI_COMM_WORLD);
    // 我们修改了输出 , 借助了 ring.c 来实现了并行程序输出的顺序性
    int token;//令牌 :))))
    if(world_rank != 0){
        MPI_Recv(&token, 1,MPI_INT, world_rank -1,0,
            MPI_COMM_WORLD,MPI_STATUS_IGNORE);
        printf("Local sum for process %d - %f, avg = %f\n",
         world_rank, local_sum, local_sum / num_elements_per_proc);
    }else{
        printf("Local sum for process %d - %f, avg = %f\n",
            world_rank, local_sum, local_sum / num_elements_per_proc);
        token = -1;
    }
    MPI_Send(&token, 1, MPI_INT,(world_rank + 1) % world_size,0,MPI_COMM_WORLD);
    MPI_Barrier(MPI_COMM_WORLD);
    if(world_rank == 0){
        MPI_Recv(&token, 1,MPI_INT,world_size -1,0,
                MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        printf("\nTotal sum = %f, avg = %f\n",global_sum,
                global_sum / (world_size * num_elements_per_proc));//因为是每一个数的平均值,所以除以的是 world_size * num_elements_per_proc
    }
    free(rand_nums);
    MPI_Barrier(MPI_COMM_WORLD);
    MPI_Finalize();
}
```

在上面的代码中，每个进程都会创建随机数并计算和保存在 `local_sum` 中。 然后使用 `MPI_SUM` 将 `local_sum` 归约至根进程。 然后，全局平均值为 `global_sum / (world_size * num_elements_per_proc)`。
```OUTPUT
mpirun -n 14 ./reduce_avg 3
Local sum for process 0 - 2.017670, avg = 0.672557
Local sum for process 1 - 1.648353, avg = 0.549451
Local sum for process 2 - 1.222409, avg = 0.407470
Local sum for process 3 - 2.427584, avg = 0.809195
Local sum for process 4 - 1.511673, avg = 0.503891
Local sum for process 5 - 2.226954, avg = 0.742318
Local sum for process 6 - 0.919631, avg = 0.306544
Local sum for process 7 - 0.507806, avg = 0.169269
Local sum for process 8 - 1.719045, avg = 0.573015
Local sum for process 9 - 0.804838, avg = 0.268279
Local sum for process 10 - 1.514375, avg = 0.504792
Local sum for process 11 - 1.724022, avg = 0.574674
Local sum for process 12 - 2.294257, avg = 0.764752
Local sum for process 13 - 2.018888, avg = 0.672963

Total sum = 22.557507, avg = 0.537084
```

**现在是时候接触 `MPI_Reduce` 的同级对象 - `MPI_Allreduce` 了。**
类似其他的集体通讯函数他还有归约也有,MPI_Allreduce
# MPI_Allreduce

==许多并行程序中，需要在所有进程而不是仅仅在根进程中访问归约的结果。 以与 `MPI_Gather` 相似的补充方式，`MPI_Allreduce` 将归约值并将结果分配给所有进程。==
```C
MPI_Allreduce(
    void* send_data,
    void* recv_data,
    int count,
    MPI_Datatype datatype,
    MPI_Op op,
    MPI_Comm communicator)
```
我们注意到 MPI_Allreduce 和 MPI_Reduce 的 参数差不多!!!!
唯一的不同在于: 不需要根进程ID root
==因为他的结果分配给所有进程,而不是只有一个根进程==
![[Pasted image 20230918100331.png]]
他等价于
==先执行MPI_Reduce 然后执行 MPI_Bcast !!!!==
# 使用 MPI_Allreduce 计算标准差
许多计算问题需要进行**多次归约**来解决。
===> ==多次连续归约就要用 MPI_Allreduce==
**因为下一次归约是在第一次的基础上的---->且为非耦合运算**
![[Pasted image 20230918101103.png]]
一个这样的问题是找到一组分布式数字的标准差。
标准差是数字与均值之间的离散程度的度量。
较低的标准差表示数字靠得更近，对于较高的标准差则相反

要找到标准差，必须
1. 首先计算所有数字的平均值。 
2. 总和均值的平方根是最终结果。 

==给定问题描述，我们知道所有数字至少会有两个和，转化为两个归约==

Code:  
[reduce_stddev.c](https://github.com/mpitutorial/mpitutorial/tree/gh-pages/tutorials/mpi-reduce-and-allreduce/code/reduce_stddev.c)

```C
#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>
#include <math.h>
#include <time.h>
#include <assert.h>
    float *create_rand_nums(int num_elements) {
    float *rand_nums = (float *)malloc(sizeof(float) * num_elements);
    assert(rand_nums != NULL);
    int i;
    for (i = 0; i < num_elements; i++) {
        rand_nums[i] = (rand() / (float)RAND_MAX);
    }
    return rand_nums;
    }
  
    int main(int argc, char** argv) {
    if (argc != 2) {
        fprintf(stderr, "Usage: avg num_elements_per_proc\n");
        exit(1);
    }
  
    int num_elements_per_proc = atoi(argv[1]);
  
    MPI_Init(&argc, &argv);
  
    int world_rank;
    MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);
    int world_size;
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);
  
    // Create a random array of elements on all processes.
    srand(time(NULL)*world_rank); // Seed the random number generator of processes uniquely
    float *rand_nums = NULL;
    rand_nums = create_rand_nums(num_elements_per_proc);
  
    // Sum the number locally
    float local_sum =0;
    int i;
    for (i=0;i< num_elements_per_proc;i++){
        local_sum += rand_nums[i];
    }
  
    //Reduce all of the local sums into the global in order to calculate the mean
    float global_sum;
    MPI_Allreduce(&local_sum,&global_sum,1,MPI_FLOAT,MPI_SUM,
                    MPI_COMM_WORLD);
    //现在 通过 MPI_Allreduce 每一个进程都有了 SUM 的数据
    //然后对每个进程都执行Mean处理
    float mean = global_sum / (num_elements_per_proc * world_size);
  
    float local_sq_diff=0;
    for (i=0;i<num_elements_per_proc;i++){
        local_sq_diff +=(rand_nums[i] - mean) * (rand_nums[i] - mean);
    }
  
    // Reduce the global sum of the squared diiferences to the root process and print off the answer
    float global_sq_diff;
   MPI_Reduce(&local_sq_diff,&global_sq_diff,1,MPI_FLOAT,MPI_SUM,0,MPI_COMM_WORLD);

  
    // The standard deviation is the square root of the mean of the squared differences.
    if(world_rank == 0){
        float stddev = sqrt(global_sq_diff /(num_elements_per_proc*world_size));
        printf("Mean - %f , Standard deviation = %f\n",mean ,stddev);
    }
  
    free(rand_nums);
    MPI_Barrier(MPI_COMM_WORLD);
    MPI_Finalize();
}
```


![[Pasted image 20230918104501.png]]
```OUTPUT
mpicc stddev_allred.c -o stddev_allred -lm
mpirun -n 14 ./stddev_allred  3
Mean - 0.545230 , Standard deviation = 0.278325
```

在上面的代码中，每个进程都会计算元素的局部总和 `local_sum`，并使用 `MPI_Allreduce` 对它们求和。 在所有进程上都有全局总和后，将计算均值 `mean`，以便可以计算局部距平的平方 `local_sq_diff`。
一旦计算出所有局部距平的平方，就可以通过使用 `MPI_Reduce` 得到全局距平的平方 `global_sq_diff`。 然后，根进程可以通过取全局距平的平方的平均值的平方根来计算标准差。


