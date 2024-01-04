这一组MPI9Matr的教学任务与以往不同，它专注于并行矩阵算法，这些算法利用了MPI库的各种功能，包括不同进程间的互动方式、新的派生类型、带有虚拟拓扑结构的通信子以及并行文件I/O。

因此，MPI9Matr组可以看作是一个总结性的组，可以复习和巩固与MPI技术相关的大部分先前学习内容。

矩阵处理是一种需要高效并行化方法的计算任务之一，其中包括基于MPI技术的方法。需要指出的是，MPI库包含一系列专门用于处理矩阵的工具；

这些工具包括与矩阵的不同片段（列、列集或块）相关联的派生类型，以及与Cartesian拓扑结构的通信子相关的额外功能。

==矩阵乘法算法是矩阵算法的代表。==

在MPI9Matr组中，讨论了两种主要类型的**并行分布式矩阵乘法算法**：

1. 条带算法，其中计算分布在进程之间通过将矩阵划分为包含相邻行或列集的条带(bands)；
2. 块算法，在其中矩阵被划分为矩形块(blocks)。

此外，该组还包括入门任务MPI9Matr1（参见第2.9.1节），其中描述了矩阵数据存储格式，提供了必要的公式，并要求实现 最简单的非并行矩阵乘法算法。

对于每种类型的并行矩阵乘法算法，都考虑了两种不同的实现方式。

条带算法的类型:
1) 水平条带:
	在条带算法1中（任务MPI9Matr2至MPI9Matr10，第2.9.2节），仅使用==水平条带（相邻行的集合）==，不需要引入额外的数据类型来进行传输。
2) 水平和垂直条带:
	在条带算法2中（任务MPI9Matr11至MPI9Matr20，第2.9.3节），使用==水平和垂直条带==，这在某种程度上简化了算法本身的实现（因为它基于对一个矩阵的行与另一个矩阵的列进行乘法运算），但**需要使用新的数据类型以实现更有效的垂直条带传输（即相邻列的集合）**。

块算法的类型:
1) Kannan算法:  在第一个块算法变体中（算法Kannan，任务MPI9Matr21至MPI9Matr31，第2.9.4节），在==迭代计算结果矩阵片段之前，首先对块在进程间进行初始重新分配==，从而简化了后续数据传输操作。在算法的**任意阶段传输块时，都使用带有方形进程矩阵拓扑结构的辅助通信子**。
2) Fox的算法: 在第二个块算法变体中（算法Fox，任务MPI9Matr32至MPI9Matr44，第2.9.5节），==不需要特殊的初始重新分配阶段，导致每一步计算结果矩阵的块之间的传输更加复杂==。在这种传输中，建议**不仅使用具有方形进程矩阵拓扑结构的通信子，还要使用基于该通信子生成的与该矩阵的各个行和列相关的通信子**。对于每个考虑的==矩阵乘法算法，都需要定义一种新的派生数据类型==，以简化矩阵块的传输；

在Kannan算法中，建议使用MPI_Send和MPI_Recv函数来传输块，
而在Fox算法中则建议使用MPI_Alltoallw集体函数（前提是使用MPI-2库）。

# 矩阵乘法算法的阶段

在所讨论的每个矩阵乘法算法中，可以分解为**三个主要阶段**：

- **数据分发阶段：** 初始时，将原始矩阵的片段（条带或块）发送到所有进程中。

- **计算阶段：** 顺序计算最终矩阵乘积的片段，每一步都伴随着原始矩阵片段在进程之间的传输（对于Kannan算法，计算阶段之前有一个与初始块重新分配相关的初始化阶段）。

- **结果收集阶段：** 将计算得到的矩阵乘积片段传送到主进程以获取最终的矩阵。

==每个阶段都与单独的任务==（或一系列任务）相关联；
在每个任务中，初始数据是上一个阶段生成的，这样有助于简化每个阶段的开发和测试，同时也使得可以实现每个阶段的独立实现，而不需要预先开发所有先前阶段的内容。

任务系列与计算阶段相关联，这是最复杂的阶段。
在每个系列的初始任务中，需要开发最基本的计算方法，该方法在算法的第一步使用，并且在随后的任务中对该方法进行修改，以便能够在计算阶段的每一步中应用。

唯一的例外是Kannan算法，在该算法中，每一步的操作与步骤编号无关（参见MPI9Matr25任务的注释）。

