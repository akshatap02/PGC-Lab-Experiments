# Experiment 2: Multithreading with Pthreads and OpenMP

Create and manage threads, divide work between them, see a race condition and fix it,
then measure how run time changes with the number of threads.

## Programs (`src/`)

Part A, Pthreads
| File | What it shows |
|---|---|
| `thread1.c` | create one thread, `pthread_create` / `pthread_join` |
| `thread2.c` | create 4 threads, output order is not guaranteed |
| `thread_sum.c` | split an array sum between 4 threads |
| `race.c` | race condition on a shared counter |
| `mutex.c` | the same counter fixed with a mutex |
| `pthread_perf.c` | timing with 1 to 16 threads |

Part B, OpenMP
| File | What it shows |
|---|---|
| `omp1.c` | parallel region, thread id and thread count |
| `omp_sum.c` | `parallel for` with `reduction` |
| `omp_race.c` | race condition in OpenMP |
| `omp_critical.c` | fixed with `critical` |
| `omp_barrier.c` | `barrier` between two stages |
| `omp_perf.c` | timing with 1 to 16 threads |

Part C, baseline
| File | What it shows |
|---|---|
| `sequential.c` | single thread sum of 10^9 terms, used as the baseline |

## Build and run

```bash
# Pthreads
gcc thread1.c -o thread1 -pthread && ./thread1

# OpenMP
gcc omp1.c -o omp1 -fopenmp && ./omp1

# performance (asks for the number of threads)
gcc sequential.c -o sequential && ./sequential
gcc pthread_perf.c -o pthread_perf -pthread && ./pthread_perf
gcc omp_perf.c -o omp_perf -fopenmp && ./omp_perf
```

## Race condition

Four threads each add 1 to a shared counter 100000 times, so the answer should be 400000.
Without protection the printed value is lower and changes from run to run (my three Pthreads runs gave 142456, 112586, 114242), because two threads can read the same
old value and both write back the same new one. A mutex (Pthreads) or `#pragma omp critical` (OpenMP) makes it 400000 every time.

## Performance results

Run on a MacBook with an Apple M4 Pro (12 cores), macOS, `cc` (clang) with libomp. Raw output is in [`logs/`](logs).
With 16 threads there are more threads than cores, so that row is oversubscribed.

Sequential baseline: 0.837061 s (average of 5 runs: 0.861, 0.835, 0.830, 0.827, 0.832).

| Threads | Pthreads (s) | OpenMP (s) |
|---|---|---|
| 1 | 0.862403 | 0.893458 |
| 2 | 0.426579 | 0.448585 |
| 4 | 0.227142 | 0.231630 |
| 6 | 0.157374 | 0.158595 |
| 16 | 0.118025 | 0.119778 |

Speedup = sequential time / parallel time. Efficiency = speedup / threads.

| Threads | Pthreads speedup | OpenMP speedup | Pthreads efficiency | OpenMP efficiency |
|---|---|---|---|---|
| 1 | 0.971x | 0.937x | 97.06% | 93.69% |
| 2 | 1.962x | 1.866x | 98.11% | 93.30% |
| 4 | 3.685x | 3.614x | 92.13% | 90.34% |
| 6 | 5.319x | 5.278x | 88.65% | 87.97% |
| 16 | 7.092x | 6.988x | 44.33% | 43.68% |

![execution time](graphs/execution_time.png)
![speedup](graphs/speedup.png)
![efficiency](graphs/efficiency.png)

## Pthreads vs OpenMP

| | Pthreads | OpenMP |
|---|---|---|
| create threads | `pthread_create()` | `#pragma omp parallel` |
| wait | `pthread_join()` | implicit at the end of the region |
| split work | done by hand | `parallel for` |
| protect shared data | mutex | `critical` |
| sync | join | `barrier` |
| combine results | done by hand | `reduction` |

## Conclusion

Both libraries cut the run time a lot as threads were added, with almost the same numbers.
Speedup is not linear: at 16 threads it is about 7x, not 16x. This machine has 12 cores, so 16 threads share them, and there is also
thread management, scheduling, memory access and the part of the program that is not parallel. Shared data needs a mutex or `critical`,
otherwise the result is wrong.
