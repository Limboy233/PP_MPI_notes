在MPI中，您可以使用不同的方法创建新的通信子（communucator），以便更有效地执行数据传输和协同操作。通信子是MPI中的一个关键概念，它代表了一组进程，允许它们在并行应用程序中进行通信和协作。在下面的内容中，我们将介绍如何使用MPI创建新的通信子。

# 一.复制
MPI中创建新通信子的最简单方法之一是**通过复制现有的通信子**。

这是使用`MPI_Comm_dup`函数实现的，每个与通信子相关联的进程都必须调用它。
```C
MPI_Comm_dup(MPI_Comm comm, MPI_Comm* newcomm)
```
新的通信子将包含与原始通信子相同的进程组，并具有与原始通信子相同的其他特性（包括==虚拟拓扑==，如1.2.8中所述）。

值得强调的是，使用一个通信子发送的消息不会影响使用另一个通信子发送的消息。

它们被发送到“不同的通道”。

**常用场景**:
通常在附加的并行库中创建MPI_COMM_WORLD的副本，并用于在进程之间传递这些库的正常运行所需的内部信息。
用户库不能访问这些副本，因此无法影响使用它们执行的数据传输。

重要: ==赋值操作只会建立通信子的引用!!!!!==--->和Python 很像
!!!请注意，常规的赋值操作，如`MPI_Comm newcomm = comm;`不会创建通信子comm的副本，它只会创建与同一通信子相关联的描述符的副本。

## 通信子比较
要比较通信子，MPI提供了`MPI_Comm_compare`函数。
```C
MPI_Comm_compare(MPI_Comm comm1, MPI_Comm comm2, int* result)
```

1) 当比较不同的描述符，它们与同一通信子相关联时，此函数将返回MPI_IDENT的值。(用赋值操作创建同一个通信子的引用时)

2) 如果比较包含相同进程集的不同通信子，并且这些进程以相同的方式排序，那么将返回MPI_CONGRUENT的值（这将在比较原始通信子和使用`MPI_Comm_dup`创建的副本时返回）。

3) 如果两个通信子包含相同的进程集，但进程的排序不同，那么将返回MPI_SIMILAR的值。

4) 如果通信子包含不同的进程集，则返回MPI_UNEQUAL的值。

==这种复制通信子的方法非常有用，因为它允许您轻松地创建与现有通信子相同的通信子，但它们是独立的，互不干扰。这对于在不同的通信子中执行不同的通信操作非常有用。==

# 二.组
创建新通信子的第二种方法涉及
在另一个通信子内部预先定义新的进程组（group）。

通过使用该在通信子`comm`内的新组`group`，您可以创建一个新的通信子`newcomm`，该通信子仅包含`group`中的进程。这是通过使用`MPI_Comm_create`函数实现的。
```C
MPI_Comm_create(MPI_Comm comm, MPI_Group group, MPI_Comm* newcomm)
```
所有包含在通信子`comm`中的进程都必须调用这个函数。对于不包含在指定组`group`中的那些进程，`newcomm`参数将返回`MPI_COMM_NULL`的值。

这种方法非常有用，因为它允许您从现有通信子中创建新的通信子，其中包含特定的进程子集。

新的通信子将允许这个子集的进程进行通信，而不受其他进程的干扰。

在MPI-2标准中，MPI_Comm_create函数的功能得到了扩展。现在，您可以在一次函数调用中创建多个新的与原始通信子关联的、不相交进程组的通信子。为此，只需在每个这样的进程组中的进程中调用MPI_Comm_create函数，参数`group`应设置为该特定组（但需要在原始通信子`comm`中的所有进程中执行MPI_Comm_create函数调用）。这一扩展使MPI_Comm_create函数更加灵活，便于创建多个新的与原始通信子相关的通信子，这些通信子分别与不同的进程子组相关，这在不同的并行计算场景中非常有用。

这一扩展还使MPI_Comm_create函数与MPI_Comm_split函数更加相似，后者也允许创建与原始通信子中的不同子进程组相关的通信子




要使用这种方法，
1. 您需要首先定义一个新的组`group`，包含您想要包括在新通信子中的进程。
2. 然后，将这个组`group`与现有通信子`comm`一起传递给`MPI_Comm_create`函数，以创建新的通信子`newcomm`。

注意，新通信子`newcomm`将包括在`group`中的进程，但不包括在组之外的进程。
所以，只有`group`中的进程才能在新通信子中相互通信。

MPI库提供了许多用于处理进程组（MPI_Group类型对象）的函数以及两个常量：
1. MPI_GROUP_EMPTY（表示空组，即不包含任何进程的组）
2. MPI_GROUP_NULL（表示错误组）。

