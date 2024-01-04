1.![[Pasted image 20231202213302.png]]

Code:
```C
#include "pt4.h"

#include "omp.h"

  

double nonparallel_f(double x, int n) {

    double res = 0;

    for (int i = 1; i <= n; ++i) {

        double t = 0;

        for (int j = i; j <= 2 * n; ++j) {

            t += (j + pow(x + j, 1. / 3)) / (2 * i * j - 1);

        }

        res += 1 / t;

    }

    return res;

}

  

double parallel_f(double x, int n) {

    double res = 0;

    #pragma omp parallel num_threads(2)

    {

        int num = omp_get_thread_num();

        int num_threads = omp_get_num_threads();

        int num_procs = omp_get_num_procs();

        int count = 0;

        if (num == 0) {

            Show("\nnum_procs: ");

            Show(num_procs);

            Show("\nnum_threads: ");

            Show(num_threads);

        }

        double time = omp_get_wtime();

        for (int i = num; i <= n; i += 2) {

            double t = 0;

            for (int j = i; j <= 2 * n; ++j) {

                t += (j + pow(x + j, 1. / 3)) / (2 * i * j - 1);

                count += 1;

            }

            res += 1 / t;

        }

        Show("\nthread_num:");

        Show(num);

        Show("Count:");

        Show(count);

        Show("Thread time:");

        Show(omp_get_wtime() - time);

    }

    return res;

}

  

void Solve()

{

    Task("OMPBegin4");

    double x;

    int n;

    pt >> x >> n;

    double start = omp_get_wtime();

    double res = nonparallel_f(x, n);

    Show("Non-parallel time:");

    double np_time = omp_get_wtime() - start;

    Show(np_time);

    pt << res;

    pt >> x >> n;

    start = omp_get_wtime();

    res = parallel_f(x, n);

    double p_time = omp_get_wtime() - start;

    Show("\nTotal parallel time:");

    Show(p_time);

    pt << res;

    Show("\nRate:");

    Show(np_time / p_time);

}
```

在这里OMP 使用注释修饰该函数==\#pragma omp parallel num_threads(4)==

它表示建立2个thread来同时处理这个函数,最后对变量result 进行约归加法运算( reduction(+:result))
for (int i = num; i <= n; i += 2) 因此我们最外层的这个求和循环 从进程序号num开始,


这段代码使用了 OpenMP（Open Multi-Processing）来并行化计算，以提高程序的执行效率。以下是代码中使用的 OpenMP 功能以及其原因：

1. **`#pragma omp parallel`：**
   - 这是 OpenMP 中最常见的指令之一，用于创建并行区域。
   - 原因：通过并行区域，程序可以利用多个线程并行执行代码块，每个线程处理一部分工作，从而提高整体计算速度。

2. **`omp_get_thread_num()` 和 `omp_get_num_threads()`：**
   - 这些函数分别用于获取当前线程编号和并行区域内的线程总数。
   - 原因：可以在代码中根据线程编号执行不同的任务，或者根据线程总数优化并行区域的工作量。

3. **`omp_get_num_procs()`：**
   - 此函数==用于获取系统中的处理器核心数量==。
   - 原因：了解系统中可用的处理器核心数，以便调整并行化任务的分配和优化。

4. **`omp_get_wtime()`：**
   - 该函数用于获取当前时间，通常用于计算程序段的执行时间。
   - 原因：通过测量执行时间，可以评估不同部分的性能，并确定是否有效利用了并行计算。

5. **`#pragma omp for`：**
   - 在并行区域内用于标记一个 for 循环，将其并行化执行。
   - 代码中没有直接使用此指令，但在 `parallel_f` 函数中通过线程编号 `num` 控制了循环的范围。
   - 原因：此类循环并行化可以通过多个线程同时执行迭代，加速循环内计算的执行。

6. **并行化的循环结构：**
   - `parallel_f` 函数中的循环结构根据线程编号 `num` 来确定每个线程计算的范围。
   - 原因：这样的设计允许多个线程同时处理不同的迭代，提高了计算效率。

这些 OpenMP 功能的使用旨在利用多线程的并行性，以并行执行循环计算部分，从而加快整体计算速度。通过并行化部分代码，程序可以利用多个处理器核心，并在多个线程之间分配计算任务，以更快地完成任务。