另外，还提供了一些任务，这些任务需要修改每个算法的起始和结束阶段，以便在这些阶段中使用并行文件输入输出：

- **读取文件数据阶段：** 每个进程直接从包含这些矩阵的文件中获取原始矩阵的片段；
- **写入到最终文件阶段：** 每个进程将计算得到的乘积片段写入相应的结果文件部分。

此外，对于需要使用新数据类型或具有笛卡尔拓扑结构的通信器的算法，还提供了与创建相应对象（如MPI_Datatype或MPI_Comm类型）相关的额外任务。

==在所有关于矩阵算法各个阶段的实现任务以及创建**辅助对象**（新的派生类型或通信器）的任务中，需要将相应的操作整理为辅助函数。==

创建新对象的函数将在后续的算法阶段实现中使用，而与阶段本身相关的函数将用于最终任务中，最终任务要求完整实现相应的矩阵算法。

下面的表格2和表格3列出了与每个算法的不同阶段实现相关的任务编号，此外还提供了需要在这些任务中开发的函数名称。

表格2适用于带状算法（MPI9Matr2–MPI9Matr20），而表格3适用于块算法（MPI9Matr21–MPI9Matr44）。
![[Pasted image 20231224011653.png]]

![[Pasted image 20231224011715.png]]
![[Pasted image 20231224011732.png]]

# 乘法基本算法(扁平化存储->用一维数组存储!!!)
!!!Why?
//A 的大小 M*P , B 的大小 P*Q--相乘--> C 为 M*Q

// 乘法公式 : C_i_j = A_i_0 * B_0_j + A_i_1 * B_1_j + ...+ A_i_p-1 * B_p-1_j

因为A B 是这么存储的
![[Pasted image 20231224021210.png]]
然后我们看 矩阵乘法公式
它是A行向量 乘以 B列向量 ,
![[Pasted image 20231224022411.png]]
紫色的是Aim
我们选择 以A 的移动为基准 ---> 在此基础上动B

==> 故最外层是i---> 控制A的层的移动 --->(同时也对印着 C 中的位置)

===> 然后 n 控制着B 每一层的移动来找到对应的列的对应层,同时n也是A上哪固定层中的第几个元素

===> 最后用j 控制着B 对应层上面的 对应要做乘积的那个元素

e.g. C[i,j] ==> A在第i层动(但这个动到第几个是由n决定) , B在第j列动(但这个j列 要通过 n 来换层)
这个n 就是乘法公式元素式子里
从i,0  \* 0 j + i,1 \* 1,j +...+ i,p-1\* p-1,j

中间那个个首尾相接的从0~p-1 :))))))))

![[Pasted image 20231224020752.png]]
**乘法公式 : C_i_j = A_i_0 * B_0_j + A_i_1 * B_1_j + ...+ A_i_p-1 * B_p-1_j **
```C
#include "pt4.h"
#include "mpi.h"
#include <cmath>
//A 的大小 M*P , B 的大小 P*Q--相乘--> C 为 M*Q
// 乘法公式 : C_i_j = A_i_0 * B_0_j + A_i_1 * B_1_j + ...+ A_i_p-1 * B_p-1_j

  

int k;              // number of processes
int r;              // rank of the current process
int m, p, q;        // sizes of the given matrices
int *a_, *b_, *c_;  // arrays to store matrices in the master process

void Solve()

{

    Task("MPI9Matr1");

    int flag;

    MPI_Initialized(&flag);

    if (flag == 0)

        return;

    int rank, size;

    MPI_Comm_size(MPI_COMM_WORLD, &size);

    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

    k = size;

    r = rank;

  

    if(r != 0)

        return ;

    pt >> m >> p >> q;

    // 扁平化存储_> 不用二维数组!!!!!!!--->MPI的特点
    a_ = new int[m*p];
    b_ = new int[p*q];

  

    for(int i=0;i < m*p;++i){

        pt >> a_[i];

    }

    for(int i =0;i<p*q;++i){

        pt >> b_[i];

    }

  

    c_ = new int[m*q];

    for (int i=0;i<m*q;++i){

        c_[i]=0;

    }

    //储存结果的数组__>

  

    for(int i=0;i<m;++i)

        for(int j=0;j<q;++j)

            for(int n=0;n<p;++n)

                c_[i*q+j] += a_[i*p+n] * b_[n*q+j];

    for(int i=0;i<m*q;++i)

        pt<<c_[i];

    delete[] a_;

    delete[] b_;

    delete[] c_;

}
```

