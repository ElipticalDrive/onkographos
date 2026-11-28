# Warp-based Ray-Fluid Surface Intersection


## Overview
A GPU rendering technique that processes one ray per thread, using warp-level parallelism to traverse the simulation domain and calculate ray-fluid surface intersections [S1]. It avoids inter-thread communication during traversal and uses intra-warp voting to identify potential intersections [S1]. It builds on [VIPER Ray-Traversal and Surface Intersection Algorithm](6590-viper-ray-traversal-and-surface-intersection-algorithm.md).

## How it works
**Inputs:** A queue of rays, each defined by an origin $O$ and direction $D$; a simulation data structure containing cells with particles or field values; a warp size $w$.
**Outputs:** Intersection distances $\lambda$, normals $n$, and colors $C$ for each ray.

**Data Structures:**
*   **Ray Queue:** Stores ray parameters. Processed one ray at a time per thread [S1].
*   **Simulation Domain:** Axis-Aligned Bounding Box (AABB) defined by $C_{min}$ and $C_{max}$ [S1].
*   **Cell Data:** Cells $c$ contain particles or field quantities $\phi(x)$ [S1].

**Steps:**
1.  **Ray Queue Processing:** All rays are stored in a queue. One ray is processed at a time by a single thread [S1].
2.  **Domain Intersection:** Calculate the intersection of the ray with the simulation domain AABB. This yields a close distance $\lambda_{min}$ and far distance $\lambda_{max}$ [S1]. Clamp $\lambda_{min}$ to positive values to get $\lambda_{min}^+ = \max(\lambda_{min}, 0)$ [S1].
3.  **Warp Traversal:** Execute simulation domain intersection and data structure traversal independently on all threads in a warp. No inter-thread communication is required during this phase [S1].
4.  **Synchronization:** Synchronize threads in the warp [S1].
5.  **Potential Intersection Check:** Check if any thread found a cell that can potentially contain fluid surfaces [S1].
6.  **Surface Intersection Calculation:** For the potential intersections, calculate the actual ray-fluid intersection and store results if found [S1]. This involves evaluating a single intersection of a ray with a found cell $\bar{c}$ in parallel using all threads in a warp [S1]. Sequentially process the overall set of potential intersections [S1].

**Subdivision Formula:**
To evaluate the intersection within a found cell $\bar{c}$, the interval $[\max\{\lambda_{min}^{\bar{c}}, 0\}, \lambda_{max}^{\bar{c}})$ is subdivided into $w-1$ segments for a warp size of $w$ [S1]. Each segment $s \in [0, w-1)$ is assigned a starting position $x_s$ [S1]:

$$
x_s = \text{ray}\left( \max\{\lambda_{min}^{\bar{c}}, 0\} + \frac{s}{w-1} \left( \lambda_{max}^{\bar{c}} - \max\{\lambda_{min}^{\bar{c}}, 0\} \right) \right) \quad (34)
$$

Each thread in a warp is assigned one position, with the last thread assigned $x = \text{ray}(\lambda_{max}^{\bar{c}})$ [S1]. Each ray is assigned an associated distance value $\lambda_s$ [S1].

**Cell Intersection Details:**
If a cell $c$ is found, calculate the intersection of the ray with the cell to yield $\lambda_{min}^c$ and $\lambda_{max}^c$ [S1]. The point of ray-cell intersection is $x_i = \text{ray}(\lambda_{min}^c)$ [S1]. The cell center is $x_c = D_{min} + c \circ C_{max}$, with half cell size $r_c = 0.5 C_{max}$ [S1]. The normal $n_c$ is determined by:

$$
n_c = \left[ (1 + \epsilon) d \oslash r_c \right] \quad (33)
$$

where $d = x_i - x_c$, $\epsilon$ is a small positive value, $[\cdot]$ denotes rounding to the nearest integer, and $\oslash$ denotes element-wise division [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def warp_based_ray_fluid_intersection(rays, data_structure, warp_size):
    results = []
    for ray in rays:
        # Step 2: Domain Intersection
        lambda_min, lambda_max = intersect_ray_aabb(ray.origin, ray.direction, data_structure.domain)
        lambda_min_pos = max(lambda_min, 0.0)
        
        # Step 3 & 4: Warp Traversal and Sync
        # Assume warp_threads is a list of thread indices in the current warp
        # Each thread executes traversal independently
        potential_cells = []
        for t_idx in warp_threads:
            # Independent traversal logic [S1]
            cell = traverse_data_structure(ray, lambda_min_pos, lambda_max, data_structure)
            if cell is not None:
                potential_cells.append((t_idx, cell))
        
        sync_warp()
        
        # Step 5: Check for potential intersections
        if potential_cells:
            # Step 6: Calculate actual intersection
            for t_idx, cell in potential_cells:
                # Subdivide interval [S1]
                lambda_min_c, lambda_max_c = intersect_ray_cell(ray, cell)
                interval_start = max(lambda_min_c, 0.0)
                interval_end = lambda_max_c
                
                # Assign positions to threads [S1]
                thread_positions = []
                for s in range(warp_size - 1):
                    t = s / (warp_size - 1)
                    pos = interval_start + t * (interval_end - interval_start)
                    thread_positions.append(pos)
                # Last thread gets the end
                thread_positions.append(interval_end)
                
                # Evaluate intersection in parallel [S1]
                # This part depends on the specific surface representation (particle, cell, or field)
                # For cell representation [S1]:
                intersection_found = False
                for pos in thread_positions:
                    x_i = ray.origin + pos * ray.direction
                    # Calculate normal and check against cell geometry
                    # See formula 33 for normal calculation
                    # If intersection detected, store result
                    if check_cell_intersection(x_i, cell):
                        results.append((ray, pos, cell))
                        intersection_found = True
                        break
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays and the cost of traversal per ray. Warp-level parallelism reduces overhead for potential intersection checks [S1].
*   **Memory Complexity:** Depends on the simulation data structure size. Rays are stored in a queue [S1].
*   **Strengths:** Avoids inter-thread communication during traversal, improving efficiency on GPU architectures [S1].
*   **Limitations:** Requires warp synchronization, which can introduce latency if threads diverge significantly [S1].
*   **When to use:** Suitable for GPU rendering of fluid simulations where ray-cell or ray-particle intersections are frequent [S1].
*   **When not to use:** If the simulation domain is sparse and traversal overhead dominates, other methods might be more efficient [S1].

## Background
Originated in the context of Smoothed Particle Hydrodynamics (SPH) simulation and rendering [S1]. The method is described in "Multi‐Level Memory Structures for Simulating and Rendering Smoothed Particle Hydrodynamics" [S1].

## Sources
[S1] Multi‐Level Memory Structures for Simulating and Rendering Smoothed Particle Hydrodynamics, https://doi.org/10.1111/cgf.14090, fetched 2026-10-28
