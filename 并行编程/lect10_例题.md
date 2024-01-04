# 一. 矩阵分配
三个关键点:
1. 一维数组存储
2. 定义新的含block的数据类型
3. 利用新的数据类型传输

## Band algorithm 2 (horizontal and vertical bands)

MPI9Matr11°. Integers _P_ and _Q_ are given in each process; in addition, a matrix _B_ of the size _P_ × _Q_ is given in the master process. The number _Q_ is a multiple of the number of processes _K_. Input the matrix _B_ into a one-dimensional array of the size _P_·_Q_ in the master process and define a new datatype named MPI_BAND_B that contains a vertical band of the matrix _B_. The width of the vertical band should be equal to _N__B_ = _Q_/_K_ columns. When defining the MPI_BAND_B datatype, use the MPI_Type_vector and MPI_Type_commit functions.

Include this definition in a Matr2CreateTypeBand(p, n, q, t) function with the input integer parameters p, n, q and the output parameter t of the MPI_Datatype type; the parameters p and n determine the size of the vertical band (the number of its rows and columns), and the parameter q determines the number of columns of the matrix from which this band is extracted.

Using the MPI_BAND_B datatype, send to each process (including the master process) the corresponding band of the matrix _B_ in the ascending order of ranks of receiving processes. Sending should be performed using the MPI_Send and MPI_Recv functions; the bands should be stored in one-dimensional arrays of the size _P_·_N__B_. Output the received band in each process.

Remark. In the MPICH2 version 1.3, the MPI_Send function call is erroneous if the sending and receiving processes are the same. You may use the MPI_Sendrecv function to send a band to the master process. You may also fill a band in the master process without using tools of the MPI library.
![[Pasted image 20231229225906.png]]

```C++
#include "pt4.h"

  

#include "mpi.h"

  

#include <cmath>

  

int k;              // number of processes

int r;              // rank of the current process

  

int m, p, q;        // sizes of the given matrices

int na, nb;         // sizes of the matrix bands

  

int *a_, *b_, *c_;  // arrays to store matrices in the master process

int *a, *b, *c;     // arrays to store matrix bands in each process

  

MPI_Datatype MPI_COLS; // datatype for the band of the matrix B

  
  

//creat the struction of MPI_Band_B

void Matr2CreateTypeBand(int p,int n,int q,MPI_Datatype* t){

    //Creat the type

    MPI_Type_vector(p,n,q,MPI_INT,t);

    MPI_Type_commit(t);

}

  

//like:

//* * * * null null ......

//number_band  

//         allCount : q

  

void Solve()

{

    Task("MPI9Matr11");

    int flag;

    MPI_Initialized(&flag);

    if (flag == 0)

        return;

    int rank, size;

    MPI_Comm_size(MPI_COMM_WORLD, &size);

    MPI_Comm_rank(MPI_COMM_WORLD, &rank);

    k = size;

    r = rank;

  

    int P=0;

    int Q=0;pt >> P;pt >> Q;

  

    int B[P*Q];

    if(rank==0)

        for(int i=0;i<P*Q;i++){

            pt >>B[i];

        }

  

    int N_b =(int) Q/size;

    int Band[P * N_b];

  

    ShowLine(N_b);

  

    MPI_Datatype MPI_BAND_B;

    Matr2CreateTypeBand(P,N_b,Q,&MPI_BAND_B);

    if(rank == 0){

        //MPI_Send(B+0*N_b,1,MPI_BAND_B,0,0,MPI_COMM_WORLD);

        //MPI_Recv(Band,P*N_b,MPI_INT,0,0,MPI_COMM_WORLD,MPI_STATUS_IGNORE);

  

        for(int r=0;r<size;r++){

            MPI_Send(B+r*N_b,1,MPI_BAND_B,r,0,MPI_COMM_WORLD);

        }

        MPI_Recv(Band,P*N_b,MPI_INT,0,0,MPI_COMM_WORLD,MPI_STATUS_IGNORE);

    }else{

        MPI_Recv(Band,P*N_b,MPI_INT,0,0,MPI_COMM_WORLD,MPI_STATUS_IGNORE);

    }

  

    for(int i=0;i<P;i++)//Traverse the line

        for(int j=0;j<N_b;j++)

            pt << Band[i*N_b+j];

  

  

}
```