# 条带算法的条带分发 和 条带收集
MPI9Matr2。

在主进程中给定了数字 M、P、Q 和大小分别为 M × P 和 P × Q 的矩阵 A 和 B。

在第一个条带矩阵乘法算法的变体中，每个乘法矩阵被分成 K 个水平条带，其中 K 是进程数（随后，这些条带将分配给进程，并用于在每个进程中计算最终矩阵乘积的部分）。

矩阵 A 的条带包含 NA 行，矩阵 B 的条带包含 NB 行；NA 和 NB 的值由以下公式计算得出：NA = ceil(M/K)，NB = ceil(P/K)，其中“/”表示浮点数除法，ceil 函数执行向上取整。

**条带的定义** : 
![[Pasted image 20231224165832.png]]

==如果矩阵包含的行数不足以填满最后一个条带，那么条带将补充零行。==

**Main_process**:
在主进程中，将需要时用零行补充的原始矩阵保存在一维数组中，然后组织从这些数组到所有进程的条带传输：**将索引为 R 的条带发送到排名为 R 的进程（R = 0, 1, …, K − 1），所有 A 的条带具有大小 NA × P，所有 B 的条带具有大小 NB × Q**。
此外，为**每个进程创建用于存储矩阵乘积 C = AB 的条带 CR；每个 CR 条带的大小为 NA × Q，并用零元素填充**。
条带和原始矩阵都应按行存储在==相应大小的一维数组中==。

要传输矩阵的大小，使用集合函数 MPI_Bcast，要传输矩阵 A 和 B 的条带，使用集合函数 MPI_Scatter。
1) 广播条带数 和 每个条带的元素数(即行元素数)
2) ==因为我们条带的数目是按找秩分的==(因为广播是按秩顺序进行广播)

最后把所有条带的收集到 主进程0 ;

**真实情况下,我们完成条带分发后,在各个进程中,我们会完成矩阵运算的部分,然后再将计算完成的所有条带统合到主进程**

==所有 条带划分是最重要的, 这个只是一个习题 , 让我们明白划分和收集的意义,正常情况下会更加严肃==

将所有描述的操作封装为 Matr1ScatterData 函数（无参数）。
调用该函数后，每个进程都会获得 NA、P、NB、Q 的值，以及填充了相应矩阵 A、B、C 的一维数组。
在调用 Matr1ScatterData 函数后，在每个进程中输出获取的数据（数字 NA、P、NB、Q 以及矩阵 A、B、C 的条带）。
输入原始数据应在 Matr1ScatterData 函数中进行，输出结果应在外部 Solve 函数中执行。


提示：为了减少 MPI_Bcast 函数的调用次数，可以将所有传输的矩阵大小放入辅助数组中。

```C
#include "pt4.h"
#include "mpi.h"
#include <cmath>
int myceil(double);
int k;              // number of processes

int r;              // rank of the current process

  

int m, p, q;        // sizes of the given matrices

int na, nb;         // sizes of the matrix bands

  

int *a_, *b_, *c_;  // arrays to store matrices in the master process

int *a, *b, *c;     // arrays to store matrix bands in each process

  

int myceil(double a){

    int tmp =a;

    if(a-tmp==0)

        return tmp;

    return tmp+1;

}

  

void Matr1ScatterData() {

    if (r == 0) { //我们已将r 用栈存储

        int m;

        pt >> m >> p >> q;

        na = (int)ceil(m / (k*1.0));

        nb = (int)ceil(p / (k*1.0));

        a_ = new int[na*k*p];

        b_ = new int[nb*k*q];

        for (int i = 0; i < m*p; ++i)

            pt >> a_[i];

        for (int i = m*p; i < na*k*p; ++i)

            a_[i] = 0;

        for (int i = 0; i < p*q; ++i)

            pt >> b_[i];

        for (int i = p*q; i < nb*k*q; ++i)

            b_[i] = 0;

		MPI_Bcast(&na, 1, MPI_INT, 0, MPI_COMM_WORLD);

        MPI_Bcast(&p, 1, MPI_INT, 0, MPI_COMM_WORLD);

        MPI_Bcast(&nb, 1, MPI_INT, 0, MPI_COMM_WORLD);

        MPI_Bcast(&q, 1, MPI_INT, 0, MPI_COMM_WORLD);

        a = new int[na*p];  b = new int[nb*q]; c = new int[na*q];

        MPI_Scatter(a_, p*na, MPI_INT, a, p*na, MPI_INT, 0, MPI_COMM_WORLD);

        MPI_Scatter(b_, q*nb, MPI_INT, b, q*nb, MPI_INT, 0, MPI_COMM_WORLD);

        for (int i = 0; i < na*q; ++i)

            c[i] = 0;

    }

}

  

void Solve()

{

    Task("MPI9Matr2");

    int flag;

    MPI_Initialized(&flag);

    if (flag == 0)

        return;

    int rank, size;

    MPI_Comm_size(MPI_COMM_WORLD, &size);

    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

    k = size;

    r = rank;

    Matr1ScatterData();

    pt << na << p << nb << q;

  

    for (int i = 0; i < na*p; ++i) pt << a[i];

    for (int i = 0; i < nb*q; ++i) pt << b[i];

    for (int i = 0; i < na*q; ++i) pt << c[i];

}
```

