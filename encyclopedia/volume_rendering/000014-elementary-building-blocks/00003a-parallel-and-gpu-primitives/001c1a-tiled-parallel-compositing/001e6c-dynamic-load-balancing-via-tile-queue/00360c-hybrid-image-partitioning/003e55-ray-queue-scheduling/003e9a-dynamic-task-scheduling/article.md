# Dynamic Task Scheduling

Also called: Deferred Work Scheduling, Dynamic Ray-Task Scheduling, Job Scheduling (Tick Events), Scheduler Tick Events

## Overview
Dynamic Task Scheduling is a load-balancing method that dynamically schedules subvolumes (flex-bricks) or ray tasks within each render node [S1, unofficial]. It addresses the NP-Complete problem of determining the optimum order in which cells are cached by adaptively assigning work using ray queues and neighbor communication [S1, unofficial]. The method operates in distributed ray tracing contexts and as a deferred workload mechanism in game engines [S1, unofficial][S2, unofficial]. For GPU rendering of Kernel-Reflection Sequences (KRS), it breaks ray evaluation into variable-length tasks to handle differing convergence times [S3].

## How it works
**Inputs:** Massive volume dataset partitioned into render nodes; flex-block dependency graph; initial ray set [S1, unofficial].
**Outputs:** Rendered image contributions from each node [S1, unofficial].
**Data Structures:**
*   **Ray Queue:** Holds pending rays for each cell [S1, unofficial].
*   **Flex-Block Dependency Graph:** Determines the next neighbor to enter, selected by the direction of the ray segment [S1, unofficial].
*   **Task Buffer:** Holds tasks for each component type in KRS models [S3].
*   **Per-Component Free Distances:** Array storing the remaining distance to travel for each ray component [S3].
*   **Sorted Index:** Small sorted index for faster minimum selection among components [S3].

**Steps (Distributed Ray Tracing):**
1. Each node calculates the ray contributions for the portion of data statically assigned to it [S1, unofficial].
2. Initialize a ray queue for each cell in the local partition to hold pending rays [S1, unofficial].
3. Process one cell at a time. When a cell is in main memory, trace all rays currently in its queue through the cell's data [S1, unofficial].
4. If a spawned ray exits the current cell, determine the next intersecting cell using the dependency graph and place the ray on that cell's queue [S1, unofficial].
5. Each node communicates ray information to neighboring nodes to update their respective queues [S1, unofficial].
6. Repeat until all rays have converged or exited the volume [S1, unofficial].

