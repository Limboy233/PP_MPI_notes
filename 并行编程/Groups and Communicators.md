在之前,我们使用的通讯器(communicators)是MPI_COMM_WORLD----->**它是把所有进程打包成一个通讯器,是最原始的通讯器**

 对于简单的程序，这已经足够了，因为我们的进程数量相对较少，并且通常要么一次要与其中之一对话，要么一次要与所有对话。 
 
 ==但是==, 当程序规模开始变大时，这变得不那么实用了，我们可能**只想一次与几个进程进行对话。**---->==这是原因== 
 
 
 在本次教程中，我们将展示**如何创建新的通讯器，以便一次与原始线程组的子集进行沟通。**

# Communicators
正如我们在学习集体例程时所看到的那样，MPI 允许您立即与通讯器中的所有进程进行对话，以执行诸如使用 `MPI_Scatter` 将数据从一个进程分发到多个进程或使用 `MPI_Reduce` 执行数据归约的操作。 但是，到目前为止，**我们仅使用了默认的通讯器 `MPI_COMM_WORLD`。**

对于简单的应用程序，使用 `MPI_COMM_WORLD` 进行所有操作并不罕见，但是对于更复杂的用例，拥有更多的通讯器可能会有所帮助。
1. 如果您想对网格中进程的子集执行计算。 
2. 每一行中的所有进程都可能希望对一个值求和。

第一个也是最常见的用于创建新的通讯器的函数: 
**把原始的,最大的那个通讯器(MPI_COMM_WORLD)进行拆分**
-----> 函数: 
##  MPI_Comm_split
```C
MPI_Comm_split(
	MPI_Comm comm,
	int color,
	int key,
	MPI_Comm* newcomm)
```
`MPI_Comm_split` 通过基于输入值 `color` 和 `key` 将通讯器“拆分”为一组子通讯器来创建新的通讯器

TIPS:==原始的通讯器并没有消失，但是在每个进程被分配到一个新的通讯器中==
1.  第一个参数 `comm` 是通讯器，它将用作新通讯器的基础。 这可能是 `MPI_COMM_WORLD`，但也可能是其他任何通讯器。 
2. 第二个参数 `color` 确定每个进程将属于哪个新的通讯器。 为 ==`color` 传递相同值的所有进程都分配给同一通讯器==。 如果 `color` 为 **MPI_UNDEFINED，则该进程将不包含在任何新的通讯器中**。 
3. 第三个参数 `key` ==确定每个新通讯器中的顺序（秩）==。 传递 `key` 最小值的进程将为 0，下一个最小值将为 1，依此类推。 如果存在平局，则在原始通讯器中秩较低的进程将是第一位。 
4. 最后一个参数 `newcomm` 是 MPI 如何将新的通讯器返回给用户。

## 使用多个通讯器的示例

![[Pasted image 20230914091356.png]]
Code:
```C
#include <mpi.h>
#include <stdio.h>

int main(int argv,char** argc){
	MPI_Init(&argv,&argc);
	//获取原始通讯器的秩和大小
	int world_rank , world_size;
	MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);
	MPI_Comm_size(MPI_COMM_WORLD, &world_size);

	//根据行确定颜色(因为我们这个问题是规律分解,所以是统一的就好了,利用余数环性质,但是如果不是统一的那就要每个进程各个分配color,有点麻烦........)
	int color = world_rank / 4;

	//根据颜色拆分原始通讯器,然后告诉每个进程你是哪个通讯器
	MPI_Comm row_comm; //给每个进程存它自己所在通讯器
    MPI_Comm_split(MPI_COMM_WORLD,color,world_rank,&row_comm);


	int row_rank,row_size;//每个进程读取他所在新通讯器的信息
	MPI_Comm_rank(row_comm,&row_rank);
	MPI_Comm_size(row_comm,&row_size);

	printf("WORLD RANK/SIZE : %d/%d ROW RANK/SIZE: %d/%d \n",
			world_rank, world_size,row_rank,row_size);

	//MPI_Barrier(MPI_COMM_WORLD);
	if(row_comm!= MPI_COMM_NULL){
		MPI_Comm_free(&row_comm);
	}
}
```

MPI_Comm_split() ---> 是在每一个进程上调用然后告诉该进程他属于哪一个新的通讯器
**Color** 相当于进程的Tag(标签)

