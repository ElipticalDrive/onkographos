# Manager/Workers Parallel Implementation

Also called: LSF-HPC Cluster Implementation, LSF-HPC worker model, Master/Slave Worker Architecture, Task-based parallelism

## Overview
The Manager/Workers Parallel Implementation is a task-based parallel architecture designed for extracting, merging, and dumping isosurfaces on load-balanced clusters where processor availability varies over time [S1, S2]. It solves the problem of distributing independent geometric extraction tasks across distant workers while maintaining global mesh consistency through a central coordinator [S1]. Unlike sibling methods that rely on static data decomposition or specific GPU primitives, this approach uses a dynamic task scheduler (e.g., Platform LSF-HPC) to dispatch work, allowing the system to adapt to fluctuating cluster resources [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset decomposed into sub-blocks; a set of $N$ worker processes with unique IDs; a Worker Manager process.
*   **Output:** A unified, simplified isosurface mesh.

**Data Structures:**
*   **Task:** A message object containing:
    *   `type`: Enum {EXTRACT, SEND, MERGE, FINISHED}.
    *   `worker_id`: Integer, the unique ID of the requesting worker.
    *   `target_id`: Integer, present only if `type` is SEND or MERGE, indicating the destination worker.
    *   `block_id`: Integer, present only if `type` is EXTRACT, identifying the sub-block to process.
*   **Worker State:** Each worker maintains a local surface mesh (initially empty). Workers do not track global state or adjacent block processing status [S2].
*   **Manager State:** The Manager maintains a global view of:
    *   Extracted blocks.
    *   Worker surface queues (which workers have waiting surfaces).
    *   Neighbor relationships (topological adjacency of blocks assigned to workers).

**Parameters:**
*   **Task Scheduler:** Platform LSF-HPC is cited as the dispatch mechanism [S1, S2].
*   **Worker Count:** Can range from 1 (sequential mode) to $N$ (parallel mode) [S2].

**Algorithm Steps:**

1.  **Initialization:** The Worker Manager initializes the global state. Workers start in an infinite loop, requesting tasks [S1, S2].
2.  **Task Request:** A worker $W_i$ sends a request to the Manager.
3.  **Manager Decision Logic:** The Manager executes the following priority logic for worker $W_i$ [S1]:
    *   If $W_i$ has a waiting surface (from a previous SEND or MERGE):
        *   Send `MERGE` task to $W_i$. The target is the worker holding the adjacent surface component to merge with.
    *   Else if $W_i$ has neighboring workers (topological adjacency exists in unmerged surfaces):
        *   Send `SEND` task to $W_i$ with `target_id` set to the neighbor.
    *   Else if there are un-extracted blocks in the dataset:
        *   Send `EXTRACT` task to $W_i$ with `block_id` set to the next available sub-block.
    *   Else:
        *   Send `FINISHED` task to $W_i$.
4.  **Worker Execution:** Upon receiving a task, the worker executes the corresponding action [S1]:
    *   **EXTRACT:** Execute the `tandem(E0)` algorithm on the specified sub-block. The `tandem` algorithm extracts the isosurface. To handle boundaries, a "time lag" method is applied to prevent edge collapse at sub-block borders (except dataset borders), preserving vertices on borders to ensure better triangulation quality [S1]. The resulting mesh is appended to the worker's current surface.
    *   **SEND:** Send the current surface mesh to the `target_id` worker. The worker then empties its local surface buffer.
    *   **MERGE:** Receive a surface from another worker (or the manager). Merge the received mesh with the current surface. Simplify the combined surface and dump finished components (triangles that are no longer adjacent to any active block) to reduce memory usage [S1].
    *   **FINISHED:** Set the internal `finished` flag to true and terminate the loop.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def worker_loop(worker_id, manager_interface):
    """
    Infinite loop for a worker process.
    manager_interface: Object providing get_task() and send_surface(target_id, mesh)
    """
    finished = False
    current_surface = EmptyMesh()

    while not finished:
        task = manager_interface.get_task(worker_id)
        
        if task.type == "EXTRACT":
            # Extract isosurface from sub-block defined in task.block_id
            # Apply time lag method on borders to preserve edge quality
            mesh = tandem_extract(task.block_id)
            current_surface.append(mesh)
            
        elif task.type == "SEND":
            # Send current surface to target worker
            manager_interface.send_surface(task.target_id, current_surface)
            current_surface = EmptyMesh()
            
        elif task.type == "MERGE":
            # Receive surface from another worker (handled by manager interface)
            incoming_mesh = manager_interface.receive_surface(worker_id)
            # Merge and simplify
            current_surface.merge(incoming_mesh)
            current_surface.simplify()
            # Dump finished components (implementation detail [G])
            current_surface.dump_finished()
            
        elif task.type == "FINISHED":
            finished = True

def manager_loop(dataset, workers, scheduler_interface):
    """
    Central coordinator managing task distribution.
    """
    # Global state
    extracted_blocks = set()
    worker_surfaces = {w: None for w in workers} # Tracks if worker has pending surface
    # Neighbor map: maps worker_id to list of neighbor worker_ids for surface merging
    neighbor_map = build_neighbor_map(dataset, workers) 
    
    finished_workers = set()

    while len(finished_workers) < len(workers):
        # Get ID of worker requesting a task
        request_id = scheduler_interface.get_request()
        
        if request_id in finished_workers:
            continue
            
        # Priority 1: Merge if worker has waiting surface
        if worker_surfaces[request_id] is not None:
            # Find a neighbor to merge with
            target = find_merge_target(request_id, neighbor_map, worker_surfaces)
            if target:
                scheduler_interface.send_task(request_id, "MERGE", target)
                continue
                
        # Priority 2: Send surface to neighbor if no merge target but neighbors exist
        if request_id in neighbor_map and len(neighbor_map[request_id]) > 0:
            # Send to a neighbor that doesn't have a surface yet or needs it
            target = find_send_target(request_id, neighbor_map, worker_surfaces)
            if target:
                scheduler_interface.send_task(request_id, "SEND", target)
                continue
                
        # Priority 3: Extract new block
        if has_unextracted_blocks(dataset, extracted_blocks):
            block = get_next_block(dataset, extracted_blocks)
            scheduler_interface.send_task(request_id, "EXTRACT", block)
            continue
            
        # Priority 4: Finish
        scheduler_interface.send_task(request_id, "FINISHED")
        finished_workers.add(request_id)
```

## Subtopics

### Python Multiprocessing with Dynamic Load Balancing
A variant implementation using Python's `multiprocessing` module for dynamic load balancing on a single HPC node [S3, unofficial]. It starts more processes than available processors to allow the OS scheduler to map them dynamically, or uses a master process to dynamically assign work to available workers [S3, unofficial]. Data is distributed using a "sort last" strategy [S3, unofficial]. Pitfalls include OS-defined process limits (e.g., on 64-core machines), lack of shared memory with NumPy arrays complicating load balancing, and limited RAM compared to multi-node MPI implementations, which reduces data caching potential [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Dependent on the `tandem` extraction algorithm and the efficiency of the Manager's decision logic. The Manager's loop runs continuously, but task dispatch is event-driven [S1].
*   **Memory Complexity:** Workers store only their local surface components. The Manager stores global state (block status, neighbor maps) which is lightweight compared to mesh data [S1].
*   **Performance:** The architecture is designed for load-balanced clusters where processor availability varies [S1, S2]. It handles large datasets by dumping finished mesh components, reducing peak memory usage [S1].
*   **Strengths:** Adapts to time-varying processor allocation; supports single-worker sequential execution without modification [S2]; effective for large isosurface extraction [S1].
*   **Limitations:** Requires a central Manager, creating a potential bottleneck if not distributed; relies on external task schedulers (LSF-HPC) for dispatch [S1, S2].

## Background
The method was proposed for extracting isosurfaces from very large geophysical datasets [S1]. It was developed to address the challenges of parallel execution on clusters with dynamic resource allocation, specifically using Platform LSF-HPC as the task scheduler [S1, S2]. The approach extends the tandem algorithm for isosurface extraction, incorporating simplification and merging strategies to handle distributed mesh assembly [S1, S2].

## Sources
[S1] Isosurface extraction and interpretation on very large datasets in geophysics, https://inria.hal.science/inria-00277070v1, fetched 2026-10-26
[S2] Parallel extraction and simplification of large isosurfaces using an extended tandem algorithm, https://doi.org/10.1016/j.cad.2009.04.016, fetched 2026-10-26
[S3] Volume Rendering of AMR Simulations, https://doi.org/10.48550/arxiv.1210.8014, fetched 2026-10-26 (unofficial)