**Steps (GPU Scheduling for KRS):**
1. Break ray evaluation into tasks, each evaluating a few sphere tracing iterations [S3].
2. Maintain per-component free distances for every ray and a small sorted index for faster minimum selection [S3].
3. Before every turn, for every ray, choose the component with the minimal free distance [S3].
4. Insert a task into the respective buffer or store the intersection point if the ray has converged [S3].
5. Process all task buffers sequentially [S3].
6. When a task finishes, update the free distance and the index [S3].
7. For primary rays, exploit coherence by performing the first few steps of sphere tracing at a lower resolution, advancing the current point only as far as the frustum of the low-res pixel is entirely covered by the unbounding sphere [S3].
8. Repeat until all rays are converged, then perform shading for all intersection points in a deferred manner [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Distributed Ray Tracing with Dynamic Task Scheduling
def dynamic_task_scheduling_ray_tracing(local_cells, dependency_graph, initial_rays):
    ray_queues = {cell: [] for cell in local_cells}
    for ray in initial_rays:
        ray_queues[dependency_graph.get_start_cell(ray)].append(ray)
    converged_points = []
    while any(ray_queues.values()):
        next_cell = dependency_graph.select_next_cell(ray_queues)
        cell_data = load_cell_data(next_cell)
        next_updates = {}
        for ray in ray_queues[next_cell]:
            hit, exit_point, remaining = trace_ray(ray, cell_data)
            if hit:
                converged_points.append((ray.id, hit))
            elif remaining is not None:
                next_cell_id = dependency_graph.get_next_cell(ray, exit_point)
                if next_cell_id not in local_cells:
                    send_ray_to_neighbor(next_cell_id, remaining)
                else:
                    next_updates.setdefault(next_cell_id, []).append(remaining)
        ray_queues[next_cell] = []
        for cell_id, rays in next_updates.items():
            ray_queues[cell_id].extend(rays)
    return converged_points

# Pseudocode for GPU Scheduling for KRS
def gpu_krs_scheduling(rays, krs_models):
    task_buffers = {comp: [] for comp in krs_models}
    free_distances = {ray.id: {comp: ray.length for comp in krs_models} for ray in rays}
    sorted_index = [(ray.id, comp, free_distances[ray.id][comp]) for ray in rays for comp in krs_models]
    sorted_index.sort(key=lambda x: x[2])
    converged_points = {}
    for ray in rays:
        if ray.is_primary:
            low_res_step = perform_low_res_sphere_tracing(ray)
            ray.current_point = low_res_step.point
            ray.remaining_length = low_res_step.remaining_length
    while any(task_buffers.values()):
        for comp, buffer in task_buffers.items():
            new_buffer = []
            for task in buffer:
                result = execute_task(task, krs_models[comp])
                if result.converged:
                    converged_points.setdefault(task.ray.id, []).append(result.point)
                else:
                    free_distances[task.ray.id][comp] = result.remaining
                    sorted_index = [(r.id, c, free_distances[r.id][c]) for r in rays for c in krs_models]
                    sorted_index.sort(key=lambda x: x[2])
                    next_comp = sorted_index[0][1]
                    new_buffer.append(create_task(task.ray, next_comp, iterations=5))
            task_buffers[comp] = new_buffer
    return deferred_shading(converged_points)
```

## Subtopics

### GPU Scheduling for KRS
A task-based GPU scheduling scheme to handle variable sphere tracing convergence times and compound KRS models with different distance evaluation code [S3]. It breaks ray evaluation into small tasks, uses task buffers per component, and exploits coherence for primary rays by using low-resolution sphere tracing [S3]. It is chosen when rendering complex procedural models with varying convergence rates on GPU hardware [S3].

## Operating characteristics
*   **Time Complexity:** Depends on the number of rays and the depth of traversal; dynamic scheduling reduces idle time but adds overhead for queue management and communication [S1, unofficial][S3].
*   **Memory Complexity:** Requires storage for distributed ray queues or GPU task buffers, scaling with the number of active rays and KRS components [S1, unofficial][S3].
*   **Performance:** Improves load balance in distributed systems by adapting to data density and ray behavior [S1, unofficial]. On GPU, it prevents thread divergence by breaking long rays into smaller tasks [S3].
*   **Strengths:** Handles variable workloads effectively; reduces cache misses in distributed systems by following data locality; improves GPU utilization for complex models [S1, unofficial][S3].
*   **Limitations:** Adds implementation complexity; requires communication overhead in distributed settings; task management overhead on GPU [S1, unofficial][S3].
*   **When to Use:** When static load balancing is insufficient due to uneven data distribution or variable ray convergence times [S1, unofficial][S3].
*   **When Not to Use:** For simple, uniform volumes where static partitioning is sufficient and overhead is not justified [G].

## Background
Dynamic Task Scheduling originated in massive data management for distributed volume visualization to address the NP-Complete problem of optimal cell caching order using dynamic scheduling of flex-bricks [S1, unofficial]. The approach was extended to GPU rendering for Kernel-Reflection Sequences (KRS) to handle variability in sphere tracing convergence [S3]. In game engine architectures, a similar deferred workload mechanism is implemented via a Scheduler class that defers actions to future game ticks using tick events and jobs [S2, unofficial].

## Sources
[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S2] DigBuild: Sandbox Voxel Game and Engine Using Vulkan, https://gredos.usal.es/handle/10366/149914, fetched 2026-10-31 (unofficial)
[S3] Kernel-Reflection Sequences, https://doi.org/10.2312/stag.20161364, fetched 2026-10-26
