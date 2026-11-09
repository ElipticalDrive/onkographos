# Farm on Demand (FD)

Also called: FD Algorithm, processor farm paradigm with on-demand distribution

## Overview
Farm on Demand (FD) is a parallel isosurface extraction algorithm that achieves dynamic load balancing by subdividing a volumetric dataset into a large number of fine-grained tasks and distributing them to worker processes on demand [S1]. It builds on [Dynamic Load Balancing via Tile Queue](1763-dynamic-load-balancing-via-tile-queue.md) by applying the processor farm paradigm to volume data, where the master assigns small intervals of slices to workers as they become available, rather than using static partitioning [S1]. Unlike its sibling [Load Balanced Cell Assignment](4194-load-balanced-cell-assignment.md) (LBC), which uses static contiguous partitions, FD provides near-perfect computational load balancing for uneven isosurface distributions but probabilistically produces a significantly higher number of shared vertices due to non-contiguous task assignment [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset (same format as the sequential algorithm) [S1].
*   **Output:** An isosurface partitioned among parallel processes. Each process produces distinct Vertex, Triangle, and Index tables for each task interval it treats [S1].

**Data Structures:**
*   **Tasks:** The dataset is subdivided into $n_T$ intervals of slices. Each task corresponds to a specific range of Z-slices.
*   **Shared State:** A task queue or pool managed by the master process, containing the list of available tasks [S1].
*   **Worker State:** Each worker maintains a local buffer for the current task's data and output tables (Vertex, Triangle, Index) [S1].

**Parameters:**
*   $n_T$: Total number of tasks (intervals of slices). $n_T > num_W$ [S1].
*   $num_W$: Number of worker processes [S1].
*   Task size: Normally $Z_{max} / n_T$ slices, with the last task possibly containing a different amount [S1].

**Steps:**
1.  **Initialization:** The master process shares a node with one worker process. The master subdivides the dataset into $n_T$ tasks [S1].
2.  **Task Distribution:** The master randomly assigns tasks to worker processes. This assignment is non-deterministic and non-contiguous [S1].
3.  **On-Demand Request:** Workers execute the isosurface extraction algorithm (identical to the one used in LBC) for their assigned tasks. Upon completing a task, a worker requests a new task from the master [S1].
4.  **Data Acquisition:** Workers acquire the data for their assigned tasks in multiple I/O chunks, rather than a single contiguous read [S1].
5.  **Computation:** Workers compute the isosurface for the assigned cells. The computation is balanced implicitly because workers with tasks containing fewer active cells finish faster and request new tasks sooner [S1].
6.  **Output Generation:** For each task, workers produce Vertex, Triangle, and Index tables. These are written to distinct files or buffers [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random

class FarmOnDemand:
    def __init__(self, dataset, num_workers, num_tasks):
        self.dataset = dataset
        self.num_workers = num_workers
        self.num_tasks = num_tasks
        # Subdivide dataset into intervals of slices
        self.tasks = self._subdivide_dataset(dataset, num_tasks)
        # Master process manages the pool of available tasks
        self.task_pool = list(self.tasks)
        random.shuffle(self.task_pool) # Random assignment

    def _subdivide_dataset(self, dataset, n):
        """
        Subdivides the volumetric dataset into n intervals of slices.
        Returns a list of task objects, each containing a slice range.
        """
        z_max = dataset.z_dimension
        tasks = []
        slices_per_task = z_max / n
        for i in range(n):
            start_slice = int(i * slices_per_task)
            end_slice = int((i + 1) * slices_per_task)
            if i == n - 1:
                end_slice = z_max # Last task may vary in size
            tasks.append(Task(slice_range=(start_slice, end_slice)))
        return tasks

    def run(self):
        """
        Master process distributes tasks to workers.
        """
        workers = [Worker(i, self.dataset) for i in range(self.num_workers)]
        
        # Initial distribution
        for worker in workers:
            if self.task_pool:
                task = self.task_pool.pop(0)
                worker.assign_task(task)
        
        # On-demand distribution loop
        while any(not w.is_idle() for w in workers):
            for worker in workers:
                if worker.is_idle() and self.task_pool:
                    task = self.task_pool.pop(0)
                    worker.assign_task(task)
                elif worker.is_idle():
                    # No more tasks, worker waits or exits
                    pass

        # Collect results
        all_vertices = []
        all_triangles = []
        all_indices = []
        for worker in workers:
            all_vertices.extend(worker.get_vertex_tables())
            all_triangles.extend(worker.get_triangle_tables())
            all_indices.extend(worker.get_index_tables())
            
        return Isosurface(vertices=all_vertices, triangles=all_triangles, indices=all_indices)

class Worker:
    def __init__(self, id, dataset):
        self.id = id
        self.dataset = dataset
        self.current_task = None
        self.vertex_tables = []
        self.triangle_tables = []
        self.index_tables = []
        self.busy = False

    def assign_task(self, task):
        self.current_task = task
        self.busy = True
        self._execute_task()

    def _execute_task(self):
        """
        Executes the isosurface extraction algorithm for the assigned task.
        Acquires data in multiple chunks.
        """
        # Acquire data in chunks
        data_chunks = self.dataset.read_chunks(self.current_task.slice_range)
        
        # Compute isosurface using the same algorithm as LBC
        vertices, triangles, indices = isosurface_extraction_algorithm(data_chunks)
        
        self.vertex_tables.append(vertices)
        self.triangle_tables.append(triangles)
        self.index_tables.append(indices)
        self.busy = False

    def is_idle(self):
        return not self.busy

    def get_vertex_tables(self):
        return self.vertex_tables

    def get_triangle_tables(self):
        return self.triangle_tables

    def get_index_tables(self):
        return self.index_tables

def isosurface_extraction_algorithm(data):
    """
    Placeholder for the specific isosurface extraction algorithm (e.g., Marching Cubes).
    This is the same algorithm used in LBC.
    """
    # Implementation details of the extraction algorithm
    # ...
    pass

class Task:
    def __init__(self, slice_range):
        self.slice_range = slice_range
```

**Edge Cases and Pitfalls:**
*   **Shared Vertices:** Because tasks are assigned randomly and non-contiguously, processes often receive non-contiguous tasks. This leads to up to $n_T - 1$ shared slices (compared to only $num_W$ in LBC), resulting in a probabilistically much higher number of shared vertices [S1]. For example, in the FC-1000 dataset, FD produced 1,007,205 shared vertices compared to 195,804 for LBC [S1].
*   **I/O Overhead:** Workers acquire data in multiple chunks, which increases I/O overhead compared to LBC's single-operation acquisition [S1].
*   **Mesh Quality:** The suitability of the isosurface partitioning is rarely completely satisfactory due to the high number of shared vertices, which may necessitate mesh redistribution [S1].

## Subtopics

### [Load Balanced on Active Cells (LBAC)](7511-load-balanced-on-active-cells-lbac.md)
LBAC is a parallel isosurface extraction algorithm that partitions a 3D volume into contiguous regions along the Z-axis and dynamically balances the computational workload by equalizing the number of **active cells** (cells containing isosurface intersections) among worker processes. Unlike its sibling [Load Balanced on Cells (LBC)](52-load-balanced-on-cells-lbc.md), which distributes the total number of cells evenly but suffers from unpredictable active cell distribution, LBAC ensures near-perfect load balancing for the triangulation phase by performing a preliminary analysis of active slice weights. It builds on [Farm on Demand (FD)](7455-farm-on-demand-fd.md) by using a static, contiguous Z-axis partitioning strategy rather than FD's random task assignment, which minimizes shared vertex overhead but requires a two-phase approach to achieve balance.

## Operating characteristics
*   **Time Complexity:** Theoretical load balancing is almost perfect and cheap, as balancing is obtained dynamically without explicit active cell counting [S1]. Speed up depends on the isosurface distribution; generally provides good performance if the grain size is suitable [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** For small-medium isosurfaces, FD is often the best possibility, especially when considering I/O cost, due to the uneven distribution of isosurfaces and their small size [S1]. For large isosurfaces, LBAC provides better results [S1].
*   **Strengths:** Excellent computational load balancing for uneven isosurface distributions; good performance for small-medium isosurfaces [S1].
*   **Limitations:** Probabilistically produces many more shared vertices than LBC and LBAC; high I/O overhead due to multi-chunk data acquisition; mesh partitioning quality is often unsatisfactory [S1].
*   **When to Use:** Suitable when load-balanced computation is prioritized over mesh partitioning quality, particularly for small-medium isosurfaces with uneven distributions [S1].
*   **When Not to Use:** When mesh quality (low shared vertices) is critical, or for large isosurfaces where LBAC outperforms FD [S1].

## Background
*   **Origin:** The Farm on Demand (FD) algorithm was proposed by D. D'Agostino, A. Clematis, and V. Gianuzzi [S1].
*   **History:** It represents the fourth class of parallel isosurface extraction algorithms, based on the processor farm paradigm [S1]. It is similar to the algorithm proposed in [44] but differs in that the random task distribution among workers is made on demand, making the distribution nondeterministic [S1].
*   **Usage:** It is used in distributed environments for 3D data analysis workflows, particularly where dynamic load balancing is crucial for performance [S1].

## Sources
[S1] Parallel Isosurface Extraction for 3D Data Analysis Workflows in Distributed Environments, https://doi.org/10.1002/cpe.1710, fetched 2026-10-26