## 组的建立
```C
MPI_Comm_group(MPI_Comm comm, MPI_Group* group).
```

使用`MPI_Comm_group`函数，您可以创建包含通信子`comm`中的所有进程的组。----->==组是一组通信子的子集==
**但是我们第一次用MPI_Comm_group 建立的组 则是先包含所有进程,然后通过一些组计算算子得到我们想要的子集组,最后由这个子集组建立通信子Communicator**

对于组，您可以使用以下函数来确定组的大小，即组中包含的进程数量（`MPI_Group_size`函数），以及当前进程在给定组中的排名（`MPI_Group_rank`函数）。
```C
MPI_Group_size(MPI_Group group, int* size)

MPI_Group_rank(MPI_Group group, int* rank)
```

如果当前进程不属于指定的组，则`MPI_Group_rank`函数的`rank`参数将返回值MPI_UNDEFINED。

```C
MPI_Group_translate_ranks(MPI_Group group1, int n, int* ranks1, MPI_Group group2, int* ranks2)
```
还有一个`MPI_Group_translate_ranks`函数，
`MPI_Group_translate_ranks`是一个用于将一个组中的进程的排名映射到另一个组中的排名的MPI函数。这对于在不同组中的进程之间建立关联非常有用，特别是在多个通信子之间进行通信时。
**函数的功能是将`ranks1`数组中的排名映射到`group2`中，并将结果存储在`ranks2`数组中。** 
---> ==即另一个Group  可以拿到我这个Group的排名==

如果`ranks1`中的某个排名无法在`group2`中找到，那么对应位置的`ranks2`中将包含值`MPI_UNDEFINED`。
这对于在不同组中的进程之间建立关联很有用。
## 组比较
组和通信子都可以进行比较。使用`MPI_Group_compare`函数，您可以比较两个组，它将返回三个可能的结果值之一：
```C
MPI_Group_compare(MPI_Group group1, MPI_Group group2, int* result)
```
1. MPI_IDENT表示两个组包含相同的进程集合，且这些进程集合的顺序相同；
2. MPI_SIMILAR表示两个组包含相同的进程集合，但进程在组中的顺序不同；
3. MPI_UNEQUAL表示两个组包含不同的进程集合。

## 组的子集
如果您有一个组`group`，您可以创建一个新的组，其中只包含原始组的一部分进程。
```C
MPI_Group_incl(MPI_Group group, int n, int* ranks, MPI_Group* newgroup)

MPI_Group_excl(MPI_Group group, int n, int* ranks, MPI_Group* newgroup)

```
为此，有两个函数可供使用：`MPI_Group_incl`和`MPI_Group_excl`，它们具有==相同的参数列表==。


1) include:
`MPI_Group_incl`函数将在新组中包括在数组`ranks`中指定的排名对应的进程，新组将包含`n`个进程。

2) exclude: 
`MPI_Group_excl`函数将排除在数组`ranks`中指定的排名对应的进程, n为要排除进程的数量(**即ranks 数组大小**) 。如果参数`n`为0，则将返回一个空组，与常量MPI_GROUP_EMPTY相等

MPI 提供了两个非常方便的函数 `MPI_Group_range_incl` 和 `MPI_Group_range_excl`，用于操作组中的一组连续的排名，当排名遵循规则范围时，这些函数非常有用。以下是详细说明：
```C
MPI_Group_range_incl(MPI Group group, int n, int ranges[][3], MPI Group *newgroup)

MPI_Group_range_excl(MPI Group group, int n, int ranges[][3], MPI Group *newgroup)
```
和上面一样,他们也有一样的参数列表..
1. **MPI_Group_range_incl**：
    
    - `MPI_Group_range_incl(MPI_Group group, int n, int ranges[][3], MPI_Group *newgroup)`：该函数用于创建一个新的组 `newgroup`，其中包括来自原始组 `group` 中指定范围的连续排名。
    - 参数说明：
        - `group`：原始组，您要从中选择排名。
        - `n`：指定 范围的数量。
        - `ranges`：一个大小为 `n` 的数组，其中每个元素都是一个三元组，表示一个范围 `(first, last, step)`。这些范围定义了要包括在新组中的排名。范围是从 `first` 开始，以 `step` 为步幅递增，一直到 `last`。
        - `newgroup`：新创建的组，其中包括了符合指定范围的排名。
