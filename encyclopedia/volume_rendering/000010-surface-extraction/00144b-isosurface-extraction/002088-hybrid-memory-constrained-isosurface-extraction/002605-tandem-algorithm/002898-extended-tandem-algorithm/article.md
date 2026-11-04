# Extended Tandem Algorithm

Also called: Parallel Tandem Algorithm with Component Dumping, Parallel tandem with dumping

## Overview
The Extended Tandem Algorithm is a parallel, block-wise extension of the Tandem Algorithm [3675-tandem-algorithm.md] designed to extract and simplify large isosurfaces while minimizing memory consumption. It solves the scalability limits of the sequential tandem algorithm by combining on-the-fly simplification with a component-based dumping strategy and a hierarchical block splitting mechanism. Unlike the parent method, which processes a single volume sequentially, this method distributes the workload across multiple workers using a manager-worker architecture, allowing it to handle datasets that exceed available RAM by dumping completed surface components to disk.

## How it works
The algorithm operates on a distributed dataset split into blocks. It uses a worker-manager architecture to dynamically dispatch tasks, ensuring that adjacent blocks are processed together to maintain mesh continuity while keeping memory usage low.

### Inputs and Outputs
- **Input**: A volumetric dataset $V$ of size $D_x \times D_y \times D_z$, an isosurface value $I$, and an error threshold $E_0$ for simplification.
- **Output**: A simplified isosurface mesh, potentially split into multiple components stored on disk with a meta-data index.

### Data Structures
**Task Structure**:
Each task sent from the manager to a worker is a tuple containing:
- `type`: Enum {EXTRACT, SEND, MERGE, FINISHED}
- `worker_id`: Integer ID of the target worker (for SEND tasks) or the requesting worker (for others).
- `block_id`: Integer ID of the block to extract (for EXTRACT tasks).

**Worker State**:
Each worker maintains a `current_surface`, which is the aggregate of semi-simplified meshes extracted locally and merged from neighbors.

### Algorithm Steps

1.  **Dataset Splitting**: The input volume is recursively split into blocks. These blocks are assigned to compute nodes (workers).
2.  **Worker Loop**: Each worker runs an infinite loop requesting tasks from the manager until a `FINISHED` task is received.
3.  **Task Execution**:
    -   **EXTRACT**: The worker extracts a local isosurface from a sub-block using the Tandem Algorithm. To handle block boundaries, a "time lag" technique is applied: sub-block boundaries (except dataset boundaries) prevent edge collapse in their neighborhood. The extracted surface is appended to the worker's `current_surface`.
    -   **SEND**: The worker sends its `current_surface` to a target worker (specified in the task). After sending, the worker's `current_surface` is emptied.
    -   **MERGE**: The worker merges its `current_surface` with a mesh received from another worker. After merging, the new surface is simplified, and finished components are dumped.
    -   **FINISHED**: The worker stops execution.
4.  **Manager Dispatching**: The worker manager iterates as long as the global surface is not totally extracted. For each worker request, it checks conditions in the following priority order:
    1.  If the worker has a waiting surface (`has_waiting_surface(id)`), send a `MERGE` task.
    2.  Else if the worker has a neighbor worker with restricted adjacency (`has_neighbor_worker(id)`), send a `SEND` task with the target ID.
    3.  Else if there is an unprocessed block with restricted adjacency (`has_unprocessed_block(blockId)`), send an `EXTRACT` task.
    4.  Else, send a `FINISHED` task.
5.  **Component Dumping**: As soon as a surface component is detected as terminated (fully extracted and simplified), it is saved to disk along with meta-data (oriented bounding box, volume, etc.) and cleared from memory.

### Formulas

**Triangle Aspect Ratio**:
The quality of a triangle $t$ with vertices $a, b, c$ is measured by its aspect ratio $\rho(t)$:
$$ \rho(t) = \sqrt{\frac{\lambda_2}{\lambda_1}} $$
where $\lambda_1 \ge \lambda_2$ are the greatest eigenvalues of the inertia matrix $M_t$:
$$ M_t = \frac{1}{3}[(a - \hat{t})(a - \hat{t})^T + (b - \hat{t})(b - \hat{t})^T + (c - \hat{t})(c - \hat{t})^T] $$
and $\hat{t}$ is the centroid of triangle $t$. $\rho(t) = 1$ for equilateral triangles and $\rho(t) = 0$ for flat triangles.

**Mesh Anisotropy**:
The global anisotropy (skewness) of a triangulation $K$ with $n$ triangles is:
$$ anisotropy(K) = 1 - \frac{1}{n} \sum_{t \in K} \rho(t) $$
The range is $0 \le anisotropy(K) \le 1$, where 0 indicates all triangles are equilateral.

