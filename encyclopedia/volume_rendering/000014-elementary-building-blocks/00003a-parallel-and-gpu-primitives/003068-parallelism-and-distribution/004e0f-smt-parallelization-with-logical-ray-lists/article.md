# SMT Parallelization with Logical Ray Lists

Also called: Logical Ray Lists, SMT Parallelization

## Overview
This method is a parallelization strategy for Simultaneous Multithreading (SMT) architectures, designed to maximize CPU utilization by allowing multiple threads to operate within the same processing block while sharing cache resources [S1, unofficial]. It solves the problem of race conditions and cache trashing that occur when multiple threads attempt to modify shared ray lists or operate on non-neighboring data [S1, unofficial]. Unlike [Parallel Image Compositing with Overlapping](3554-parallel-image-compositing-with-overlapping.md) or [Ray Compaction](4039-ray-compaction.md), which often rely on global synchronization or compacted data structures, this method uses a hierarchical distribution of ray lists—one per logical CPU per block—to eliminate the need for expensive synchronization primitives like mutexes [S1, unofficial]. It builds on [Parallelism and Distribution](6265-parallelism-and-distribution.md) by specifically addressing the hardware constraints of SMT where logical processors share physical resources.

## How it works
**Inputs:** A set of rays to be cast through a volume; the number of physical CPUs ($N_{phys}$); the number of logical CPUs per physical CPU ($N_{log}$); the volume data structure; and the viewing directions.
**Outputs:** The final rendered image (composited colors and opacities).

**Data Structures:**
*   **Block List:** A linked list or array of blocks (spatial partitions of the volume) to be processed.
*   **Ray List:** A list of rays currently inside a specific block, waiting to be processed.
*   **Per-Block Ray Lists:** Each block maintains $N_{phys} \times N_{log}$ distinct ray lists.
    *   Indexing: `ray_lists[phys_id][log_id]`
    *   `phys_id`: Identifier of the physical CPU currently processing the block.
    *   `log_id`: Identifier of the logical CPU (thread) within that physical CPU.

**Parameters:**
*   $N_{phys}$: Number of physical CPUs.
*   $N_{log}$: Number of logical CPUs per physical CPU (threads per block).
*   Total ray lists per block: $N_{phys} \times N_{log}$.

**Steps:**
1.  **Initialization:** Start $N_{phys} \times N_{log} + 1$ threads (or processes). Thread $T_0$ is the master/preprocessor. Threads $T_1$ to $T_{N_{phys} \times N_{log}}$ are worker threads.
2.  **Ray Assignment (Master $T_0$):**
    *   $T_0$ preprocesses all rays to determine their entry blocks into the volume.
    *   $T_0$ assigns each ray to the appropriate initial block.
    *   $T_0$ selects which blocks can be processed simultaneously based on viewing directions and available logical CPUs.
    *   $T_0$ partitions the block lists among the worker threads. Specifically, it distributes blocks such that each physical CPU's workers receive a subset of blocks.
    *   $T_0$ sends the initial partition of blocks to the slave threads.
    *   $T_0$ sleeps until all slaves finish their current assignments.
