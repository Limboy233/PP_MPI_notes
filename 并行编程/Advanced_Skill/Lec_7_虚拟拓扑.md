在并行计算中，通过标准通信子MPI_COMM_WORLD，每个进程都可以与其他进程交换数据。但在某些情况下，我们可能需要选择一部分进程进行交互，或者按特定方式组织进程之间的通信，而不仅仅是使用MPI_COMM_WORLD。
为此，MPI库提供了一种方法，允许我们为并行应用程序中的一组或所有进程定义虚拟拓扑结构，也就是所谓的"**虚拟拓扑**"。

虚拟拓扑结构将一组进程组织成一种结构，允许更复杂的进程排列，而不仅仅是线性的方式。有两种主要类型的虚拟拓扑：笛卡尔拓扑和图拓扑

1) 在笛卡尔拓扑（Cartesian topology）中，所有进程被视为n维网格（grid）的节点，这个网格的大小为k1 x k2 x ... x kn（如果n = 2，则可以将进程看作是k1 x k2的矩阵元素）。
2) 在图拓扑（graph topology）中，进程被视为图的节点，进程之间的关系由图的边缘定义。

3) 在MPI-2标准中，还引入了一种特殊的分布式图拓扑，称为"分布式图拓扑"（distributed graph topology）。

5. 笛卡尔拓扑
有关使用的虚拟拓扑结构信息与通信子关联在一起。要检查通信子comm是否与虚拟拓扑结构相关，可以使用MPI_Topo_test函数，该函数将虚拟拓扑的类型存储在status参数中。

status参数可以具有以下值：
- MPI_CART：通信子与笛卡尔拓扑相关
- MPI_GRAPH：通信子与图拓扑相关
- MPI_DIST_GRAPH：通信子与分布式图拓扑相关（这个常量在MPI-2标准中引入）
- MPI_UNDEFINED：通信子与任何虚拟拓扑都不相关

在本节中，我们将重点介绍与笛卡尔拓扑相关的MPI函数。这些函数可以分为四组：

==重点API== :
1. 创建笛卡尔拓扑结构以及与之相关的通信子的函数（MPI_Cart_create和辅助函数MPI_Dims_create）。
2. 获取已有笛卡尔拓扑结构的特性信息的函数（MPI_Cartdim_get、MPI_Cart_get、MPI_Cart_rank和MPI_Cart_coords）。
3. 将现有笛卡尔网格分割为较低维度子网格的函数（MPI_Cart_sub）。
4. 在笛卡尔网格上确定数据沿某个坐标轴的发送和接收的进程的排名（MPI_Cart_shift）。

Вот основные функции и понятия, связанные с Декартовой топологией:

1. **MPI_Cart_create**: Эта функция используется для создания коммуникатора с Декартовой топологией. Она принимает параметры, такие как общее количество процессов, размер решетки в каждом измерении и логический флаг, который указывает, нужно ли создавать циклическую топологию. Пример:
```C
int dims[2] = {3, 2}; // 2x3 решетка
int periods[2] = {0, 1}; // Не циклическая в первом измерении, циклическая во втором
MPI_Cart_create(MPI_COMM_WORLD, 2, dims, periods, 0, &cart_comm);

```
1. `MPI_Comm comm_old`: 一个输入参数，代表您要在其基础上创建笛卡尔拓扑的旧通信子。
    
2. `int ndims`: 一个整数，指定笛卡尔拓扑的维度数。通常，这对应于物理空间的维度。例如，如果您有一个 2D 网格，ndims 将为 2。
    
3. `const int dims[]`: 一个整数数组，包含每个维度的大小。`dims` 的长度必须等于 `ndims`，并且每个元素表示该维度上的进程数。
    
4. `const int periods[]`: 一个布尔数组，长度也为 `ndims`，表示每个维度是否是周期性的。如果 `periods[i]` 为 true，那么在该维度上的拓扑是周期性的，否则不是。周期性表示在边界上的进程是相邻边界上的进程。
    
5. `int reorder`: 一个布尔值，指示 MPI 是否可以重新排列通信子以最佳方式安排进程，以便提高通信性能。如果为真，MPI 可能会重新排列通信子，如果为假，则通信子将保留进程的原始顺序。


2. **MPI_Cartdim_get**: Эта функция позволяет получить количество измерений в Декартовой топологии. Пример:
```C
int ndims;
MPI_Cartdim_get(cart_comm, &ndims);
```
    
3. **MPI_Cart_get**: Эта функция позволяет получить информацию о текущей Декартовой топологии, такую как размер решетки, периодичность и координаты текущего процесса. Пример:
```C
	int new_rank;
    MPI_Comm_rank(newcomm,&new_rank);
    int* Cart =new int[2];

    MPI_Cart_coords(newcomm,new_rank,2,Cart);
	//----> 因为我们这里是二维笛卡尔拓扑(2D grids)
	delete[] Cart;


``` 
4. **MPI_Cart_rank** и **MPI_Cart_coords**: Эти функции позволяют преобразовывать ранги процессов в Декартовой топологии в их координаты и наоборот. Пример:
**生成笛卡尔拓扑坐标的参数,需要==笛卡尔拓扑通信子的rank==(而不是原来通信子(MPI_COMM_WORLD)的rank)**

