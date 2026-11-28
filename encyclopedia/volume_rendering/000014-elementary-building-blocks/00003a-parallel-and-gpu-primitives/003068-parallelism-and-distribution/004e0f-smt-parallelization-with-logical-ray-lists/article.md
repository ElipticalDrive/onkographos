# SMT Parallelization with Logical Ray Lists

Also called: Logical Ray Lists, SMT Parallelization

## Overview
This method is a parallelization strategy for Simultaneous Multithreading (SMT) architectures where multiple threads operate within the same processing block to share cache resources [S1, unofficial]. It solves the problem of race conditions and cache trashing that occur when multiple threads attempt to modify shared ray lists or operate on non-neighboring data [S1, unofficial]. Unlike [Parallel Image Compositing with Overlapping](3554-parallel-image-compositing-with-overlapping.md) or [Ray Compaction](4039-ray-compaction.md), which often rely on global synchronization or compacted data structures, this method uses a hierarchical distribution of ray lists—one per logical CPU per block—to eliminate the need for expensive synchronization primitives like mutexes [S1, unofficial]. It builds on [Parallelism and Distribution](6265-parallelism-and-distribution.md) by specifically addressing the hardware constraints of SMT where logical processors share physical resources [S1, unofficial].

## How it works
**Data Structures:**
*   **Block List:** A list of blocks (spatial partitions of the volume) to be processed [S1, unofficial].
*   **Ray List:** A list of rays currently inside a specific block, waiting to be processed [S1, unofficial].
*   **Per-Block Ray Lists:** Each block maintains $N_{phys} \times N_{log}$ distinct ray lists, indexed as `ray_lists[phys_id][log_id]` [S1, unofficial].
    *   `phys_id`: Identifier of the physical CPU currently processing the block.
    *   `log_id`: Identifier of the logical CPU (thread) within that physical CPU.

**Parameters:**
*   $N_{phys}$: Number of physical CPUs (`count_physical`) [S1, unofficial].
*   $N_{log}$: Number of logical CPUs per physical CPU, i.e., the number of threads that will simultaneously process a block (`count_logical`) [S1, unofficial].
*   Total ray lists per block: $N_{phys} \times N_{log}$ [S1, unofficial].

