# Chamfer-L1 Distance Computation
Also called: Distance-Aided Ray Marching

## Overview
Chamfer-L1 Distance Computation estimates the Chamfer distance between a predicted SDF surface and a ground-truth mesh by using sphere tracing to sample points on the SDF surface [S1]. It sets itself apart from standard sphere tracing by repurposing the ray-marching process to generate surface samples for metric evaluation rather than direct rendering [S1].

## How it works
The method combines sphere tracing with a sampling strategy to generate points on the surface of an SDF-defined object [S1]. These points are then used to compute the Chamfer distance, a metric measuring the similarity between two sets of points [S1].

**Inputs and Outputs:**
*   **Input:** A bounding volume $B$, a predicted SDF function $f(\mathbf{p})$, a ground-truth mesh $M_{gt}$, and a target number of samples $N$ [S1].
*   **Output:** A set of sampled points $P_{pred}$ on the predicted surface and the computed Chamfer distance $D_{Chamfer}$ between $P_{pred}$ and points sampled from $M_{gt}$ [S1].

**Data Structures:**
*   **Bounding Volume $B$:** A 3D box or sphere enclosing the object [G].
*   **Sample Points:** A list of 3D coordinates $\mathbf{p}_i$ [G].
*   **Ray Data:** Origin $\mathbf{o}$, direction $\mathbf{d}$ (unit vector), current distance $t$ [G].

**Numbered Steps:**
1.  **Initialization:** Define the bounding volume $B$ and initialize an empty list $P_{pred}$ for sampled points [G].
2.  **Ray Generation:** Uniformly sample $2^{17} = 131,072$ points in the bounding volume $B$, and assign each point a random spherical direction $\mathbf{d}$ [S1].
3.  **Sphere Tracing:** For each ray $(\mathbf{o}, \mathbf{d})$:
    *   Initialize $t = 0$ [G].
    *   Loop while $t < t_{max}$ and steps $< max\_steps$:
        *   Evaluate $\mathbf{p} = \mathbf{o} + t\mathbf{d}$ [G].
        *   Compute $d = f(\mathbf{p})$ [S1].
        *   If $d < \epsilon$, the ray has hit the surface [G].
        *   Advance $t \leftarrow t + \max(d, 10^{-6})$ to prevent zero or negative steps that break sphere tracing [G].
    *   If a hit occurred, record $\mathbf{p}$ in $P_{pred}$ [S1].
4.  **Sampling Refinement:** If $|P_{pred}|$ is less than the minimum required number of points, repeat steps 2-3 with new random rays until the threshold is met [S1].
5.  **Ground Truth Sampling:** Uniformly sample points $P_{gt}$ from the ground-truth mesh $M_{gt}$ (exact sampling method not specified in sources) [G].
6.  **Chamfer Distance Calculation:** Compute the Chamfer distance between $P_{pred}$ and $P_{gt}$ using squared Euclidean distance as implemented in PyTorch3D [S1].

**Formulas:**
*   **Ray Marching Step:** $t_{new} = t_{old} + f(\mathbf{o} + t_{old}\mathbf{d})$ [S1].
*   **Chamfer Distance:** $D_{Chamfer} = \frac{1}{|P_{pred}|} \sum_{\mathbf{p} \in P_{pred}} \min_{\mathbf{q} \in P_{gt}} \|\mathbf{p} - \mathbf{q}\|^2 + \frac{1}{|P_{gt}|} \sum_{\mathbf{q} \in P_{gt}} \min_{\mathbf{p} \in P_{pred}} \|\mathbf{q} - \mathbf{p}\|^2$ [S1].

**Parameters:**
*   **Initial Samples:** $2^{17} = 131,072$ rays [S1].
*   **Convergence Threshold $\epsilon$:** Not given in the sources [G].
*   **Max Distance $t_{max}$:** Not given in the sources [G].