==MPI_Cart_coords 提供了个在笛卡尔通信拓扑中 ranks表示 和 坐标表示的转换 !!!!!==
    
```C
int rank, coords[2];
MPI_Comm_rank(cart_comm, &rank);
MPI_Cart_coords(cart_comm, rank, 2, coords);
```

5. **MPI_Cart_sub**: Эта функция позволяет разбивать Декартову топологию на подрешетки меньшей размерности, создавая новые коммуникаторы для них.
	`MPI_Cart_sub` 函数用于创建一个新的通信子（communicator），该通信子是从原始的笛卡尔拓扑通信子中派生的子集。这可以用于减小通信子的大小，使进程在一个较小的拓扑子集上进行通信。下面是 `MPI_Cart_sub` 函数的参数以及它们的意义：
1. `MPI_Comm comm`: 一个输入参数，代表您要从中派生子通信子的原始笛卡尔拓扑通信子。
2. `const int remain_dims[]`: 一个整数数组，用于指定哪些维度应该在新的子通信子中保留。如果 `remain_dims[i]` 等于 1，那么第 `i` 维将保留在新通信子中。如果 `remain_dims[i]` 等于 0，那么第 `i` 维将在新通信子中被删除.
3. `MPI_Comm* newcomm`: 一个输出参数，用于存储派生的新通信子

example: **Question**--->==MPI_Cart_sub (**按层分割**)==
The number of processes _K_ is equal to 8 or 12. A real number is given in each process. Define a Cartesian topology for all processes as a three-dimensional (2 × 2 × _K_/4) grid (ranks of processes should not be reordered), which should be considered as _K_/4 two-dimensional (2 × 2) subgrids (namely, _matrices_) that contain processes with the identical third coordinate in the Cartesian topology. Split this grid into _K_/4 matrices of processes. Using one global reduction operation, find the sum of all numbers given in the processes of each matrix. Output the sum in the master process of the corresponding matrix.
```C
	int rank, size;
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

    double num;pt >> num;

    int dims[3] ={2,2,(int)(size/4)};
    int periods[3] ={1,0,0};
    MPI_Comm newcomm;  MPI_Cart_create(MPI_COMM_WORLD,3,dims,periods,0,&newcomm);

    int remain[3]={1,1,0};//Tensor--> (2*2) * size/num floor
    MPI_Comm sub_comm_Cart;
    MPI_Cart_sub(newcomm,remain,&sub_comm_Cart);
    //然后我们通过 MPI_Cart_sub ---> 把Tensor 按层分割了
    int subrank,subsize;
    MPI_Comm_rank(sub_comm_Cart,&subrank);
    MPI_Comm_size(sub_comm_Cart,&subsize);

    //每个进程得到自己在自己层中的大小
    double save;
    // 求和约归到根进程
       MPI_Reduce(&num,&save,1,MPI_DOUBLE,MPI_SUM,0,sub_comm_Cart);

    //再把这个从根进程广播到每一层的所有进程
    MPI_Bcast(&save,1,MPI_DOUBLE,0,sub_comm_Cart);
    pt << save;

    MPI_Comm_free(&sub_comm_Cart);
    MPI_Comm_free(&newcomm);
```
    
6. **MPI_Cart_shift**: Эта функция позволяет определить ранги источников и приемников при передаче данных вдоль определенного измерения Декартовой топологии.


这些函数有助于在进程之间建立更复杂的交互，根据它们在笛卡尔网格中的坐标来定位进程。这对于许多并行计算任务，如网格方法和多维问题，非常有用。
![[Pasted image 20231019084222.png]]
![[Pasted image 20231019084614.png]]
![[Pasted image 20231019084336.png]]
![[Pasted image 20231019084847.png]]
![[Pasted image 20231019084731.png]]
轮换的坐标维取 0 , 1 表示纵轮换与横轮换

![[Pasted image 20231019084758.png]]
![[Pasted image 20231019085038.png]]
![[Pasted image 20231019085103.png]]
![[Pasted image 20231019085128.png]]
![[Pasted image 20231019085156.png]]


使用场景例子: MPI5Comm17,
![[Pasted image 20231026163634.png]]


3. 图拓扑

现在，让我们来看一下另一种虚拟拓扑结构 - 图形拓扑。需要注意的是，与笛卡尔拓扑不同，MPI库提供了较少的功能来处理图形拓扑。回顾一下，对于参与笛卡尔拓扑的进程，可以根据它们的排名来确定它们的笛卡尔坐标（反之亦然 - 根据笛卡尔坐标确定排名）。此外，还可以创建较低维度的子网格（每个子网格都自动与新通信子关联），还有一个MPI_Cart_shift函数，它简化了沿着笛卡尔网格某个坐标的消息传递。

