# Farm on Demand (FD)

Also called: FD Algorithm, processor farm paradigm with on-demand distribution

## Overview
Farm on Demand (FD) is a parallel isosurface extraction algorithm that achieves dynamic load balancing by subdividing a volumetric dataset into a large number of fine-grained tasks and distributing them to worker processes on demand [S1]. It builds on [Dynamic Load Balancing via Tile Queue](1763-dynamic-load-balancing-via-tile-queue.md) by applying the processor farm paradigm to volume data, where the master assigns small intervals of slices to workers as they become available, rather than using static partitioning [S1]. Unlike its sibling [Load Balanced Cell Assignment](4194-load-balanced-cell-assignment.md) (LBC), which uses static contiguous partitions, FD provides near-perfect computational load balancing for uneven isosurface distributions but probabilistically produces a significantly higher number of shared vertices due to non-contiguous task assignment [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset in the same format as the sequential algorithm [S1].
*   **Output:** An isosurface partitioned among parallel processes. Each process produces distinct Vertex, Triangle, and Index tables for each task interval it treats [S1].

**Data Structures:**
*   **Tasks:** The dataset is subdivided into $n_T$ intervals of slices. Each task corresponds to a specific range of Z-slices [S1].
*   **Shared State:** A task pool managed by the master process, containing the list of available tasks [S1].
*   **Worker State:** Each worker maintains output tables (Vertex, Triangle, Index) for each task it processes [S1].

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
import math
import random

class FarmOnDemand:
    def __init__(self, dataset, num_workers, num_tasks):
        self.dataset = dataset
        self.num_workers = num_workers
        self.num_tasks = num_tasks
        self.tasks = self._subdivide_dataset(dataset, num_tasks)
        self.task_pool = list(self.tasks)
        random.shuffle(self.task_pool)

    def _subdivide_dataset(self, dataset, n):
        z_max = dataset.z_dimension
        tasks = []
        slices_per_task = z_max / n
        for i in range(n):
            start_slice = int(i * slices_per_task)
            end_slice = int((i + 1) * slices_per_task)
            if i == n - 1:
                end_slice = z_max
            tasks.append(Task(slice_range=(start_slice, end_slice)))
        return tasks

    def run(self):
        workers = [Worker(i, self.dataset) for i in range(self.num_workers)]
        for worker in workers:
            if self.task_pool:
                task = self.task_pool.pop(0)
                worker.assign_task(task)
        while any(not w.is_idle() for w in workers):
            for worker in workers:
                if worker.is_idle() and self.task_pool:
                    task = self.task_pool.pop(0)
                    worker.assign_task(task)
        all_vertices, all_triangles, all_indices = [], [], []
        for worker in workers:
            all_vertices.extend(worker.vertex_tables)
            all_triangles.extend(worker.triangle_tables)
            all_indices.extend(worker.index_tables)
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
        data_chunks = self.dataset.read_chunks(self.current_task.slice_range)
        vertices, triangles, indices = marching_cubes(data_chunks, threshold=0.5)
        self.vertex_tables.append(vertices)
        self.triangle_tables.append(triangles)
        self.index_tables.append(indices)
        self.busy = False

    def is_idle(self):
        return not self.busy

class Task:
    def __init__(self, slice_range):
        self.slice_range = slice_range

class Isosurface:
    def __init__(self, vertices, triangles, indices):
        self.vertices = vertices
        self.triangles = triangles
        self.indices = indices

def marching_cubes(grid, threshold):
    # Standard Marching Cubes algorithm [G]
    # grid: 3D list of floats, dimensions (Z, Y, X)
    # Returns: vertices (list of tuples), triangles (list of tuples), indices (list of tuples)
    edge_table = [
        0x0, 0x109, 0x205, 0x30c, 0x403, 0x50a, 0x606, 0x70f, 0x80c, 0x905, 0xa09, 0xb00, 0xc0f, 0xd06, 0xe0a, 0xf03,
        0x190, 0x99, 0x395, 0x29c, 0x593, 0x49a, 0x796, 0x69f, 0x90c, 0x805, 0xb09, 0xa00, 0xd0f, 0xc06, 0xf0a, 0xe03,
        0x250, 0x359, 0x55, 0x15c, 0x653, 0x75a, 0x456, 0x55f, 0xa0c, 0xb05, 0x809, 0x900, 0xe0f, 0xf06, 0xc0a, 0xd03,
        0x3c0, 0x2c9, 0x1c5, 0xcc, 0x7c3, 0x6ca, 0x5c6, 0x4cf, 0xb0c, 0xa05, 0x909, 0x800, 0xf0f, 0xe06, 0xd0a, 0xc03,
        0x483, 0x58a, 0x686, 0x78f, 0x88, 0x181, 0x28d, 0x384, 0xc0c, 0xd05, 0xe09, 0xf00, 0x80f, 0x906, 0xa0a, 0xb03,
        0x513, 0x41a, 0x716, 0x61f, 0x113, 0x11a, 0x216, 0x31f, 0xd0c, 0xc05, 0xf09, 0xe00, 0x90f, 0x806, 0xb0a, 0xa03,
        0x653, 0x75a, 0x456, 0x55f, 0x253, 0x35a, 0x56, 0x15f, 0xe0c, 0xf05, 0xc09, 0xd00, 0xb0f, 0xa06, 0x90a, 0x803,
        0x7c3, 0x6ca, 0x5c6, 0x4cf, 0x3c3, 0x2ca, 0x1c6, 0xccf, 0xf0c, 0xe05, 0xd09, 0xc00, 0xa0f, 0xb06, 0x80a, 0x903,
        0x80c, 0x905, 0xa09, 0xb00, 0xc0f, 0xd06, 0xe0a, 0xf03, 0x0, 0x109, 0x205, 0x30c, 0x403, 0x50a, 0x606, 0x70f,
        0x990, 0x899, 0xb95, 0xa9c, 0xd93, 0xc9a, 0xf96, 0xe9f, 0x10c, 0x05, 0x309, 0x200, 0x50f, 0x406, 0x70a, 0x603,
        0xa50, 0xb59, 0x855, 0x95c, 0xe53, 0xf5a, 0xc56, 0xd5f, 0x20c, 0x305, 0x009, 0x100, 0x60f, 0x706, 0x40a, 0x503,
        0xbc0, 0xac9, 0x9c5, 0x8cc, 0xfc3, 0xec6, 0xdc6, 0xccf, 0x30c, 0x205, 0x109, 0x000, 0x70f, 0x606, 0x50a, 0x403,
        0xc83, 0xd8a, 0xe86, 0xf8f, 0x88, 0x181, 0x28d, 0x384, 0x40c, 0x505, 0x609, 0x700, 0x00f, 0x106, 0x20a, 0x303,
        0xd13, 0xc1a, 0xf16, 0xe1f, 0x113, 0x11a, 0x216, 0x31f, 0x50c, 0x405, 0x709, 0x600, 0x10f, 0x006, 0x30a, 0x203,
        0xe53, 0xf5a, 0xc56, 0xd5f, 0xa53, 0xb5a, 0x856, 0x95f, 0x60c, 0x705, 0x409, 0x500, 0x20f, 0x306, 0x00a, 0x103,
        0xfc3, 0xec6, 0xdc6, 0xccf, 0xbc3, 0xac6, 0x9c6, 0x8cf, 0x70c, 0x605, 0x509, 0x400, 0x30f, 0x206, 0x10a, 0x003
    ]
    tri_table = [
        [-1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [0, 8, 3, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [0, 1, 9, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [0, 1, 2, 8, 3, 9, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [1, 8, 2, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [0, 8, 1, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [1, 2, 9, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [2, 3, 9, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [3, 4, 0, 5, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [0, 4, 3, 1, 5, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 1, 5, 9, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 0, 9, 5, 1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 2, 3, 5, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [2, 1, 4, 5, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [1, 0, 4, 2, 5, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [3, 0, 1, 5, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [5, 8, 2, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [5, 2, 3, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [2, 0, 2, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [5, 0, 8, 2, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [2, 0, 1, 5, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [0, 5, 9, 2, 3, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [0, 5, 9, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 5, 9, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [7, 8, 0, 3, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [0, 7, 3, 9, 5, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [0, 7, 1, 9, 5, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [2, 3, 0, 1, 9, 5, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [1, 7, 8, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [1, 7, 3, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [1, 7, 2, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [1, 5, 9, 7, 2, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [3, 7, 8, 5, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [3, 0, 5, 7, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [0, 1, 5, 7, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [1, 2, 5, 7, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [8, 5, 2, 7, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [2, 3, 5, 7, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [2, 0, 5, 7, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [0, 1, 5, 7, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [6, 8, 4, 5, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 5, 6, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [5, 6, 1, 9, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [6, 1, 9, 5, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [6, 4, 3, 0, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 6, 3, 1, 0, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [1, 6, 3, 9, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [3, 9, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 5, 7, 6, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [5, 7, 0, 4, 3, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [0, 7, 6, 1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [0, 7, 6, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 7, 6, 1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 3, 7, 6, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [6, 8, 4, 5, 2, 3, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 5, 2, 3, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [2, 1, 6, 8, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [1, 6, 8, 5, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [8, 2, 3, 4, 6, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 6, 8, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [6, 1, 9, 5, 2, 3, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 6, 1, 9, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [6, 0, 4, 9, 3, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [9, 5, 4, 2, 0, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [0, 1, 4, 9, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 9, 5, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [5, 7, 2, 6, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [2, 3, 5, 7, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [0, 6, 1, 7, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [0, 6, 7, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 7, 6, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 7, 6, 1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [2, 5, 1, 3, 0, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [3, 0, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [1, 6, 8, 5, 3, 0, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [3, 0, 8, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 6, 8, 5, 2, 3, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 6, 8, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [6, 1, 9, 5, 2, 3, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [2, 3, 9, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [2, 0, 4, 9, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [9, 5, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [5, 7, 2, 6, 1, 0, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [5, 7, 0, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [0, 1, 6, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [0, 6, 4, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 1, 6, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1],
        [4, 1, 6, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1]
    ]
    vertices, triangles, indices = [], [], []
    z_dim, y_dim, x_dim = len(grid), len(grid[0]), len(grid[0][0])
    for z in range(z_dim - 1):
        for y in range(y_dim - 1):
            for x in range(x_dim - 1):
                idx = 0
                if grid[z][y][x] >= threshold: idx |= 1
                if grid[z][y][x+1] >= threshold: idx |= 2
                if grid[z][y+1][x+1] >= threshold: idx |= 4
                if grid[z][y+1][x] >= threshold: idx |= 8
                if grid[z+1][y][x] >= threshold: idx |= 16
                if grid[z+1][y][x+1] >= threshold: idx |= 32
                if grid[z+1][y+1][x+1] >= threshold: idx |= 64
                if grid[z+1][y+1][x] >= threshold: idx |= 128
                edge_mask = edge_table[idx]
                if edge_mask == 0: continue
                for i in range(0, 16, 3):
                    if tri_table[idx][i] == -1: break
                    v0, v1, v2 = tri_table[idx][i], tri_table[idx][i+1], tri_table[idx][i+2]
                    # Interpolate edge vertices (simplified for brevity)
                    p0, p1, p2 = (x, y, z), (x+1, y, z), (x, y+1, z)
                    vertices.extend([p0, p1, p2])
                    triangles.append((len(vertices)-3, len(vertices)-2, len(vertices)-1))
                    indices.append((v0, v1, v2))
    return vertices, triangles, indices
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
*   **Memory Complexity:** Memory complexity is not given in the sources [S1].
*   **Measured Performance:** For small-medium isosurfaces, FD is often the best possibility, especially when considering I/O cost, due to the uneven distribution of isosurfaces and their small size [S1]. For large isosurfaces, LBAC provides better results [S1].
*   **Strengths:** Excellent computational load balancing for uneven isosurface distributions; good performance for small-medium isosurfaces [S1].
*   **Limitations:** Probabilistically produces many more shared vertices than LBC and LBAC; high I/O overhead due to multi-chunk data acquisition; mesh partitioning quality is often unsatisfactory [S1].
*   **When to Use:** Suitable when load-balanced computation is prioritized over mesh partitioning quality, particularly for small-medium isosurfaces with uneven distributions [S1].
*   **When Not to Use:** When mesh quality (low shared vertices) is critical, or for large isosurfaces where LBAC outperforms FD [S1].

## Background
*   **Origin:** The Farm on Demand (FD) algorithm was proposed by D. D'Agostino, A. Clematis, and V. Gianuzzi [S1].
*   **History:** It represents the fourth class of parallel isosurface extraction algorithms, based on the processor farm paradigm [S1]. It is similar to the algorithm proposed in [44] but differs in that the random task distribution among workers is made on demand, making the distribution nondeterministic [S1].

## Sources
[S1] Parallel Isosurface Extraction for 3D Data Analysis Workflows in Distributed Environments, https://doi.org/10.1002/cpe.1710, fetched 2026-10-26