### Parameters
-   **$E_0$**: The error threshold used in the tandem algorithm's edge collapse operations. It determines the maximum allowed deviation for collapsed edges.
-   **$\alpha$**: The isotropy parameter in the mesh quality cost function. It balances the cost between shape quality and distance from the original surface. A value of $\alpha = 0.4$ is recommended as a good compromise. Values $> 0.9$ ignore shape criteria, leading to degenerate triangles rejected by $E_0$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Worker Algorithm
def worker(E0: float):
    finished = False
    current_surface = Mesh()
    
    while not finished:
        task = get_task()
        
        if task.type == EXTRACT:
            # Extract surface from sub-block using Tandem Algorithm
            # Apply time lag at sub-block boundaries to prevent invalid edge collapses
            local_mesh = tandem(E0, task.block_id)
            current_surface = merge(current_surface, local_mesh)
            
        elif task.type == SEND:
            # Send current surface to target worker
            send_mesh(task.target, current_surface)
            current_surface = Mesh() # Empty current surface
            
        elif task.type == MERGE:
            # Merge received mesh with current surface
            received_mesh = get_received_mesh()
            current_surface = merge(current_surface, received_mesh)
            
            # Simplify the merged surface
            simplify(current_surface, E0)
            
            # Dump finished components
            dumping(current_surface)
            
        elif task.type == FINISHED:
            # Final merge and dump before stopping
            received_mesh = get_received_mesh()
            current_surface = merge(current_surface, received_mesh)
            simplify(current_surface, E0)
            dumping(current_surface)
            finished = True

# Worker Manager Algorithm
def worker_manager():
    while not global_extraction_finished:
        id = get_task_request_id()
        
        if has_waiting_surface(id):
            send_task(id, MERGE)
        elif has_neighbor_worker(id):
            target_id = get_neighbor_id(id)
            send_task(id, SEND, target_id)
        elif has_unprocessed_block(id):
            block_id = get_best_block(id)
            send_task(id, EXTRACT, block_id)
        else:
            send_task(id, FINISHED)

# Dumping Function
def dumping(surface: Mesh):
    for component in surface.components:
        if is_component_finished(component):
            save_to_disk(component)
            clear_component(surface, component)

# Helper Functions for Manager
def has_waiting_surface(id: int) -> bool:
    # Returns true if worker 'id' has a surface waiting to be merged
    return surface_in_queue(id) is not None

def has_neighbor_worker(id: int) -> int:
    # Returns ID of a worker dealing with a node fulfilling restricted adjacency
    # with the querying worker's aggregate surfaces, or false if none
    neighbor = find_adjacent_worker(id)
    return neighbor

def has_unprocessed_block(id: int) -> int:
    # Finds the best next block to extract.
    # Prioritizes blocks with restricted adjacency to current aggregate.
    # If none, returns any unprocessed block. If none, returns false.
    block = find_best_block(id)
    return block
```

## Subtopics

### Extended Tandem Algorithm with Dumping
This variant refers to the core sequential extension of the tandem algorithm that introduces the component dumping mechanism. It processes cross-sections sequentially, activating them one by one, simplifying the mesh, and dumping finished components. The loop runs from $k=1$ to $N-1$ (number of cross-sections), activating cross-section $k$, simplifying with $E_0$, and dumping. Finally, it activates all remaining components with infinity and simplifies again. This variant is the foundation for the parallel version but lacks the distributed task dispatching and block splitting [S1].

## Operating characteristics
-   **Time Complexity**: The parallel version significantly reduces extraction time compared to the sequential tandem algorithm. On a seismic dataset of size $1626 \times 7028 \times 2000$ (40 GB), extraction takes 5 minutes with 56 processors, compared to over three hours for the sequential tandem algorithm [S1].
-   **Memory Complexity**: By dumping completed components and processing blocks, the algorithm minimizes memory consumption. It avoids storing the entire isosurface in memory, which is critical for large datasets where the sequential algorithm fails due to memory limits (e.g., generating 1,250,000 triangles per layer in noisy datasets) [S1].
-   **Scalability**: The algorithm scales well with the number of processors, limited primarily by the communication overhead between workers and the manager. The hierarchical block splitting allows efficient load balancing [S1].
-   **Mesh Quality**: The parallel algorithm produces a mesh with equivalent quality (anisotropy variation) to the sequential approach. The use of time lag at block boundaries ensures that the mesh quality is not degraded by the parallel processing [S1].
-   **Strengths**: Handles very large datasets that exceed RAM; reduces memory footprint via dumping; maintains high mesh quality; enables interactive exploration via component indexing.
-   **Limitations**: Requires a manager-worker infrastructure; complex task scheduling logic; performance depends on the distribution of surface components across blocks.

## Background
The Extended Tandem Algorithm was introduced by Guilhem Dupuy and Bruno Jobard in 2009 [S1]. It addresses the limitations of the original Tandem Algorithm [3675-tandem-algorithm.md], which, while effective for large datasets, suffered from limited scalability on very large, noisy datasets due to high memory consumption during queue updates and simplification. The extension was motivated by the need to process geophysical datasets (such as seismic cubes) that are often large and contain numerous small, disconnected components. The method builds on the tandem algorithm's simplification strategy and incorporates component dumping and parallel block processing to enable the extraction of isosurfaces from datasets exceeding available memory.

## Sources
[S1] Parallel extraction and simplification of large isosurfaces using an extended tandem algorithm, https://doi.org/10.1016/j.cad.2009.04.016, fetched 2026-10-26