至于图形拓扑，一旦使用MPI_Graph_create函数定义了它，您只能恢复其特性（使用MPI_Graphdims_get和MPI_Graph_get函数），还可以获取有关特定进程在由该拓扑定义的图形中的邻居数量和排名的信息。为此，可以使用MPI_Graph_neighbors_count和MPI_Graph_neighbors函数。

[这里有中文的详细介绍](https://scc.ustc.edu.cn/zlsc/cxyy/200910/MPICH/mpi653.htm)

1) **MPI_Graph_create**是一个 MPI 函数，用于创建一个具有图拓扑结构的通信器。这个函数允许您定义进程之间的拓扑关系，以便进行相应的通信
```C
int MPI_Graph_create(MPI_Comm comm_old, int nnodes, const int indx[], const int edges[], int reorder, MPI_Comm *comm_graph);
```
- `comm_old`: 旧通信器，也就是要使用的原始通信器。
- `nnodes`: 图中的节点数量，通常**等于进程数量**。
- `indx[]`: 一个整数数组，==长度为 `nnodes`==，指定**节点的度**。例如，数组第i号单元的index 存储了前i个图结点的邻居的总数
- `edges[]`: 一个整数数组，==长度为 `nnodes`==，**它包含图中所有的边**。数组edges表示为一个展开的边列表
边的数量等于 `edges[nnodes-1]`。
通常，**`edges` 包含相邻节点的排列，以表示图的连接关系。**

- `reorder`: 一个布尔值，如果设置为 1，MPI 可以重新排序节点的排列以提高通信性能；如果设置为 0，则节点顺序保持不变。
- `comm_graph`: 新的通信器，包含了定义的图拓扑结构。

![[Pasted image 20231028220429.png]]

indx[i]=a ===> 数组第i号单元的index 存储了前i个图结点的邻居的总数
edges[i] ===> 数组edges表示为一个展开的边列表

**这种方式就是图论中描述图的方式, 忘记了可以重新看离散数学!!!**

2) **MPI_Graph_neighbors**
函数用于获取图拓扑中指定节点的相邻节点。这个函数可以帮助您确定一个节点在图拓扑中与哪些其他节点直接相连
```C
int MPI_Graph_neighbors(MPI_Comm comm, int rank, int maxneighbors, int neighbors[])
```

- `comm`：输入，图拓扑的通信器。
- `rank`：输入，要查询其相邻节点的节点的秩（排名）。
- `maxneighbors`：输入，`neighbors` 数组的最大大小，通常设置为节点的最大相邻节点数。
- `neighbors`：输出，存储相邻节点的秩（排名）的数组。
==重点== :请注意，`MPI_PROC_NULL` 表示不存在的相邻节点。根据图拓扑的结构，某些相邻节点可能不存在，因此需要检查 `neighbors` 数组中的每个元素是否等于 `MPI_PROC_NULL`
```C
MPI_Comm G_comm;    MPI_Graph_create(MPI_COMM_WORLD,9,index,edges,0,&G_comm);

    int new_rank;
    MPI_Comm_rank(G_comm,&new_rank);

    const int maxneighbour=4;
    int neighb[4];
    MPI_Graph_neighbors(G_comm,new_rank,maxneighbour,neighb);
    
for (int i = 0; i < maxneighbor; i++) {
    if (neighb[i] != MPI_PROC_NULL) {
        printf("%d ", neighb[i]);
    }
}
printf("\n");

```
3) **MPI_Graph_neighbors_count**
函数用于获取图拓扑中指定节点的相邻节点数目。这个函数返回指定节点的相邻节点数量，有助于确定图拓扑中每个节点与多少其他节点直接相连。
```C
int MPI_Graph_neighbors_count(MPI_Comm comm, int rank, int *nneighbors)
```
- `comm`：输入，图拓扑的通信器。
- `rank`：输入，要查询其相邻节点数量的节点的秩（排名）。
- `nneighbors`：输出，存储相邻节点数量的整数。
```C
MPI_Comm G_comm;    MPI_Graph_create(MPI_COMM_WORLD,9,index,edges,0,&G_comm);

    int new_rank;
    MPI_Comm_rank(G_comm,&new_rank);
    int rank, size;
int nneighbors;
MPI_Graph_neighbors_count(G_comm, rank, &nneighbors);

printf("Node %d has %d neighbors.\n", rank, nneighbors);

```
==这个函数对于确定图拓扑中每个节点的连接性很有用，因为不同节点可能与不同数量的相邻节点直接相连。==

4) ==**上文所有的rank 都要用MPI_Comm_rank 在图拓扑中读取一次**== ![[Pasted image 20231026172618.png]]

例子: MPI5Comm29
![[Pasted image 20231026163751.png]]
![[Pasted image 20231026163818.png]]

![[Pasted image 20231026171723.png]]