==所以,在读取新通讯器信息时,我们要保证,每一个要分配到新通讯器的进程都已经分配到了,即都调用了函数MPI_Comm_split(),具有了color==

**程序解释:**
前几行获得原始通讯器 `MPI_COMM_WORLD` 的秩和大小。 
下一行执行确定局部进程 `color` 的重要操作。 
请记住，`color` 决定了拆分后该进程所属的通讯器。 
接下来，我们将看到所有重要的拆分操作。 
这里的新事物是，我们使用原始秩（`world_rank`）作为拆分操作的 `key`。
由于我们希望新通讯器中的所有进程与原始通讯器中的所有进程处于相同的顺序，因此在这里使用原始等级值最有意义，因为它已经正确地排序了。

OUTPUT:
```
WORLD RANK/SIZE: 0/16 	 ROW RANK/SIZE: 0/4
WORLD RANK/SIZE: 1/16 	 ROW RANK/SIZE: 1/4
WORLD RANK/SIZE: 2/16 	 ROW RANK/SIZE: 2/4
WORLD RANK/SIZE: 3/16 	 ROW RANK/SIZE: 3/4
WORLD RANK/SIZE: 4/16 	 ROW RANK/SIZE: 0/4
WORLD RANK/SIZE: 5/16 	 ROW RANK/SIZE: 1/4
WORLD RANK/SIZE: 6/16 	 ROW RANK/SIZE: 2/4
WORLD RANK/SIZE: 7/16 	 ROW RANK/SIZE: 3/4
WORLD RANK/SIZE: 8/16 	 ROW RANK/SIZE: 0/4
WORLD RANK/SIZE: 9/16 	 ROW RANK/SIZE: 1/4
WORLD RANK/SIZE: 10/16 	 ROW RANK/SIZE: 2/4
WORLD RANK/SIZE: 11/16 	 ROW RANK/SIZE: 3/4
WORLD RANK/SIZE: 12/16 	 ROW RANK/SIZE: 0/4
WORLD RANK/SIZE: 13/16 	 ROW RANK/SIZE: 1/4
WORLD RANK/SIZE: 14/16 	 ROW RANK/SIZE: 2/4
WORLD RANK/SIZE: 15/16 	 ROW RANK/SIZE: 3/4
```
如果您的顺序不正确，请不要惊慌。 当您在 MPI 程序中显示内容时，每个进程都必须将其输出发回启动 MPI 作业的位置，然后才能将其打印到屏幕上。 这往往意味着排序变得混乱，因为您永远无法假设仅以特定的秩顺序打印内容，即输出结果实际上将按照您期望的顺序结束，但是显示不是。(==如果想按顺序输出的画可以加上之前写过的ring.c 程序 用 获得token的顺序 来让他们顺序输出 :))))))))))))))))) ==) 

最后，我们使用 `MPI_Comm_free` 释放通讯器。 这似乎不是一个重要的步骤，但与在其他任何程序中使用完内存后释放内存一样重要。 
==当不再使用 MPI 对象时，应将其释放，以便以后重用。==
**why?**
MPI 一次可以创建的对象数量有限，如果 MPI 用完了可分配对象，则不释放对象可能会导致运行时错误.

# **最重要的是**这个 MPI_Comm_free()它调用一次,全体线程的该Communicator都不存在了,==所以我们在free之前要做一次阻塞(MPI_Barrier(MPI_COMM_WORLD);),确定所有线程关于要释放的这个Communicator的操作都完成了才可以释放它==

why? **MPI_Barrier(MPI_COMM_WORLD)**
因为我要对所有线程来一个同步呀,不只是在row_comm 通讯器内的,故是在MPI_COMM_WORLD中.

==但其实==--->**好像也不是这个问题**: MPI_Comm_free() 好像内置了阻塞 :))))))

最重要的是
```C
	if(row_comm!= MPI_COMM_NULL){
		MPI_Comm_free(&row_comm);
	}
```

==MPI_COMM_NULL 的 MPI_Comm对象可不能free掉呀!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!==

**在这里问题不大但是在下面的例子中,问题很大因为他用特定组建立通讯器,所以有些进程没有在新通讯器中,那他们进程里的新MPI_Comm 对象就是 MPI_COMM_NULL 它可不能给free掉**

# 其他通讯器创建函数

