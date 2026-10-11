# Lab Report: Parallel Matrix Multiplication

## Abstract

A 4000 x 4000 matrix multiplication is implemented four ways: sequential, OpenMP (shared memory), MPI (distributed memory on
4 virtual machines) and CUDA (GPU). The same program logic and check value (`C[0][0] = 4000.00`) are used throughout so the run times can be compared.
A second experiment covers Pthreads and OpenMP basics: thread creation, race conditions, synchronization, and scaling with thread count.

## Objectives

- Understand the difference between shared-memory, distributed-memory and GPU parallelism.
- Measure the speedup of each model against a sequential baseline.
- See why communication cost and memory access patterns limit speedup.
- Create threads, divide work, show a race condition and fix it with a mutex / `critical`.

## Setup

- WSL2 Ubuntu, GCC with `-O2`, OpenMP via `-fopenmp` (machine with 16 logical CPUs, 8 threads used)
- Four Ubuntu VMs in VMware (master, worker1, worker2, worker3) with Open MPI and passwordless SSH
- NVIDIA GPU with CUDA toolkit for Part D

## Results (experiment 1)

| Implementation | Time | Speedup | Source |
|---|---|---|---|
| Sequential | 321.416197 s | 1.00x | measured |
| OpenMP, 8 threads | 111.864688 s | 2.87x | measured |
| MPI, 4 processes | 92.979510 s | 2.63x* | reference |
| CUDA | 0.165004 s | 1479.48x* | reference |

\* uses the reference sequential time of 244.12 s.

Source code is in [src](src), screenshots and the chart are in [images](images).

## Discussion

OpenMP reduced the time from 321.4 s to 111.9 s, a 2.87x speedup on 8 threads. The gain is far below 8x because the inner loop walks matrix B
column by column, so the threads wait on memory more than they compute. MPI divides the rows of A across four processes but also has to
broadcast all of B and gather C over the network, which adds communication time that OpenMP does not pay. CUDA assigns one GPU thread to each
output element (62,500 blocks of 256 threads), which suits this problem, and it was the fastest by orders of magnitude.

In experiment 2, four threads incrementing one shared counter 100000 times each gave a value below 400000 because updates were lost.
A mutex (Pthreads) and `#pragma omp critical` (OpenMP) fixed it. Run time fell as threads were added but not linearly, because of
thread management, scheduling and memory overhead.

## Conclusion

Parallelism gave a clear speedup in every model, but the amount depended on how the work and data were shared. Shared memory has the least
overhead, distributed memory pays for communication, and the GPU gives the largest gain when the work maps to many small independent threads.
Shared data must always be protected when threads write to it.
