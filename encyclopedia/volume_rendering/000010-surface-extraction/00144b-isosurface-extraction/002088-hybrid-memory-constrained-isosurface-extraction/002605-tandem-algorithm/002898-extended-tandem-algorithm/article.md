# Extended Tandem Algorithm

Also called: Parallel Bloc-wise Extension, Parallel Tandem Algorithm, Parallel Tandem Algorithm with Component Dumping, Parallel tandem with dumping

## Overview
The Extended Tandem Algorithm is a parallel, bloc-wise extension of the Tandem Algorithm [3675-tandem-algorithm.md] designed to extract and simplify isosurfaces from very large volumetric datasets while minimizing memory consumption. It solves the scalability bottleneck of the sequential Tandem Algorithm, where queue updates and simplification steps become too memory-intensive for high-resolution data (e.g., generating millions of triangles per layer) [S1, S2]. Unlike the parent method, which processes the entire volume in a single pass, this method splits the dataset into blocks, distributes them across computing nodes, and employs a component dumping mechanism to persist finalized surface components to disk [S1, S2]. This allows the extraction process to continue on sub-parts of the dataset without losing previously computed geometry, making it suitable for geophysical datasets with numerous small, disconnected components [S1, S2].

## How it works
The algorithm operates using a worker-manager architecture. The dataset is split into blocks and distributed to computing nodes (workers). Each worker extracts and semi-simplifies a local isosurface, then merges it with adjacent semi-simplified isosurfaces received from other workers. The core simplification logic relies on the Tandem Algorithm's edge collapse strategy with a "time lag" mechanism to prevent artifacts near the advancing front [S1, S2].

### Data Structures
*   **Priority Queue Q**: Stores edge candidates for collapse, ordered by their global cost $\varepsilon_\alpha(c)$.
*   **Priority Queue W**: Stores blocked edges, ordered by their reach value. Used to delay collapses near the advancing front [S1].
*   **Task Structure**: Composed of a task type and a worker ID. Task types are:
    *   `EXTRACT`: Extract surface from a sub-block.
    *   `SEND`: Send mesh to a target worker.
    *   `MERGE`: Merge received mesh with current surface.
    *   `FINISHED`: Stop worker [S2].

### Simplification Metrics
The edge collapse cost $\varepsilon_\alpha(c)$ balances shape measure and mesh isotropy [S1, S2]:

$$ \varepsilon_\alpha(c) = c^T \left[ (1 - \alpha) \frac{H_c}{W_c} + \alpha \frac{G_c}{W} \right] c $$

Where:
*   $\alpha$ is the isotropy parameter (typically 0.4) [S1, S2].
*   $H_c$ is a positive definite matrix related to the shape measure.
*   $W_c$ is the sum of areas of neighboring triangles $U_c$.
*   $G_c$ is a matrix related to the mesh isotropy criterion.
*   $W$ is a normalization factor [S1].

The **shape measure** $h_c(x)$ for a point $x$ is:
$$ h_c(x) = \frac{1}{W_c} \sum_{t \in U_c} w_t d^2(x, P_t) = \frac{1}{2} x^T H_c x $$
where $P_t$ is the plane of triangle $t$, $w_t$ is its area, and $U_c$ is the patch of neighboring triangles [S1].

The **mesh isotropy** criterion $g_c(x)$ is:
$$ g_c(x) = \sum_{t \in S_{ab}} w_t (\|x - \hat{t}\|^2 + \text{avg}(t)) = x^T G_c x $$
where $S_{ab}$ is the set of triangles containing vertices $a, b$ or both, $\hat{t}$ is the barycenter of triangle $t$, and $\text{avg}(t) = \frac{1}{2}(\|p\|^2 + \|q\|^2 + \|r\|^2)$ with $p, q, r$ vectors from $\hat{t}$ to vertices [S1].

A candidate edge collapse is rejected if its shape measure $\varepsilon_0(c)$ exceeds the maximum shape error threshold $E_0$ [S1, S2].