3.  **Block Distribution (Slave Threads):**
    *   Each slave thread (acting as an intermediate manager for its physical CPU's logical cores) receives a list of blocks.
    *   For each block in its list, the thread distributes the block to the logical CPUs (threads) associated with its physical CPU.
    *   Example: If $N_{phys}=2$ and $N_{log}=2$, Thread $T_1$ (on Phys CPU 0) sends blocks to $T_3$ and $T_4$ (Logical CPUs 0 and 1 of Phys CPU 0). Thread $T_2$ (on Phys CPU 1) sends blocks to $T_5$ and $T_6$.
4.  **Ray Casting (Worker Threads):**
    *   Each worker thread (e.g., $T_3$) processes the rays in its assigned **logical** ray list for the current block.
    *   Crucially, threads on the same physical CPU operate on **neighboring data items** (rays within the same block) to ensure cache locality.
    *   The thread casts rays, updates the volume traversal, and determines exit blocks.
5.  **Ray Transfer:**
    *   When a ray exits the current block, it is added to the ray list of the **next** block.
    *   The destination list is determined by the physical CPU currently processing that next block and the logical CPU assigned to it.
    *   Specifically, the ray is added to `ray_lists[next_block_phys_id][next_block_log_id]`.
6.  **Synchronization and Flow Control:**
    *   After sending a block to its logical workers, the sender thread sleeps until the workers finish processing that block.
    *   Once finished, the sender retrieves the next block from its list and repeats the distribution.
    *   This continues until all blocks are processed and all rays have exited the volume.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global constants
N_PHYS = 2  # Example: 2 physical CPUs
N_LOG = 2   # Example: 2 logical CPUs per physical CPU
TOTAL_THREADS = N_PHYS * N_LOG + 1  # 5 threads total in this example (T0-T4)

# Data structures
# blocks: List of Block objects
# rays: List of Ray objects
# block_ray_lists[block_id][phys_id][log_id]: List of Ray objects

def main():
    # Initialize blocks and rays
    blocks = create_blocks(volume)
    rays = create_rays(viewing_directions)
    
    # Initialize ray lists: blocks[i].ray_lists[phys][log]
    for block in blocks:
        block.ray_lists = [[[] for _ in range(N_LOG)] for _ in range(N_PHYS)]
        
    # Assign rays to initial blocks
    for ray in rays:
        entry_block = find_entry_block(ray, volume)
        # Determine which physical CPU will process this block initially
        # For simplicity, assume T0 distributes based on block ID hash
        phys_id = hash(entry_block.id) % N_PHYS
        log_id = 0 # Initial assignment, refined by distribution logic
        block.ray_lists[phys_id][log_id].append(ray)
        
    # Thread 0: Master/Preprocessor
    if is_thread(0):
        # Partition blocks among physical CPUs
        # T0 sends lists of blocks to T1 (Phys 0) and T2 (Phys 1)
        blocks_for_phys_0 = [b for b in blocks if hash(b.id) % N_PHYS == 0]
        blocks_for_phys_1 = [b for b in blocks if hash(b.id) % N_PHYS == 1]
        
        send_to_thread(1, blocks_for_phys_0)
        send_to_thread(2, blocks_for_phys_1)
        
        wait_for_threads([1, 2])
        
    # Thread 1: Manager for Physical CPU 0
    elif is_thread(1):
        blocks = receive_from_thread(0)
        for block in blocks:
            # Distribute block to logical CPUs 0 and 1 of Physical CPU 0
            # In a real implementation, rays might be split or both threads process
            # the same block's specific logical ray list.
            # Here, we assume the block is processed by the logical threads.
            # The logical threads (3 and 4) will pick up the rays from 
            # block.ray_lists[0][0] and block.ray_lists[0][1]
            
            # Signal logical threads to process this block
            send_to_thread(3, block)
            send_to_thread(4, block)
            
            # Wait for logical threads to finish this block
            wait_for_threads([3, 4])
            
            # After processing, rays are moved to next blocks' lists
            # The logical threads handle the actual ray casting and list updates
            
    # Thread 2: Manager for Physical CPU 1
    elif is_thread(2):
        blocks = receive_from_thread(0)
        for block in blocks:
            send_to_thread(5, block)
            send_to_thread(6, block)
            wait_for_threads([5, 6])
            
    # Thread 3: Logical CPU 0 of Physical CPU 0
    elif is_thread(3):
        while True:
            block = receive_from_thread(1)
            if block is None: break
            
            # Process rays in the logical ray list for this block
            # Phys ID is 0, Log ID is 0
            ray_list = block.ray_lists[0][0]
            for ray in ray_list:
                cast_ray(ray, volume)
                if ray.exited:
                    next_block = ray.next_block
                    # Determine which phys/log will process next_block
                    next_phys = hash(next_block.id) % N_PHYS
                    next_log = get_logical_id_for_block(next_block, next_phys)
                    next_block.ray_lists[next_phys][next_log].append(ray)
                    
            # Signal completion
            signal_thread(1) # Notify manager
            
    # Thread 4: Logical CPU 1 of Physical CPU 0
    elif is_thread(4):
        while True:
            block = receive_from_thread(1)
            if block is None: break
            
            # Process rays in the logical ray list for this block
            # Phys ID is 0, Log ID is 1
            ray_list = block.ray_lists[0][1]
            for ray in ray_list:
                cast_ray(ray, volume)
                if ray.exited:
                    next_block = ray.next_block
                    next_phys = hash(next_block.id) % N_PHYS
                    next_log = get_logical_id_for_block(next_block, next_phys)
                    next_block.ray_lists[next_phys][next_log].append(ray)
                    
            signal_thread(1)

    # Threads 5 and 6 follow the same pattern for Physical CPU 1
    # ...

def get_logical_id_for_block(block, phys_id):
    # Determine which logical CPU should handle the next block
    # This depends on the distribution strategy. 
    # For SMT, we want to keep threads on the same physical CPU working on 
    # neighboring data.
    # A simple heuristic: round-robin or based on block ID.
    return hash(block.id) % N_LOG
```

**Edge Cases and Pitfalls:**
*   **Cache Trashing:** If logical CPUs are treated as independent physical CPUs and operate on non-neighboring data, cache performance degrades significantly due to sharing the same L1/L2 caches [S1, unofficial]. Threads must operate on neighboring data items (rays within the same block) [S1, unofficial].
*   **Race Conditions:** If multiple threads on different physical CPUs try to add rays to the same block's ray list simultaneously, race conditions occur. The use of per-logical-CPU ray lists per block prevents this without mutexes [S1, unofficial].
*   **Load Imbalance:** If rays are not evenly distributed among logical CPUs, some threads may idle while others are busy. The master thread ($T_0$) should balance the load when partitioning blocks [S1, unofficial].

## Subtopics

### SMP Parallelization with Per-CPU Ray Lists
This variant applies to Symmetric Multiprocessing (SMP) systems where each physical CPU has its own cache. Blocks are partitioned among physical CPUs. To avoid race conditions when rays from two simultaneously processed blocks target the same subsequent block, each block maintains a separate ray list for every physical CPU. When a ray leaves a block, it is added to the new block's ray list corresponding to the CPU currently processing the ray. This avoids the need for mutexes, which would cause drastic performance overhead [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays and blocks, $O(R + B)$, where $R$ is the number of rays and $B$ is the number of blocks. The constant factor is reduced by parallelism.
*   **Memory Complexity:** $O(B \times N_{phys} \times N_{log})$ for the ray lists, plus $O(R)$ for the rays themselves. The overhead of multiple ray lists per block is significant but necessary to avoid synchronization.
*   **Performance:** On SMT hardware (e.g., Hyper-Threading), this method can increase CPU utilization by keeping execution resources busy with independent instructions from different threads [S1, unofficial]. However, performance gains are contingent on cache locality. If threads operate on non-neighboring data, performance may decrease due to cache trashing [S1, unofficial].
*   **Strengths:** No expensive synchronization primitives (mutexes/critical sections) are required. Efficient use of SMT resources.
*   **Limitations:** Requires careful data distribution to maintain cache locality. Overhead of managing multiple ray lists per block. Not suitable for systems where logical CPUs do not share caches (i.e., standard SMP without SMT, though the SMP variant exists).
*   **When to Use:** When rendering on SMT-enabled CPUs (e.g., Intel Hyper-Threading, IBM Power SMT) and cache locality can be maintained.
*   **When Not to Use:** When the workload cannot be partitioned to ensure neighboring data access for logical threads, or on hardware without SMT support (use SMP variant instead).

## Background
This method originates from the work of Stefan Bruckner et al. in the context of efficient volume visualization of large medical datasets [S1, unofficial]. It was developed to exploit the hardware features of SMT, such as Intel's Hyper-Threading technology, which was first available on the Pentium 4 CPU [S1, unofficial]. The approach addresses the specific challenges of SMT, where logical processors share execution resources and caches, requiring a different parallelization strategy than traditional SMP or GPU-based approaches. The method is described in the paper "Efficient Volume Visualization of Large Medical Datasets" presented at CESCG-2004 [S1, unofficial].

## Sources
[S1] Efficient Volume Visualization of Large Medical Datasets, https://old.cescg.org/CESCG-2004/web/Bruckner-Stefan/html/, fetched 2026-10-30 (unofficial)
