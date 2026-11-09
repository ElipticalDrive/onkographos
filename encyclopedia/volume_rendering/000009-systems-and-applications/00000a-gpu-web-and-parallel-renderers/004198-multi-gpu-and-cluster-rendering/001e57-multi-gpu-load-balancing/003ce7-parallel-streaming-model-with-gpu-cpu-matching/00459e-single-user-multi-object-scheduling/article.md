# Single-User Multi-Object Scheduling

Also called: Heavy/Light Farm Assignment, Load-balancing Schedule Policy

## Overview
This method is a scheduling policy for distributing rendering jobs for a single user viewing multiple objects in a large scene [S1, unofficial]. It solves the problem of unbalanced GPU workload and I/O bandwidth bottlenecks by separating objects into "Heavy" (larger-sized) and "Light" (smaller-sized) tasks [S1, unofficial]. It distinguishes itself from uniform resource assignment by dynamically allocating dedicated GPUs to heavy tasks and shared GPUs to light tasks, thereby reducing long tail latency for large objects and preventing I/O saturation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of NGP-NOLF objects $O$ to be rendered for a single user view.
*   **Output:** Assignment of each object $i \in O$ to either `HeavyFarm` or `LightFarm`.

**Data Structures:**
*   `nhit`: Number of rays hit by each object (estimated).
*   `n_pixel`: Estimated pixel quantity on the final screen.
*   `avg_depth`: Average depth of the hit rays.
*   `th_pix_rate`: Threshold for pixels per frame.
*   `th_depth`: Threshold for average depth.

**Steps:**
1.  **Generate Prior Scene:** Use EGL to generate a rough prior scene offscreen to act as an estimator for `nhit` [S1, unofficial]. This step is designed to be negligible in impact on rendering time [S1, unofficial].
2.  **Estimate Metrics:** Calculate `n_pixel` and `avg_depth` from the estimated `nhit` using specific estimators [S1, unofficial].
3.  **Decision Logic:** Evaluate the following condition for each object:
    $$ \text{if } \frac{n\_pixel}{frame} > th\_pix\_rate \text{ OR } avg\_depth < th\_depth $$
    *   If **True**: Assign the object to `HeavyFarm`. Larger-sized objects are assigned here, with one object per GPU [S1, unofficial].
    *   If **False**: Assign the object to `LightFarm`. Smaller-sized objects are assigned here, sharing GPU resources [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def schedule_objects(objects, th_pix_rate, th_depth):
    """
    Assigns objects to HeavyFarm or LightFarm based on estimated workload.
    
    Args:
        objects: List of object instances.
        th_pix_rate: Threshold for pixels per frame.
        th_depth: Threshold for average depth.
        
    Returns:
        heavy_farm: List of objects assigned to HeavyFarm.
        light_farm: List of objects assigned to LightFarm.
    """
    heavy_farm = []
    light_farm = []
    
    for obj in objects:
        # Step 1 & 2: Estimate nhit, n_pixel, and avg_depth
        # Note: The source describes using EGL for a rough prior scene 
        # to estimate nhit, then deriving n_pixel and avg_depth.
        # Implementation detail: This requires a pre-pass or approximation.
        nhit = estimate_nhit(obj) 
        n_pixel = estimate_n_pixel(nhit)
        avg_depth = estimate_avg_depth(nhit)
        
        # Step 3: Decision Logic
        # Condition: n_pixel / frame > th_pix_rate OR avg_depth < th_depth
        # Note: 'frame' is treated as a constant normalization factor or 
        # the denominator in the rate calculation.
        if (n_pixel / frame > th_pix_rate) or (avg_depth < th_depth):
            heavy_farm.append(obj)
        else:
            light_farm.append(obj)
            
    return heavy_farm, light_farm
```

**Parameters:**
*   `th_pix_rate`: Threshold for pixels per frame. Value not given in the sources.
*   `th_depth`: Threshold for average depth. Value not given in the sources.
*   `nhit`: Number of rays hit by each NGP-NOLF object [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **I/O Bandwidth Limits:** Simultaneous image transferring for all objects can exceed the Master Node's I/O bandwidth, leading to unacceptable response times [S1, unofficial].
*   **Unbalanced Workload:** Uniform resource assignment causes waste for small images and long tail latency for large ones [S1, unofficial].

## Subtopics

### [Multi-User Scheduling with Ray Sharing](11550-multi-user-scheduling-with-ray-sharing.md)
Extends the scheduling concept to multiple users viewing the same object from adjacent angles by sharing rendered rays. It categorizes tasks into "Heavy" (shareable) and "Light" (necessary) to amortize geometry costs.

## Operating characteristics
**Time and Memory Complexity:**
*   The total time spent on rendering a frame with multiple objects $T_{frame}$ is expressed as:
    $$ T_{frame} = \max_{i \subset O} (T_i) + \sum_{i \subset O} \frac{D_i}{BW} + C $$
    Where $T_i$ is the rendering time, $D_i$ is the result data size of object $i$, $BW$ is the bandwidth limit of the Master Node, $C$ is the consolidation time, and $O$ is the collection of rendered entities [S1, unofficial].
*   The scheduler aims to minimize $T_{frame}$ by reducing the number of instances in $O$ and choosing a proper subset in the Render Farm [S1, unofficial].

**Measured Performance:**
*   Not given in the sources.

**Scaling:**
*   Uses Infiniband and CUDA-aware MPI for cross-node communication [S1, unofficial].
*   Exploits GPU Direct RDMA optimization on Nvidia Advanced GPUs to expedite readout of rendering results, lowering cross-node overhead [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Reduces long tail latency for large objects by dedicating GPUs; prevents I/O saturation by separating heavy/light tasks; negligible overhead from the estimation phase [S1, unofficial].
*   **Limitations:** Relies on accurate estimation of `nhit`, `n_pixel`, and `avg_depth`; I/O bandwidth remains a bottleneck if the separation logic fails to sufficiently reduce simultaneous transfers [S1, unofficial].

**When to Use:**
*   Use when a single user views multiple objects in a large scene with varying sizes and depths [S1, unofficial].
*   Use when GPU resources are heterogeneous or need to be shared efficiently [S1, unofficial].

**When Not to Use:**
*   Not suitable for multi-user scenarios without ray sharing, as it does not address repetitive rendering labor for adjacent angles [S1, unofficial].

## Background
**Origin:**
*   Introduced in "NEPHELE: A Neural Platform for Highly Realistic Cloud Radiance Rendering" [S1, unofficial].

**History and Development:**
*   The method addresses obstacles in previous solutions where a Master Node gathers and composites frames from individual renderers, which fails due to I/O bandwidth limits and unbalanced GPU workloads [S1, unofficial].

**Where it is used:**
*   Cloud radiance rendering platforms using Neural Graphics Primitives (NGP) and Neural Octree Fields (NOLF) [S1, unofficial].

## Sources
[S1] NEPHELE: A Neural Platform for Highly Realistic Cloud Radiance Rendering, https://arxiv.org/abs/2303.04086v1, fetched 2026-10-27 (unofficial)