### Time Lag and Reach
To prevent artifacts, the algorithm uses a time lag mechanism. The **reach** of an edge collapse $c$ is defined as:
$$ \text{reach}(c) = \text{height}(c) + \text{rad}(c) $$
where $\text{height}(c) = (\text{height}(a) + \text{height}(b))/2$ and $\text{rad}(c) = (\|a - b\| + \text{rad}(a) + \text{rad}(b))/2$. For new vertices, $\text{rad}=1$. An edge is prevented from collapsing if $\text{reach}(c) \ge \text{rank}(\text{front})$, where $\text{rank}(\text{front})$ is the maximum rank (coordinate along extraction direction) of the extracted layers [S1]. Blocked edges are stored in queue $W$ and moved to $Q$ via the `activate` function when their reach is less than the current front rank [S1].

### Worker Loop
Workers run an infinite loop requesting tasks from the worker manager until a `FINISHED` task is received [S2].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def worker(E0: float):
    finished = False
    while not finished:
        task = get_task()
        if task.type == EXTRACT:
            # Extract surface from sub-block using tandem algorithm
            # Sub-block boundaries prevent edge collapse in neighborhood using time lag
            tandem(E0)
            # Appended to worker's current surface
        elif task.type == SEND:
            # Send current mesh to target worker
            send_surface(task.target)
            # Worker's current surface is emptied
        elif task.type == MERGE:
            # Merge received mesh with current surface
            merge_surface(get_surface(), E0)
            # Simplify new current surface
            # Dump finished components to disk
        elif task.type == FINISHED:
            # Merge any remaining surface
            merge_surface(get_surface(), E0)
            finished = True
```

### Worker Manager Loop
The manager iterates as long as the global surface is not totally extracted. For each worker request, it checks conditions in order [S2]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def worker_manager():
    while not finished:
        id = get_task_request_id()
        if has_waiting_surface(id):
            # Worker has a waiting surface sent by other workers
            send_task_to_worker(id, MERGE)
        elif has_neighbor_worker(id):
            # Worker has a neighbor worker with restricted adjacency
            target_id = get_neighbor_worker_id(id)
            send_task_to_worker(id, SEND, target_id)
        elif has_unprocessed_block(id):
            # Find best next block fulfilling restricted adjacency
            block_id = get_unprocessed_block(id)
            send_task_to_worker(id, EXTRACT, block_id)
        else:
            send_task_to_worker(id, FINISHED)
```

Helper functions:
*   `has_waiting_surface(id)`: Returns true if worker `id` has a waiting surface, else false [S2].
*   `has_neighbor_worker(id)`: Returns the ID of a worker dealing with a node fulfilling restricted adjacency requirements with the querying worker's aggregate surfaces, else false [S2].
*   `has_unprocessed_block(id)`: Finds the best next block. Prioritizes unprocessed blocks fulfilling restricted adjacency with the querying worker's current aggregate. If none, returns any unprocessed block. If none, returns false [S2].

### Tandem Core Steps
The local extraction on each block follows the Tandem Algorithm structure [S1, S2]:
1.  Iterate $k$ from 1 to number of cross-sections - 1:
    *   `extract(k)`: Adds layer $k$ (vertices, edges, triangles from marching cubes between cross-sections $k-1$ and $k$) to current triangulation.
    *   `delay(k)`: Adds all edges of layer $k$ to blocked queue $W$, ordered by reach value.
    *   `activate(k)`: Moves edges from $W$ to $Q$ if their reach value is less than rank of advancing front $k$.
    *   `simplify(E0)`: Empties queue $Q$ by applying consecutive edge collapses.
2.  After loop:
    *   `activate(∞)`: Processes remaining delayed edges.
    *   `simplify(E0)`: Final simplification.

### Merging and Dumping
When merging two semi-simplified isosurfaces, a simplification stage with relaxed edge constraints is applied at the common interface to remove seams [S1, S2]. The algorithm detects the termination of a surface component (when extraction and simplification are complete) and dumps the finalized surface to disk along with a meta-data index (oriented bounding box, volume, etc.) [S1, S2]. This frees memory for further processing [S1].

