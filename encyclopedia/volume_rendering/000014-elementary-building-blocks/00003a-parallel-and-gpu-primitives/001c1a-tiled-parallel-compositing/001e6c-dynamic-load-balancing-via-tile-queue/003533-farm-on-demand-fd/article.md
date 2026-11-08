# Farm on Demand (FD)

Also called: FD Algorithm, processor farm paradigm with on-demand distribution

## Overview
Farm on Demand (FD) is a parallel isosurface extraction algorithm that utilizes the processor farm paradigm to achieve dynamic load balancing by distributing fine-grained, non-contiguous tasks to workers on demand [S1]. It distinguishes itself from static partitioning methods like [Load-Balanced Cell Assignment](4194-load-balanced-cell-assignment.md) by using a number of tasks $n_T$ strictly greater than the number of workers $num_W$, allowing for probabilistically perfect computational load balancing at the cost of increased mesh fragmentation and shared vertices [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset (same format as the sequential algorithm) [S1].
*   **Output:** An isosurface partitioned among parallel processes. Each process produces distinct Vertex, Triangle, and Index tables for each interval of slices (task) it treats [S1].

**Data Structures:**
*   **Tasks:** The dataset is subdivided into $n_T$ intervals (tasks). Normally, each task consists of $\lfloor Z_{max} / n_T \rfloor$ slices, with the last task containing the remainder [S1].
*   **Process Roles:** The system consists of $num_P$ parallel processes, divided into one master process and $num_W$ worker processes ($num_P = num_W + 1$) [S1].

**Parameters:**
*   $n_T$: Total number of tasks (intervals of slices). Constraint: $n_T > num_W$ [S1].
*   $num_W$: Number of worker processes [S1].
*   $Z_{max}$: Maximum Z-dimension of the volume [S1].

**Algorithm Steps:**
1.  **Initialization:** The master process shares a node with one worker process. These are lightweight processes, so CPU sharing introduces negligible overhead [S1].
2.  **Subdivision:** The master subdivides the volumetric dataset into $n_T$ tasks (intervals of slices) [S1].
3.  **Initial Assignment:** The master randomly assigns these tasks to the $num_W$ worker processes. Tasks are assigned non-contiguously [S1].
4.  **Data Acquisition:** Each worker acquires the data for its assigned tasks. Unlike static methods that may read data in a single operation, FD workers acquire data in multiple I/O chunks corresponding to their assigned tasks [S1].
5.  **Computation Loop (On-Demand):**
    *   Workers execute the same isosurface extraction algorithm used in the [Load-Balanced Cell Assignment](4194-load-balanced-cell-assignment.md) (LBC) algorithm for each assigned task [S1].
    *   Workers monitor the computational cost of their current task. Because tasks have different amounts of active cells, workers finish at different rates.
    *   When a worker finishes a task, it requests a new task from the master [S1].
    *   The master assigns the next available task from the pool of unassigned tasks. This assignment is random and nondeterministic [S1].
    *   This continues until all $n_T$ tasks are completed [S1].
6.  **Output Generation:** For each task processed, the worker produces Vertex, Triangle, and Index tables. Consequently, the total number of output files is greater than in algorithms with fewer tasks [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def farm_on_demand(dataset, num_workers, n_tasks):
    """
    Parallel isosurface extraction using Farm on Demand paradigm.
    
    Args:
        dataset: The volumetric dataset.
        num_workers: Number of worker processes (numW).
        n_tasks: Total number of tasks (nT), where nT > numW.
        
    Returns:
        List of output files (Vertex, Triangle, Index tables) from all workers.
    """
    # 1. Subdivide dataset into n_tasks intervals
    # Each task i covers a range of Z-slices.
    # Task size is typically Z_max / n_tasks, with the last task taking the remainder.
    tasks = create_tasks(dataset, n_tasks)
    
    # 2. Master process manages the task queue
    # The master shares a node with one worker (negligible overhead)
    master = MasterProcess()
    worker_processes = [WorkerProcess(i) for i in range(num_workers)]
    
    # 3. Initial random assignment
    # Shuffle tasks to ensure random distribution
    random.shuffle(tasks)
    task_queue = tasks[:]
    
    # 4. Distribute initial tasks to workers
    # Workers acquire data in multiple I/O chunks
    for i, worker in enumerate(worker_processes):
        if task_queue:
            task = task_queue.pop(0)
            worker.assign_task(task)
            worker.acquire_data_in_chunks(task)
            worker.start()
            
    # 5. On-demand distribution loop
    # Workers request new tasks when they finish
    active_workers = list(worker_processes)
    
    while active_workers:
        # Check for finished workers
        finished_workers = []
        for worker in active_workers:
            if worker.is_finished():
                finished_workers.append(worker)
                
        for worker in finished_workers:
            active_workers.remove(worker)
            
            # Request new task
            if task_queue:
                new_task = task_queue.pop(0)
                worker.assign_task(new_task)
                worker.acquire_data_in_chunks(new_task)
                worker.start()
                active_workers.append(worker)
            else:
                # No more tasks, worker terminates
                worker.terminate()
                
    # 6. Collect results
    # Each worker produces Vertex, Triangle, and Index tables for each task
    all_output_files = []
    for worker in worker_processes:
        all_output_files.extend(worker.get_output_files())
        
    return all_output_files

class WorkerProcess:
    def __init__(self, id):
        self.id = id
        self.current_task = None
        self.output_files = []
        
    def assign_task(self, task):
        self.current_task = task
        
    def acquire_data_in_chunks(self, task):
        # Acquire data for the task in multiple I/O chunks
        # This is distinct from LBC's single-operation acquisition
        pass
        
    def is_finished(self):
        # Check if isosurface extraction for current task is complete
        # Computation uses the same algorithm as LBC
        return self.current_task is None or self.current_task.is_processed()
        
    def start(self):
        # Execute isosurface extraction for the assigned task
        # Produce Vertex, Triangle, and Index tables
        pass
        
    def get_output_files(self):
        return self.output_files

class MasterProcess:
    def __init__(self):
        pass
```

**Edge Cases and Pitfalls:**
*   **Shared Vertices:** Because tasks are assigned randomly and non-contiguously, processes often receive non-contiguous tasks. This leads to up to $n_T - 1$ shared slices (compared to only $num_W$ shared slices in LBC and LBAC) and probabilistically many more shared vertices [S1].
*   **I/O Overhead:** Workers acquire data in multiple chunks, which increases I/O overhead compared to LBC's single-operation acquisition [S1].
*   **Nondeterminism:** The random on-demand assignment makes the distribution nondeterministic [S1].

## Subtopics

### [Load Balanced on Active Cells (LBAC)](7511-load-balanced-on-active-cells-lbac.md)
A parallel isosurface extraction algorithm based on the data parallel paradigm with load balancing strategy based on the total number of active cells. It represents the third class of parallel algorithms.

## Operating characteristics
*   **Time Complexity:** The theoretical load balancing is almost perfect and cheap [S1]. Speed up depends on the isosurface distribution; generally provides good performance if the grain size is suitable [S1]. For small-medium isosurfaces, FD is often the best possibility, especially when considering I/O cost, due to uneven distribution and small size [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:**
    *   On the FC-1000 dataset with 16 nodes, FD produced 1,007,205 shared vertices [S1].
    *   On the XT-25 dataset with 16 nodes, FD produced 11,847,754 shared vertices [S1].
    *   For FC-1000, FD achieved a percentage of active cells received by a worker varying between 5.5% and 6.4%, indicating quite perfect balancing [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:**
    *   Provides good load balancing for small-medium isosurfaces due to random assignment [S1].
    *   Theoretically allows almost perfect and cheap load balancing [S1].
    *   Effective when load-balanced computation is prioritized [S1].
*   **Limitations:**
    *   Probabilistically produces many more shared vertices than LBC and LBAC because contiguous tasks are assigned to different processes [S1].
    *   Mesh partitioning quality is rarely completely satisfactory due to high shared vertices [S1].
    *   Multi-chunk I/O acquisition increases overhead compared to LBC [S1].
*   **When to use:** Suitable when load-balanced computation is prioritized over mesh partitioning quality, particularly for small-medium isosurfaces [S1].
*   **When not to use:** When mesh partitioning quality is critical or when the number of shared vertices significantly impacts downstream processing (e.g., rendering or mesh redistribution) [S1].

## Background
*   **Origin:** The Farm on Demand (FD) algorithm was proposed by D. D’Agostino, A. Clematis, and V. Gianuzzi [S1].
*   **History:** It represents the fourth class of parallel isosurface extraction algorithms, improving upon previous methods by introducing dynamic, on-demand task distribution [S1]. It is similar to the algorithm in [44], but the main difference is that the random task distribution among workers is made on demand, making the distribution nondeterministic [S1].
*   **Usage:** It is used in distributed environments for 3D data analysis workflows, particularly where computational load is uneven [S1].

## Sources
[S1] Parallel Isosurface Extraction for 3D Data Analysis Workflows in Distributed Environments, https://doi.org/10.1002/cpe.1710, fetched 2026-10-26