**Edge Cases and Pitfalls:**
*   **Sampling Bias:** Random spherical directions may miss thin structures or concavities if the bounding volume is large relative to the object [G].
*   **Precision:** Floating-point errors in SDF evaluation can cause rays to miss the surface or oscillate near it [G].
*   **Mesh Extraction Error:** If ground truth is obtained via Marching Cubes from an SDF, additional error is introduced [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def chamfer_l1_distance_computation(sdf, mesh_gt_points, num_initial_rays=131072, epsilon=1e-4, max_steps=100, t_max=10.0):
    """
    Compute Chamfer distance between SDF surface and ground truth mesh using sphere tracing.
    
    Args:
        sdf: Callable SDF function f(p)
        mesh_gt_points: List of 3D coordinates sampled from ground truth mesh
        num_initial_rays: Number of initial rays to cast [S1]
        epsilon: Convergence threshold for surface hit [G]
        max_steps: Maximum sphere tracing steps per ray [G]
        t_max: Maximum ray distance [G]
        
    Returns:
        chamfer_distance: Scalar Chamfer distance
    """
    # 1. Initialize
    pred_points = []
    
    # 2. Ray Generation: Uniformly sample points in bounding volume B (assumed unit cube [-0.5, 0.5]^3)
    # Assign each a random spherical direction [S1]
    origins = [(random.uniform(-0.5, 0.5), random.uniform(-0.5, 0.5), random.uniform(-0.5, 0.5)) for _ in range(num_initial_rays)]
    directions = [(random.gauss(0, 1), random.gauss(0, 1), random.gauss(0, 1)) for _ in range(num_initial_rays)]
    # Normalize directions to unit vectors
    directions = [(d[0]/math.sqrt(sum(x*x for x in d)), d[1]/math.sqrt(sum(x*x for x in d)), d[2]/math.sqrt(sum(x*x for x in d))) for d in directions]
    
    # 3. Sphere Tracing
    for i in range(num_initial_rays):
        o = origins[i]
        d = directions[i]
        t = 0.0
        hit = False
        for _ in range(max_steps):
            p = (o[0] + t*d[0], o[1] + t*d[1], o[2] + t*d[2])
            dist = sdf(p)
            if dist < epsilon:
                pred_points.append(p)
                hit = True
                break
            # Prevent zero or negative steps that break sphere tracing
            step_size = max(dist, 1e-6)
            t += step_size
            if t > t_max:
                break
        if hit:
            continue
            
    # 4. Refinement: Repeat ray generation/marching until minimum points obtained [S1]
    # (Implementation omitted as loop condition depends on external target count)
    
    # 5. Ground Truth Sampling: Sample points P_gt from mesh_gt [G]
    # (Method not specified in sources)
    
    # 6. Chamfer Distance Calculation
    def euclidean_dist_sq(a, b):
        return sum((x - y)**2 for x, y in zip(a, b))

    def min_dist_sq(point, points_list):
        return min(euclidean_dist_sq(point, q) for q in points_list)

    if not pred_points or not mesh_gt_points:
        return float('inf')
        
    forward_sum = sum(min_dist_sq(p, mesh_gt_points) for p in pred_points)
    backward_sum = sum(min_dist_sq(q, pred_points) for q in mesh_gt_points)
    
    chamfer = (forward_sum / len(pred_points)) + (backward_sum / len(mesh_gt_points))
    return chamfer
```

## Subtopics

### [Sphere Assisted Raymarching](4497-sphere-assisted-raymarching.md)
Stores the radius of the largest empty sphere centered at each voxel in a separate shape texture to take large steps through empty space. It sets itself apart by utilizing the exact Euclidean distance to the nearest solid voxel rather than the L1 distance, providing optimal step sizes in empty regions at the cost of higher preprocessing complexity.

### [STEVE 4D Normal Vector Computation](16340-steve-4d-normal-vector-computation.md)
Computes the oriented normal vector of a 4D simplex defined by four space-time events for relativistic heavy-ion collision simulations. It distinguishes itself by handling the mixed spatial-temporal signature of Minkowski space.

### [2-Pass Distance Transformation](20316-2-pass-distance-transformation.md)
Estimates wall thickness by computing the distance transform of a part relative to the die. The wall thickness is approximated as twice the maximum distance value found in the part voxels.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S + M \log M + P \log P)$, where $N$ is the number of rays, $S$ is the average number of sphere tracing steps, $M$ is the number of ground truth samples, and $P$ is the number of predicted samples [G]. The nearest neighbor search dominates if $M, P$ are large [G].
*   **Memory Complexity:** $O(M + P)$ to store the sampled points [G].
*   **Performance:** The method is computationally expensive due to the large number of ray casts ($2^{17}$) and SDF evaluations [S1].
*   **Strengths:** Provides a robust metric for training implicit neural representations against mesh ground truth [S1].
*   **Limitations:** Sensitive to SDF quality and sampling bias [G].
*   **When to use:** When evaluating or training implicit neural fields (like NGLOD) against mesh ground truth [S1].
*   **When not to use:** For real-time rendering where speed is critical; use standard sphere tracing for visualization instead [G].

## Background
The technique is described in the context of the "Neural Geometric Level of Detail" (NGLOD) paper [S1]. The visualization of sphere tracing, where the distance between steps is pictured as the radius of a sphere touching at least one object, originates from ray casting literature [S2, unofficial]. The specific application to Chamfer distance computation for neural implicit shapes is a modern development in 3D deep learning [S1].

## Sources
[S1] Neural Geometric Level of Detail: Real-time Rendering with Implicit 3D Shapes, https://doi.org/10.1109/cvpr46437.2021.01120, fetched 2026-10-26
[S2] http://www.hugi.scene.org/online/hugi37/hugi%2037%20-%20coding%20adok%20on%20ray%20casting,%20ray%20tracing,%20ray%20marching%20and%20the%20like.htm, http://www.hugi.scene.org/online/hugi37/hugi%2037%20-%20coding%20adok%20on%20ray%20casting,%20ray%20tracing,%20ray%20marching%20and%20the%20like.htm, fetched 2026-10-26 (unofficial)
