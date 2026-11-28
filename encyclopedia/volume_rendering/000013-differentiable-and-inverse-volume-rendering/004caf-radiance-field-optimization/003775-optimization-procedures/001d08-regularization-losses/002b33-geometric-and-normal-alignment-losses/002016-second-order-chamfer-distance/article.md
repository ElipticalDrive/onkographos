# Second-Order Chamfer Distance

Also called: Second-Order CD

## Overview
Second-Order Chamfer Distance is a supervised geometric loss that measures the discrepancy between a predicted mesh vertex and a ground truth surface by incorporating local surface curvature [S1]. It applies a quadratic approximation of the point-to-point distance to penalize deviations along the principal curvature directions and surface normal [S1]. It serves as a supervised baseline for meshing implicit functions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A predicted mesh vertex $\mathbf{x}$ (output of network $h_\phi$) and a ground truth mesh [S1].
*   **Output:** A scalar loss value $\mathcal{L}$ representing the second-order distance error [S1].

**Data Structures:**
*   $\mathbf{x} \in \mathbb{R}^3$: The 3D coordinates of the query point (predicted vertex) [S1].
*   $\mathbf{y} \in \mathbb{R}^3$: The closest point on the reference (ground truth) surface to $\mathbf{x}$ [S1].
*   $\vec{n} \in \mathbb{R}^3$: The unit surface normal at $\mathbf{y}$ [S1].
*   $\vec{e}_1, \vec{e}_2 \in \mathbb{R}^3$: The orthonormal principal curvature directions at $\mathbf{y}$ [S1].
*   $\hat{\delta}_1, \hat{\delta}_2 \in \mathbb{R}$: The magnitudes of the principal curvatures at $\mathbf{y}$ [S1].

**Steps:**
1.  **Find Closest Point:** For each predicted vertex $\mathbf{x}$, find the closest point $\mathbf{y}$ on the ground truth mesh [S1].
2.  **Compute Geometric Properties:** At the reference point $\mathbf{y}$, compute the surface normal $\vec{n}$, the principal curvature directions $\vec{e}_1, \vec{e}_2$, and their magnitudes $\hat{\delta}_1, \hat{\delta}_2$ [S1].
3.  **Compute Displacement Vector:** Calculate the vector $\mathbf{d} = \mathbf{x} - \mathbf{y}$ [S1].
4.  **Project Displacement:** Project $\mathbf{d}$ onto the normal and the two principal curvature directions: $d_n = \vec{n} \cdot \mathbf{d}$, $d_{e1} = \vec{e}_1 \cdot \mathbf{d}$, $d_{e2} = \vec{e}_2 \cdot \mathbf{d}$ [S1].
5.  **Calculate Loss:** Compute the loss using the quadratic approximation formula [S1]:
    $$
    \mathcal{L} = \hat{\delta}_1 (d_{e1})^2 + \hat{\delta}_2 (d_{e2})^2 + (d_n)^2
    $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def second_order_chamfer_distance(predicted_vertices, ground_truth_mesh):
    total_loss = 0.0
    n_points = len(predicted_vertices)
    for i in range(n_points):
        x = predicted_vertices[i]
        y = ground_truth_mesh.find_closest_point(x)
        n = ground_truth_mesh.get_normal(y)
        e1, e2 = ground_truth_mesh.get_principal_directions(y)
        delta1, delta2 = ground_truth_mesh.get_principal_curvatures(y)
        d = [x[j] - y[j] for j in range(3)]
        dot_n = n[0]*d[0] + n[1]*d[1] + n[2]*d[2]
        dot_e1 = e1[0]*d[0] + e1[1]*d[1] + e1[2]*d[2]
        dot_e2 = e2[0]*d[0] + e2[1]*d[1] + e2[2]*d[2]
        loss_i = (delta1 * dot_e1**2) + (delta2 * dot_e2**2) + (dot_n**2)
        total_loss += loss_i
    return total_loss / n_points
```

**Parameters:**
*   $\mathbf{x}$: The mesh vertex predicted by the network $h_\phi$ [S1].
*   $\mathbf{y}$: The closest point on the ground truth mesh from the training dataset (ABC dataset) [S1].
*   $\hat{\delta}_1, \hat{\delta}_2$: Magnitude of principal curvatures at $\mathbf{y}$ [S1].

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

### [Approximate Level Set Discrepancy](2496-approximate-level-set-discrepancy.md)
Approximate Level Set Discrepancy (LSD-A) is a metric used to quantify the geometric discrepancy between a generated mesh and a ground truth mesh, specifically designed for scenarios where the ground truth is non-watertight or an analytic Signed Distance Field (SDF) is ill-defined. Unlike the Precise Level Set Discrepancy (LSD-P), which evaluates distance errors on a fixed grid near the surface, LSD-A measures the point-to-face distance from points sampled directly on the ground truth mesh to the generated mesh. It serves as a robust alternative to standard Chamfer Distance when strict watertightness cannot be guaranteed.

## Operating characteristics
Not given in the sources.

## Background
The method is a supervised baseline for meshing implicit functions that uses a quadratic approximation of point-point distance proposed in [51] (cited in [S1]) [S1]. It was applied in "Self-Supervised Dual Contouring" to measure distance between a query point and a reference surface using local surface curvature [S1]. The ground truth meshes used in the cited work were from the ABC dataset [S1].

## Sources
[S1] Self-Supervised Dual Contouring, https://inria.hal.science/hal-04590906v1, fetched 2026-10-26