2. **MPI_Group_range_excl**：
    
    - `MPI_Group_range_excl(MPI_Group group, int n, int ranges[][3], MPI_Group *newgroup)`：此函数用于创建一个新的组 `newgroup`，其中排名来自原始组 `group` 但不在指定范围内。
    - 参数说明：
        - `group`：原始组，您要从中排除排名。
        - `n`：指定范围的数量。
        - `ranges`：一个大小为 `n` 的数组，其中每个元素都是一个三元组，表示一个范围 `(first, last, step)`。这些范围定义了要在新组中排除的排名。范围是从 `first` 开始，以 `step` 为步幅递增，一直到 `last`。
        - `newgroup`：新创建的组，其中包括原始组中未在指定范围内的排名。

这些函数非常有用，因为它们使您能够轻松创建新组，该组包括或排除特定范围内的排名，而无需显式指定单个排名。这对于处理连续的排名范围非常方便，可以更轻松地组织和管理并行计算中的进程组。

## 组的运算

在MPI中，您可以使用以下三个函数进行不同组的操作，即组合（union）、交集（intersection）和差异（difference）：

1. **MPI_Group_union**：
    
    - `MPI_Group_union(MPI_Group group1, MPI_Group group2, MPI_Group* newgroup)`：此函数用于将两个组合并成一个新组。新组包含两个原始组中的所有进程，以与它们在原始组中的顺序一样。如果一个进程在两个原始组中都存在，它只会出现在新组中一次。这是一种组合操作。
2. **MPI_Group_intersection**：
    
    - `MPI_Group_intersection(MPI_Group group1, MPI_Group group2, MPI_Group* newgroup)`：此函数用于创建一个新组，其中只包含在两个原始组中都存在的进程。新组中的进程的顺序与它们在原始组中的顺序相同。这是一个交集操作。
3. **MPI_Group_difference**：
    
    - `MPI_Group_difference(MPI_Group group1, MPI_Group group2, MPI_Group* newgroup)`：此函数用于创建一个新组，其中只包含在第一个原始组中存在，但不在第二个原始组中存在的进程。新组中的进程的顺序与它们在第一个原始组中的顺序相同。这是一个差异操作。

这些组操作允许您轻松地将不同的组合并、查找它们的交集或计算它们的差异。这对于根据不同的标准组织和管理进程组非常有用，例如，根据其角色、功能或其他特性。如果新组为空，即没有共同进程，则在 `newgroup` 参数中返回值 `MPI_GROUP_EMPTY`。

## 组的销毁与通讯器销毁
```C
MPI_Group_free(MPI_Group* group)
```
最后，您可以使用 `MPI_Group_free` 函数来释放已创建的组，以释放与它们关联的资源，将参数 `group` 设置为 `MPI_GROUP_NULL`，以表明组不再存在。这是一种良好的实践，以确保不会出现资源泄漏。

同样，使用 `MPI_Comm_free` 函数来释放通信器。
```C
MPI_Comm_free(MPI_Comm* comm).
```

# 三.(第三种建立通讯子的方法)MPI_Comm_split


`MPI_Comm_split` 是一种在 MPI 中用于将原始通信组分割成具有不重叠进程组的一组通信组的函数。下面是一个使用示例：
```C
MPI_Comm_split(MPI_Comm comm, int color, int key, MPI_Comm* newcomm)
```

假设您有原始通信组 `MPI_COMM_WORLD`，其中包括所有进程。您想要基于某些标准创建进程组。例如，您想要创建两个通信组：一个包含所有偶数排名的进程，另一个包含所有奇数排名的进程。

您可以使用 `MPI_Comm_split` 完成此任务。以下是如何执行的示例：
```C
#include <mpi.h>
int main(int argc, char *argv[]) {
    MPI_Init(&argc, &argv);

    int world_rank, world_size;
    MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);

    int color = world_rank % 2; // 定义分割通信组的标准

    MPI_Comm new_comm;
    MPI_Comm_split(MPI_COMM_WORLD, color, world_rank, &new_comm);

    int new_rank, new_size;
    MPI_Comm_rank(new_comm, &new_rank);
    MPI_Comm_size(new_comm, &new_size);

    printf("全局排名/大小: %d/%d, 新排名/大小: %d/%d\n", world_rank, world_size, new_rank, new_size);

    MPI_Comm_free(&new_comm);
    MPI_Finalize();
    return 0;
}
```

在这个示例中，我们使用 `world_rank` 除以 2 的余数来确定每个进程的颜色 (`color`)。拥有相同颜色的进程（偶数或奇数）将位于同一通信组 (`new_comm`) 中，具有不同颜色的进程将位于另一个通信组中。我们还显示了两个通信组的排名和大小以进行演示。

`MPI_Comm_split` 函数基于指定的标准（在此示例中为颜色）创建了一个新的通信组。您可以使用此方法根据您的任务创建包含不同进程集的通信组。

请注意，在使用新通信组后，您需要使用 `MPI_Comm_free` 释放它。