这部分内容描述了在某些矩阵乘法算法（例如Cannon算法）的初始阶段中重新分配块的过程。==这种重新分配通常是为了优化并行计算。在MPI的上下文中，重新分配块意味着将初始数据分发到各个处理器，以便进行后续计算==。举例来说，假设要实现MPI9Matr24任务，它涉及到算法的一个阶段，在初始数据分发之后立即执行（在这个案例中是关于Cannon算法的一个子任务）

# Cannon 算法的重新分配步骤
这段文字描述了在使用块矩阵乘法的卡农(Cannon)算法中的初始块重新分配过程。

对于每个给定的进程，它包含了M0、P0、Q0的值，以及包含矩阵A、B、C相应块的一维数组（这样做的目的是确保初始数据与任务MPI9Matr23的结果相匹配）。

首先，需要在给定的进程集上**定义一个二维方形周期性拓扑结构，其阶数为K0（其中K0 × K0等于进程的数量），并保留原始进程编号的顺序**。

对于该拓扑结构得到的每一行I0（I0 = 0, …, K0 − 1），将块矩阵AR按照I0向左移动I0个位置（即在进程秩递减的方向），对于每一列J0（J0 = 0, …, K0 − 1），将块矩阵BR按照J0向上移动J0个位置（即在进程秩递减的方向）。

为了创建与二维笛卡尔拓扑相关联的通信子MPI_COMM_GRID，使用了在任务MPI9Matr22中实现的Matr3CreateCommGrid函数。

在进行循环移位时，使用了MPI_Cart_coords、MPI_Cart_shift和MPI_Sendrecv_replace等MPI函数（参见MPI9Matr22）。

将上述操作封装为Matr3Init函数（无参数）。在每个进程中输出由移位得到的AR和BR块（输入和输出数据在外部函数Solve中完成）。

```С
#include "pt4.h"
#include "mpi.h"
#include <cmath>

int k;              // number of processes

int r;              // rank of the current process

  

int m, p, q;        // sizes of the given matrices

int m0, p0, q0;     // sizes of the matrix blocks

int k0;             // order of the Cartesian grid (equal to sqrt(k))

  

int *a_, *b_, *c_;  // arrays to store matrices in the master process

int *a, *b, *c;     // arrays to store matrix blocks in each process

  

MPI_Datatype MPI_BLOCK_A; // datatype for the block of the matrix A

MPI_Datatype MPI_BLOCK_B; // datatype for the block of the matrix B

MPI_Datatype MPI_BLOCK_C; // datatype for the block of the matrix C

  

MPI_Comm MPI_COMM_GRID = MPI_COMM_NULL;

             // communicator associated with a two-dimensional Cartesian grid

void Matr3CreateCommGrid(MPI_Comm &comm) {

    int dims[] = {k0, k0}, periods[] = {1, 1};

    MPI_Cart_create(MPI_COMM_WORLD, 2, dims, periods, 0, &comm);

}

  

void Matr3Init() {

    Matr3CreateCommGrid(MPI_COMM_GRID);

    int coord[2];

    MPI_Cart_coords(MPI_COMM_GRID, r, 2, coord);

    int src, dst;

    MPI_Cart_shift(MPI_COMM_GRID, 1, -coord[0], &src, &dst);

    MPI_Sendrecv_replace(a, m0*p0, MPI_INT, dst, 0, src, 0, MPI_COMM_GRID, MPI_STATUS_IGNORE);

    MPI_Cart_shift(MPI_COMM_GRID, 0, -coord[1], &src, &dst);

    MPI_Sendrecv_replace(b, p0*q0, MPI_INT, dst, 0, src, 0, MPI_COMM_GRID, MPI_STATUS_IGNORE);

}

  

void Solve()

{

    Task("MPI9Matr24");

    int flag;

    MPI_Initialized(&flag);

    if (flag == 0)

        return;

    int rank, size;

    MPI_Comm_size(MPI_COMM_WORLD, &size);

    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

    k = size;

    r = rank;

    k0 = (int)floor(sqrt((double)k) + 0.1);

  

    pt >> m0 >> p0 >> q0;

    a = new int[m0*p0]; b = new int[p0*q0]; c = new int[m0*q0];

    for (int i = 0; i < m0*p0; i++) pt >> a[i];

    for (int i = 0; i < p0*q0; i++) pt >> b[i];

    for (int i = 0; i < m0*q0; i++) pt >> c[i];

  

    Matr3Init();

    for (int i = 0; i < m0*p0; ++i) pt << a[i];

    for (int i = 0; i < p0*q0; ++i) pt << b[i];

}
```