1. ==MPI_Comm_dup==是最基本的，它创建了一个通讯器的副本。 似乎存在一个仅创建副本的函数似乎很奇怪，但这对于使用库执行特殊函数的应用（例如数学库）非常有用。 在这类应用中，重要的是用户代码和库代码不要互相干扰。 为了避免这种情况，每个应用程序应该做的第一件事是创建 `MPI_COMM_WORLD` 的副本，这将避免其他使用 `MPI_COMM_WORLD` 的库的问题。 库本身也应该复制 `MPI_COMM_WORLD` 以避免相同的问题。

2. ==MPI_Comm_create==。 此功能与 `MPI_Comm_create_group` 非常相似。 其原型几乎相同：

```
MPI_Comm_create(
	MPI_Comm comm,
	MPI_Group group,
    MPI_Comm* newcomm)
```

然而，主要区别（除了缺少 `tag` 参数之外）是，`MPI_Comm_create_group` 仅是 `group` 中包含的一组进程的集合，而 `MPI_Comm_create` 是 `comm` 中每个进程的集合。 

**当通讯器的规模很大时，这是一个重要的区别**。 如果尝试在运行 1,000,000 个进程时创建 `MPI_COMM_WORLD` 的子集，则重要的是使用尽可能少的进程来执行该操作，因为大型集的开销会变得非常昂贵。

# 其他通讯器功能
通讯器还有其他一些更高级的功能，我们在这里不介绍，例如内部通讯器与内部通讯器之间的差异以及其他高级通讯器创建功能。 这些仅用于非常特殊的应用
-------> 在**[[Advanced]] 部分会介绍**


# ==其他通讯器创建方法==  :----> 借助集合理论!!!!(set)

MPI 的Group(**组**) <---(对应)----> 集合论中的Set(**集合**)
# Groups--> 这是一个不同于Communicators(MPI_Comm)对象的一个新MPI对象Groups(MPI_Grouo),==组由通讯器创建再由它(组)为中间媒介,再次建立新的通讯器==


**意义在于: 为新通讯器的创建引入了集合理论,使我们可以进行更加复杂的通讯器创建与进程分配**

## 我们先回顾一下通讯器的含义 : 

1. 在内部,MPI必须保持通讯器的两个主要部分:
		1) 区分一个通讯器与另一个通讯器的上下文(或ID)
		2) 该通讯器的一组进程--->(a Group of process)

上下文阻止了与一个通讯器上的操作匹配到另一通讯器上的类似操作
---->  MPI 在内部为每个通讯器保留一个 ID，以防止混淆

## 而组 : 易于理解，因为它只是通讯器中所有进程的集合

1. 对于 `MPI_COMM_WORLD`，组是由 `mpiexec` 启动的所有进程
2. 对于其他通讯器，组将有所不同。 在上面的示例代码中，组是所有以相同的 `color` 传参给 `MPI_Comm_split` 的进程。(也就是下一级的通讯器)
------> ==因为我并没有在通讯器中分组,
所以一个通讯器就是一个组==

接下来,我们**在一个通讯器中分组**,会使用 **MPI_Group_incl()**
----> 它会使一个通讯器中有多个组,然后我们对这些组进行运算,最后用得到的组来构建我们所期望的通讯器




# 一. (通讯器组生成函数)MPI_Comm_group
它是由这个communitor(MPI_Comm comm)生成的一个Group(MPI_Group* group),这个组里包含这个Communicator 的所有进程(==也就是上完说的一个通讯器就是一个组==)-->**这个组是由这个函数生成的**
```C
MPI_Comm_group(
	MPI_Comm comm,
	MPI_Group* group)
```
调用 `MPI_Comm_group` 会得到对该组对象的引用。 
组对象的工作方式与通讯器对象相同，**不同之处在于您不能使用它与其他秩进行通信（因为它没有附加上下文)**。
您仍然可以获取组的秩和大小（分别为 **MPI_Group_rank** 和 **MPI_Group_size**）。

==组特有的功能而通讯器无法完成的工作是可以使用组在本地构建新的组。==

在此记住本地操作和远程操作之间的区别很重要:

1. 远程操作涉及与其他秩的通信，而本地操作则没有。 
2. 创建新的通讯器是一项远程操作，因为所有进程都需要决定相同的上下文和组，而在本地创建组是因为它不用于通信，因此每个进程不需要具有相同的上下文。
3. 您可以随意操作一个组，而无需执行任何通信。

=> **并**：