### Parameters
*   $E_0$: Maximum shape error threshold (positive constant). Typical values depend on dataset noise [S1, S2].
*   $\alpha$: Isotropy parameter. Recommended value is 0.4 [S1, S2].

### Edge Cases and Pitfalls
*   **High Isotropy Parameter**: If $\alpha > 0.9$, the shape criterion is ignored, leading to collapsed points far from the original surface. These are rejected by $E_0$, increasing anisotropy. Solution: Use $\alpha = 0.4$ [S1, S2].
*   **Blocked Edges**: Edges near the advancing front can introduce artifacts if collapsed immediately. The time lag mechanism prevents this [S1].
*   **Memory Bottlenecks**: On noisy datasets (e.g., $1626 \times 7028 \times 2000$), each extraction on advancing fronts generates 1,250,000 triangles, making queue updates and simplification too memory consuming for the sequential Tandem Algorithm. The Extended version mitigates this via dumping [S1, S2].

## Subtopics

### [Extended Tandem Algorithm with Dumping](4618-extended-tandem-algorithm-with-dumping.md)
The Extended Tandem Algorithm with Dumping is a parallel, bloc-wise extension of the Tandem algorithm designed to increase scalability for very large datasets by dumping extracted surface components to disk. It prevents memory exhaustion during queue updates and simplification steps, which are the primary bottlenecks in the standard Tandem algorithm for high-resolution data. Unlike its parent, it introduces a mechanism to persist finalized components to storage, allowing the extraction process to continue on sub-parts of the dataset without losing previously computed geometry.

### [Hierarchical Binary Partitioning](8559-hierarchical-binary-partitioning.md)
Hierarchical Binary Partitioning is a parallel isosurface extraction method that recursively subdivides a volumetric dataset into sub-blocks to enable independent surface extraction and simplified merging. It distinguishes itself from sibling methods by using a recursive binary tree structure based on the dataset's longest dimension, rather than a fixed regular grid, to balance workload and facilitate the aggregation of semi-simplified surfaces.

## Operating characteristics
*   **Time Complexity**: The parallel algorithm significantly reduces extraction time compared to the sequential Tandem Algorithm. On a seismic dataset ($1626 \times 7028 \times 2000$, 40 GB), extraction takes 5 minutes with 56 processors, compared to more than three hours for the sequential algorithm [S2].
*   **Memory Complexity**: The method minimizes memory consumption by processing blocks independently and dumping completed components. It avoids storing the full mesh before simplification, unlike brute-force approaches [S1, S2].
*   **Performance**: On the seismic dataset, the mesh quality is high, with 328 components kept out of 18,111 in the unfiltered version after sorting by geometrical criteria [S2].
*   **Scalability**: The parallel algorithm with time lag produces a mesh with equivalent quality (anisotropy variation is similar) to the sequential tandem algorithm [S2].
*   **Strengths**: Suitable for very large geophysics datasets with numerous small, disconnected components. Reduces memory footprint drastically.
*   **Limitations**: Requires a worker-manager architecture and network communication for merging. Complexity increases with the number of workers and blocks.

## Background
The Tandem Algorithm was introduced by Attali et al. in 2005 to reduce memory requirements by combining isosurface extraction and simplification in one pass [S1, S2]. The Extended Tandem Algorithm was developed to address the limited scalability of the original algorithm for very large datasets, particularly in geophysics [S1, S2]. It was supported by Total company and applied to datasets of size $7000 \times 1600 \times 2000$ [S1, S2].

## Sources
[S1] Isosurface extraction and interpretation on very large datasets in geophysics, https://inria.hal.science/inria-00277070v1, fetched 2026-10-26
[S2] Parallel extraction and simplification of large isosurfaces using an extended tandem algorithm, https://doi.org/10.1016/j.cad.2009.04.016, fetched 2026-10-26
