# Octree Partitioning for Sphere Tracing

Also called: Octree-based sphere tracing, Spatial partitioning with Lipschitz bounds

## Overview
Octree partitioning for sphere tracing accelerates the rendering of implicit surfaces by dividing the domain into an octree where each cell stores a local Lipschitz bound of the field function [S1]. This method allows the ray tracer to cull empty or fully interior nodes without evaluating the implicit surface, solving the problem of slow convergence in regions with high gradient magnitudes [S1]. It differs from [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md) by using static cell bounds rather than dynamic segment-specific bounds, making it susceptible to field function queries at every cell intersection [S1].

## How it works
**Inputs:** A signed distance field (SDF) function $f(\mathbf{x})$, a ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, and an octree structure where each node stores a local Lipschitz constant $L$ and potentially an index to the closest object.
**Outputs:** The intersection distance $t$ to the implicit surface, or a failure state if no intersection is found within the maximum distance.

**Data Structures:**
*   **Octree Node:**
    *   `type`: Enum {Empty, Leaf, Branch}.
    *   `bbox`: Axis-aligned bounding box in world space.
    *   `L`: Local Lipschitz constant (bound on the gradient magnitude $\|\nabla f\|$) for the cell.
    *   `closest_obj_idx`: Optional integer index to the object closest to the cell [S1].

**Steps:**
1.  **Initialization:** Set current traversal distance $t_{curr} = 0$. Start at the root of the octree.
2.  **Ray-Node Intersection:** For the current node, calculate the intersection interval $[t_{enter}, t_{exit}]$ of the ray with the node's bounding box. If the ray misses the node, terminate traversal for this branch.
3.  **Culling Check:**
    *   If the node is `Empty`, skip to the next candidate node.
    *   If the node is `Leaf`, evaluate the SDF at the ray position $\mathbf{x} = \mathbf{o} + t_{curr}\mathbf{d}$.
    *   If the node is `Branch`, determine which child octant the ray enters next. The child coordinates are determined by comparing the intersection point $\mathbf{intersect}$ at the current entry time to the node's midpoint $\mathbf{mid}$:
        $$ \text{coords}[i] = 1 \text{ if } \mathbf{intersect}[i] \ge \text{mid}[i], \text{ else } 0 $$
        Recursively traverse the selected child.
4.  **Sphere Tracing Step with Lipschitz Bound:**
    *   At the current ray position $\mathbf{x}$, compute the SDF value $f(\mathbf{x})$.
    *   Calculate the step size $\Delta t$ using the local Lipschitz constant $L$ of the current node. The step size is bounded by the condition that the function value cannot change faster than $L \cdot \Delta t$. To ensure we do not overshoot the surface, the step is:
        $$ \Delta t = \frac{|f(\mathbf{x})|}{L} $$
        [G] In practice, a small epsilon $\epsilon$ is often added to prevent division by zero or infinite steps if $L=0$ in empty space, though strict adherence to the Lipschitz bound implies $L > 0$ for non-empty regions.
    *   Update $t_{curr} \leftarrow t_{curr} + \Delta t$.
5.  **Convergence Check:**
    *   If $|f(\mathbf{x})| < \epsilon_{surface}$, the surface is hit. Return $t_{curr}$.
    *   If $t_{curr} > t_{max}$, no intersection. Return failure.
    *   If the ray has exited the current node's bounding box, update the current node to the next intersected node in the octree and repeat from Step 2.

**Edge Cases and Pitfalls:**
*   **Closest Object Definition:** When storing an index to the closest object, an object is considered closest to a cell if it is the closest object to every point in the cell. This is verified by the triangle inequality: an object is closest to a cell if the distance from the cell’s centroid to the object, plus the distance from the centroid to the cell corner, is still less than the distance from the centroid to any other object [S1]:
    $$ d(\text{centroid}, \text{object}) + d(\text{centroid}, \text{corner}) < d(\text{centroid}, \text{any\_other\_object}) $$
*   **No Closest Object:** Some cells may not have a closest object under this strict definition, requiring fallback logic or larger bounding volumes [S1].
*   **Precision:** To prevent precision errors when subdivision levels approach floating-point limits, interval arithmetic with correct rounding modes is often employed to guarantee that no octant is missed [S5].

## Subtopics

### [Lipschitz Constant Based Ray Tracing](4065-lipschitz-constant-based-ray-tracing.md)
Accelerates sphere tracing by partitioning the domain into an octree where each cell stores a local Lipschitz bound. It differs from Segment Tracing by using static cell bounds rather than dynamic segment-specific bounds, making it susceptible to field function queries at every cell intersection.

## Operating characteristics
*   **Time Complexity:** Depends on the depth of the octree and the distribution of the Lipschitz constants. In regions with high gradient magnitudes, it significantly reduces the number of SDF evaluations compared to uniform sphere tracing [S1].
*   **Memory Complexity:** $O(N)$ where $N$ is the number of octree nodes. Each node stores a Lipschitz constant and potentially an object index [S1].
*   **Strengths:** Eliminates empty space and imposes local bounds on Lipschitz constants, yielding tighter signed distance bounds and culling of nodes that cannot intersect the implicit surface [S1]. It allows each chunk of the object to be treated individually, penalized only by the largest gradient within its bounds [S1].
*   **Limitations:** Requires pre-computation of Lipschitz constants for each cell. The static nature of the bounds means it may not adapt as dynamically as segment-based methods [S1].

## Background
Octree partitioning was originally used in the polygonization [Bloomenthal, 1989] and ray tracing [Kalra & Barr, 1989] of implicit surfaces. Kalra and Barr used the Lipschitz constant to cull octree nodes guaranteed not to intersect the implicit surface. Sphere tracing reaps the same benefits from spatial partitioning as did the root finding method in [Kalra & Barr, 1989] [S1].

## Sources
[S1] Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces, https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