# 结果收集过程
结果收集阶段：基于文件输出的实现示例 在任何矩阵算法的最终阶段中，需要将来自不同进程的结果片段合并成最终的矩阵乘积 C。
这样的合并结果可能是一个数组，在主进程中包含了矩阵 C 的所有元素（可以将其输出到屏幕上或保存在文件中）。

在使用 MPI 2.0 标准的实现中，可以通过直接将矩阵 C 的各个片段写入二进制文件来完成这一阶段，利用此标准提供的并行文件输入输出工具（参见1.3.2-1.3.3）。这样可以避免与将结果数据重新发送到主进程相关的额外操作。以 MPI9Matr19 任务为例，该任务要求将使用第二种条带算法获得的矩阵乘积保存到结果文件中

题目:

在每个进程中，给定了数字 NA、NB，以及填充有 CR 条带的一维数组（大小为（NA·K）×NB），这些条带是通过执行 K 步条带矩阵乘法算法得到的（参见 MPI9Matr15）。

此外，在主进程中给出了数字 M（结果矩阵的行数）以及存储该乘积的文件名。此外还知道结果矩阵的列数 Q 是进程数的倍数（因此等于 NB·K）。

将数字 M 和文件名广播到所有进程中（使用 MPI_Bcast 函数），然后将包含在 CR 条带中的所有矩阵乘积片段写入结果文件。最终的结果文件将包含大小为 M×Q 的矩阵 C。

在文件写入时，使用 MPI_File_set_view 函数设置适当的数据视图，并使用在 MPI9Matr11 中定义的新文件类型 MPI_BAND_C，该类型由 Matr2CreateTypeBand 函数创建，然后使用 MPI_File_write_all 的集合功能进行文件写入。

将读取文件名、M 值和文件名的传输，以及将条带写入文件的所有操作，整理成 Matr2GatherFile 函数形式（除文件名之外的所有原始数据读取应在外部 Solve 函数中进行）。

提示：在写入结果文件时，需要考虑到 CR 条带可能包含不属于所得乘积的结尾零行（传输值 M 到所有进程就是为了检查这种情况）。

```C

```

题目 22:

在每个过程中给出整数M0、P0和大小为M0×P0的矩阵a。过程的数量K是一个完美的平方：K=K0·K0。在每个过程中将矩阵A输入到尺寸为M0·P0的一维数组中，并使用MPI_Cart_create函数创建一个名为MPI_COMM_GRID的新通信器。MPI_COMM_GRID通信器将所有进程的笛卡尔拓扑定义为二维周期性K0×K0网格（进程的列不应重新排序）。

将MPI_COMM_GRID通讯器的创建包括在Matr3CreateCommGrid（COMM）函数中，该函数具有MPI_COMM类型的输出参数COMM。使用此通信器的MPI_Cart_coords函数，输出每个进程中的进程坐标（I0，J0）。

使用MPI_Cart_shift和MPI_Sandrecv_replace函数，将每个网格行I0的所有进程中给定的矩阵a循环移位I0个位置（即，按进程的降序）。输出每个过程中接收到的矩阵。
![[Pasted image 20240103195825.png]]
![[Pasted image 20240103195843.png]]
![[Pasted image 20240103195752.png]]