**Steps:**
1.  Distribute blocks among physical processors [S1, unofficial].
2.  Within a block, multiple threads (each on a logical CPU) simultaneously process rays [S1, unofficial].
3.  Instead of one ray list per physical CPU, use $N_{log}$ lists per physical CPU [S1, unofficial].
4.  Each block has $N_{phys} \times N_{log}$ ray lists [S1, unofficial].
5.  **Workflow (e.g., $N_{phys}=2, N_{log}=2$, threads T0–T6):**
    *   T0 preprocesses, assigns rays to entry blocks, and chooses lists for the eight principal viewing directions [S1, unofficial].
    *   T0 partitions the block lists and sends them to T1 and T2 (slave managers) [S1, unofficial].
    *   T0 sleeps until the slaves finish their current assignments [S1, unofficial].
    *   T1 sends blocks to T3 and T4; T2 sends blocks to T5 and T6 [S1, unofficial].
    *   T3/T4 and T5/T6 simultaneously process the blocks [S1, unofficial].
    *   After a block is sent, the sender sleeps until its slaves finish, then sends the next block [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import queue
import threading

# Parameters from evidence
COUNT_PHYSICAL = 2
COUNT_LOGICAL = 2

# Data structures
# blocks: list of block objects
# rays: list of ray objects
# block_ray_lists[block_id][phys_id][log_id]: list of ray objects

def main():
    # Initialize data structures
    blocks = [...] # Not given in the sources.
    rays = [...]   # Not given in the sources.
    block_ray_lists = [[[[] for _ in range(COUNT_LOGICAL)] for _ in range(COUNT_PHYSICAL)] for _ in range(len(blocks))]

    # Communication channels
    master_to_slave_queues = [queue.Queue() for _ in range(COUNT_PHYSICAL)]
    slave_to_master_event = threading.Event()
    worker_queues = [[queue.Queue() for _ in range(COUNT_LOGICAL)] for _ in range(COUNT_PHYSICAL)]
    worker_done_events = [[threading.Event() for _ in range(COUNT_LOGICAL)] for _ in range(COUNT_PHYSICAL)]

    # Thread 0: Master/Preprocessor
    def thread_0():
        # 1. T0 preprocesses, assigns rays to entry blocks, chooses lists for 8 viewing directions.
        for ray in rays:
            entry_block = find_entry_block(ray)
            block_ray_lists[entry_block.id][0][0].append(ray)

        # 2. T0 partitions lists and sends to T1/T2 (slaves)
        for phys_id in range(COUNT_PHYSICAL):
            blocks_for_phys = [b for b in blocks if hash(b.id) % COUNT_PHYSICAL == phys_id]
            master_to_slave_queues[phys_id].put(blocks_for_phys)

        # 3. T0 sleeps until slaves finish
        slave_to_master_event.wait()
        slave_to_master_event.clear()

    # Slave Manager (one per physical CPU)
    def slave_manager(phys_id):
        blocks = master_to_slave_queues[phys_id].get()
        for block in blocks:
            # 4. T1 sends blocks to T3/T4; T2 sends blocks to T5/T6
            for log_id in range(COUNT_LOGICAL):
                worker_queues[phys_id][log_id].put(block)
                worker_done_events[phys_id][log_id].clear()

            # 5. T3/T4 and T5/T6 simultaneously process blocks
            for log_id in range(COUNT_LOGICAL):
                worker_done_events[phys_id][log_id].wait()

            # 6. After block sent, sender sleeps until slaves finish, then sends next block
            # (Loop continues to next block)
        slave_to_master_event.set()

    # Worker Thread (logical CPU)
    def worker_thread(phys_id, log_id):
        while True:
            block = worker_queues[phys_id][log_id].get()
            if block is None: break

            # 2. Within a block, multiple threads simultaneously process rays
            ray_list = block_ray_lists[block.id][phys_id][log_id]
            for ray in ray_list:
                cast_ray(ray)
                if ray.exited:
                    next_block = ray.next_block
                    next_phys = hash(next_block.id) % COUNT_PHYSICAL
                    next_log = hash(next_block.id) % COUNT_LOGICAL
                    block_ray_lists[next_block.id][next_phys][next_log].append(ray)

            worker_done_events[phys_id][log_id].set()

    # Launch threads
    t0 = threading.Thread(target=thread_0)
    t0.start()

    managers = [threading.Thread(target=slave_manager, args=(i,)) for i in range(COUNT_PHYSICAL)]
    for m in managers: m.start()

    workers = []
    for p in range(COUNT_PHYSICAL):
        for l in range(COUNT_LOGICAL):
            workers.append(threading.Thread(target=worker_thread, args=(p, l)))
    for w in workers: w.start()

    t0.join()
    for m in managers: m.join()
    for w in workers: w.join()

# Standard operations
def find_entry_block(ray):
    # Traces ray to find initial volume block
    return ray.entry_block

def cast_ray(ray):
    # Advances ray through current block, updates exit state
    ray.step()
```

**Edge Cases and Pitfalls:**
*   **Cache Trashing:** Treating logical CPUs like physical CPUs leads to little or no performance increase, or even a decrease, due to cache trashing because logical processors share caches [S1, unofficial]. Threads must operate on neighboring data items (rays within the same block) [S1, unofficial].
*   **Race Conditions:** If multiple threads on different physical CPUs try to add rays to the same block's ray list simultaneously, race conditions occur. The use of per-logical-CPU ray lists per block prevents this without mutexes [S1, unofficial].

## Subtopics

### SMP Parallelization with Per-CPU Ray Lists
This variant applies to Symmetric Multiprocessing (SMP) systems where blocks in block lists are partitioned among physical CPUs [S1, unofficial]. To avoid race conditions and synchronization overhead when adding rays to block lists, each block maintains a separate ray list for every physical CPU [S1, unofficial]. When a block is being processed, rays from all its per-CPU lists are cast [S1, unofficial]. When a ray leaves the block, it is added to the new block's ray list corresponding to the CPU currently processing the ray [S1, unofficial]. Race conditions occur when rays from two simultaneously processed blocks have the same subsequent block; using mutexes or critical sections causes drastic performance overhead, so the solution is using per-CPU ray lists per block [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** On SMT hardware, this method can increase CPU utilization by keeping execution resources busy with more independent instructions available from different threads [S1, unofficial]. Performance gains are contingent on cache locality; if threads operate on non-neighboring data, performance may decrease due to cache trashing [S1, unofficial].
*   **Strengths:** No expensive synchronization primitives (mutexes/critical sections) are required [S1, unofficial]. Efficient use of SMT resources [S1, unofficial].
*   **Limitations:** Requires careful data distribution to maintain cache locality [S1, unofficial]. Overhead of managing multiple ray lists per block [S1, unofficial].
*   **When to Use:** When rendering on SMT-enabled CPUs (e.g., Intel Hyper-Threading) and cache locality can be maintained [S1, unofficial].
*   **When Not to Use:** When the workload cannot be partitioned to ensure neighboring data access for logical threads, or on hardware without SMT support [S1, unofficial].

## Background
This method originates from work on efficient volume visualization of large medical datasets [S1, unofficial]. It was developed to exploit the hardware features of SMT, such as Intel's Hyper-Threading technology, which was first available on the Pentium 4 CPU [S1, unofficial]. The approach addresses the specific challenges of SMT, where logical processors share execution resources and caches, requiring a different parallelization strategy than traditional SMP or GPU-based approaches [S1, unofficial]. Currently, two logical CPUs per physical CPU are supported [S1, unofficial].

## Sources
[S1] Efficient Volume Visualization of Large Medical Datasets, https://old.cescg.org/CESCG-2004/web/Bruckner-Stefan/html/, fetched 2026-10-30 (unofficial)
