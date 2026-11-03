# Phase 2 — Reflection: where does parallelism stop helping?

Colab T4 runtime. CPU (`lscpu`): Intel Xeon @ 2.00GHz, 2 vCPUs = 1 physical core with 2 hyperthreads. Model: 4,935,680 parameters.


## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->573.474<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.377<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->506712.7<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->12766.6<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4586<!--/answer--> |

**The thread sweep.**

<!--answer:thread_sweep-->
1 thread: 2781.926 ms, 2 threads: 2420.185 ms, 4 threads: 2616.926 ms, 8 threads: 2803.087 ms. Fastest: 2 threads.
<!--/answer-->



## 2. Intra-node parallelism, three ways

<!--answer:vectorization_why-->
The naive version is a Python loop: every multiply-add goes through the interpreter, one float at a time. The vectorized version calls OpenBLAS, compiled code using AVX2/FMA SIMD instructions that handle several floats per instruction. 573.474 ms → 1.377 ms (416.47×).
<!--/answer-->

<!--answer:threading_plateau-->
Fastest at 2 threads (2420.185 ms), then slower: 2616.926 ms at 4 and 2803.087 ms at 8, which is worse than 1 thread (2781.926 ms). Colab has only 2 vCPUs, which are 2 hyperthreads of 1 physical core, so the gain is small (1.15×). Past 2, the threads fight for the same core, have to split and sync every matrix multiply, and share the cache. Coordination costs more than it saves.
<!--/answer-->

<!--answer:gpu_why-->
A training step is mostly big matrix multiplies: the same operation on lots of independent data, which suits the T4's thousands of CUDA cores. 506712.7 ms → 12766.6 ms (39.7×). A workload that would not benefit is sequential work, such as generating one token at a time with batch 1, where kernel-launch and copy overhead dominate.
<!--/answer-->



## 3. The question

<!--answer:parallelism_limit-->
Each experiment hit a different limit. Vectorization added no hardware; it removed Python overhead on one core (416.47×). The threads hit hardware I don't have: one physical core, so 2 threads only gave 2781.926 → 2420.185 ms. Past that, coordination costs more than it saves (8 threads: 2803.087 ms, slower than 1), and the serial parts (Python, batching) cap the gain too (Amdahl). The GPU gave 39.7× but was limited by data movement: the model is small (5M params), so kernel launches and host→device copies keep the T4 partly idle.
<!--/answer-->

<!--answer:next_experiment-->
A GPU batch-size sweep (16, 64, 256), timing ms per sample. If it keeps dropping, the GPU was starved and 39.7× understates it. If it stays flat, the GPU was already compute-bound.
<!--/answer-->