```C
MPI_Group_union(
	MPI_Group group1,
	MPI_Group group2,
	MPI_Group* newgroup)
```
**交**：

```C
MPI_Group_intersection(
	MPI_Group group1,
	MPI_Group group2,
	MPI_Group* newgroup)
```

在这两种情况下，操作均在 `group1` 和 `group2` 上执行，结果存储在 `newgroup` 中。

![[Pasted image 20230914171054.png]]

## Group更多用法:
1. 您可以比较组以查看它们是否相同，
2. 从另一个组中减去一个组，
3. 从组中排除特定秩，
4. 或使用一个组将一个组的秩转换为另一组。

## 但最重要的两个是:
1. **MPI_Group_incl** :该函数允许您选择**组**中的特定秩并**构建为新组**
2. **MPI_Comm_create_group**: 这是一个用于==创建新通讯器==的函数
```C
MPI_Group_incl(
	MPI_Group group,
	int n,
	const int ranks[],
	MPI_Group* newgroup)
```
MPI_Group group :原始组
int n:新租包含的线程数
const int ranks[]:化为新组的原来组的rank号
MPI_Group* newgroup: 新组存放位置

该函数中，`newgroup` 将包含 `group` 中的秩存在于 `ranks` 数组中的 `n` 个进程

```C
MPI_Comm_create_group(
	MPI_Comm comm,
	MPI_Group group,
	int tag,
	MPI_Comm* newcomm)
```
这是一个用于创建新通讯器的函数，但无需像 `MPI_Comm_split` 之类那样需要进行计算以决定组成，该函数将使用一个 `MPI_Group` 对象并创建一个与组具有相同进程的新通讯器(MPI_Comm* newcomm).

# 创建一个包含来自 `MPI_COMM_WORLD` 的主要秩的通讯器(Application)

```C
#include <stdio.h>

#include <mpi.h>

  

int main(int argc,char** argv){

  

    MPI_Init(&argc,&argv);

  

    //获取本线程 原始通讯器大小和其自身rank

    int world_size,world_rank;

    MPI_Comm_rank(MPI_COMM_WORLD,&world_rank);

    MPI_Comm_size(MPI_COMM_WORLD,&world_size);

  

    //获取由原通讯器MPI_COMM_WORLD生成的进程组

    MPI_Group world_group;

    MPI_Comm_group(MPI_COMM_WORLD,&world_group);

  

    int n =7;

    const int ranks[7] = {1, 2, 3, 5, 7, 11, 13};

  

    //构造一个包含 world_group 中所有主要秩的组

    MPI_Comm prime_group;

    MPI_Group_incl(world_group, 7, ranks, &prime_group);

  

    //根据组创建一个新的通讯器,包含原来的{1, 2, 3, 5, 7, 11, 13};

    MPI_Comm prime_comm;

    MPI_Comm_create_group(MPI_COMM_WORLD, prime_group,0,&prime_comm);

  

    int prime_rank =-1, prime_size = -1;//用-1----->方便勘误!!!!

    //如果这个rank的进程不在新communicator中,prime_comm为MPI_COMM_NULL

    //故我们用这个作为错误判断

  

    if(prime_comm != MPI_COMM_NULL){

        MPI_Comm_rank(prime_comm, &prime_rank);

        MPI_Comm_size(prime_comm, &prime_size);

    }

  

    printf("WORLD RANK/SIZE: %d/%d \t PRIME RANK/SIZE: %d/%d\n",

            world_rank,world_size,prime_rank,prime_size);

  

    //先来一个同步(阻塞)

    MPI_Barrier(MPI_COMM_WORLD);

    //用完的Group要free掉

    MPI_Group_free(&world_group);

    MPI_Group_free(&prime_group);

    //新建的Communicator也要free掉

    MPI_Comm_free(&prime_comm);

  

    MPI_Finalize();

  

}
```

注意要先同步完所有进程(MPI_Barrier(MPI_COMM_WORLD);),确保MPI_Group和MPI_Comm 对象都使用完成了再free掉.
但是他的输出==还是有错误的==,我不知道原因.
```
mpicc groups_comm.c -o groups_comm
```

```
mpirun -n 14 ./groups_comm

WORLD RANK/SIZE: 12/14   PRIME RANK/SIZE: -1/-1
WORLD RANK/SIZE: 6/14    PRIME RANK/SIZE: -1/-1
WORLD RANK/SIZE: 10/14   PRIME RANK/SIZE: -1/-1
WORLD RANK/SIZE: 4/14    PRIME RANK/SIZE: -1/-1
WORLD RANK/SIZE: 0/14    PRIME RANK/SIZE: -1/-1
WORLD RANK/SIZE: 9/14    PRIME RANK/SIZE: -1/-1
WORLD RANK/SIZE: 1/14    PRIME RANK/SIZE: 0/7
WORLD RANK/SIZE: 7/14    PRIME RANK/SIZE: 4/7
WORLD RANK/SIZE: 11/14   PRIME RANK/SIZE: 5/7
WORLD RANK/SIZE: 13/14   PRIME RANK/SIZE: 6/7
WORLD RANK/SIZE: 3/14    PRIME RANK/SIZE: 2/7
WORLD RANK/SIZE: 2/14    PRIME RANK/SIZE: 1/7
WORLD RANK/SIZE: 8/14    PRIME RANK/SIZE: -1/-1
WORLD RANK/SIZE: 5/14    PRIME RANK/SIZE: 3/7
Abort(68243205) on node 0 (rank 0 in comm 0): Fatal error in internal_Comm_free: Invalid communicator, error stack:
internal_Comm_free(87): MPI_Comm_free(comm=0x7ffcad44ec58) failed
internal_Comm_free(44): Null communicator
```

# 我现在知道了:==我们free的时候只能free哪些不为 MPI_COMM_NULL 的MPI_Comm对象==

**更改代码最后面**
```C
...
    //先来一个同步(阻塞)
    //MPI_Barrier(MPI_COMM_WORLD);
    //用完的Group要free掉
    MPI_Group_free(&world_group);
    MPI_Group_free(&prime_group);
    if(prime_comm != MPI_COMM_NULL){
        //新建的Communicator也要free掉
        MPI_Comm_free(&prime_comm);
    }
    MPI_Finalize();
```
```OUTPUT
mpirun -n 14 ./groups_comm
WORLD RANK/SIZE: 12/14   PRIME RANK/SIZE: -1/-1
WORLD RANK/SIZE: 0/14    PRIME RANK/SIZE: -1/-1
WORLD RANK/SIZE: 4/14    PRIME RANK/SIZE: -1/-1
WORLD RANK/SIZE: 9/14    PRIME RANK/SIZE: -1/-1
WORLD RANK/SIZE: 3/14    PRIME RANK/SIZE: 2/7
WORLD RANK/SIZE: 8/14    PRIME RANK/SIZE: -1/-1
WORLD RANK/SIZE: 10/14   PRIME RANK/SIZE: -1/-1
WORLD RANK/SIZE: 1/14    PRIME RANK/SIZE: 0/7
WORLD RANK/SIZE: 5/14    PRIME RANK/SIZE: 3/7
WORLD RANK/SIZE: 6/14    PRIME RANK/SIZE: -1/-1
WORLD RANK/SIZE: 7/14    PRIME RANK/SIZE: 4/7
WORLD RANK/SIZE: 2/14    PRIME RANK/SIZE: 1/7
WORLD RANK/SIZE: 13/14   PRIME RANK/SIZE: 6/7
WORLD RANK/SIZE: 11/14   PRIME RANK/SIZE: 5/7
```

1. 在此示例中，我们通过仅选择 `MPI_COMM_WORLD` 中的主要秩来构建通讯器。
2. 这是通过 `MPI_Group_incl` 完成的，并将结果存储在 `prime_group` 中。 
3. 接下来，我们将该组传递给 `MPI_Comm_create_group` 以创建 `prime_comm`。 
4. 最后，我们必须小心不要在没有 `prime_comm` 的进程上使用 `prime_comm`，因此我们要检查以确保通讯器不是 `MPI_COMM_NULL` 状态 —— 不在 `ranks` 中而从 `MPI_Comm_create_group` 返回的结果。


==然后发现 **同步** MPI_Group_free() 和 MPI_Comm_free() 是会自己做的==: )))

5. 我们也不能释放掉`MPI_COMM_NULL` 状态的通讯器 :))))))))
最重要的是
```C
    if(prime_comm != MPI_COMM_NULL){
        //新建的Communicator也要free掉
        MPI_Comm_free(&prime_comm);
    }
```

==MPI_COMM_NULL 的 MPI_Comm对象可不能free掉呀!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!==

