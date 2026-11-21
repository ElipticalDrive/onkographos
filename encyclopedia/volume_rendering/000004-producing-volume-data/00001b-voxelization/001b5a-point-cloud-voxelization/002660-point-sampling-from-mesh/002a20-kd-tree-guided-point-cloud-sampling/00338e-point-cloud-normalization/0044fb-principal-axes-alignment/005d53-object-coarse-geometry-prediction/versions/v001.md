# Object Coarse Geometry Prediction

Also called: Coarse Geometry Prediction, Object pose initialization

## Overview
Object Coarse Geometry Prediction initializes the 6-DoF pose (rotation and translation) and scale of individual objects within a scene by fitting a parametric mesh to an approximate point cloud derived from image segmentation [S1]. It serves as the initialization step for differentiable rendering optimization, ensuring that subsequent gradient-based refinement starts from a geometrically plausible state rather than a random or poorly scaled configuration [S1]. It differs from sibling methods by explicitly decoupling scale initialization (via median-based radius matching) from rotational initialization (via discrete search over vertical-axis rotations), and by incorporating heuristic constraints for floor contact and floating objects to stabilize the optimization landscape [S1].

## How it works
**Inputs:**
*   $P_{obj}$: Approximate point cloud for the object, represented as a set of 3D points $\{p_i\}$.
*   $M_{obj}$: CAD mesh or bounding box geometry for the object, with known center $c_{mesh}$ and radius $r_{mesh}$.
*   $W_{obj}$: Image-space segmentation mask for the object, denoted $W_i \subset M_i$ [S1].
*   $R_{floor}$: Normal vector of the room floor (assumed constant for the scene).
*   $Mask_{floor}$: Image-space segmentation mask for the floor.
*   $P_{other}$: Point clouds or bounding boxes of other objects in the scene (for floating object detection).

**Outputs:**
*   Initial pose parameters $\theta_0 = (R_0, t_0, s_0)$ for the object, where $R_0$ is a rotation matrix, $t_0$ is a translation vector, and $s_0$ is a scale factor.

**Data Structures:**
*   Point Cloud: List of $(x, y, z)$ coordinates.
*   Bounding Box: Defined by center $c$ and extents.
*   Mask: Binary image or set of pixel coordinates.

**Steps:**

1.  **Compute Object Center:**
    Calculate the center of the object point cloud $c_{pc}$ as the median value of the point cloud coordinates over the three axes [S1].
    $$ c_{pc} = \text{median}(P_{obj}) $$
    [G] The median is computed element-wise across the x, y, and z dimensions of the point set.

2.  **Compute Scaling Factor:**
    Determine the scale factor $s_0$ by matching the mesh radius to half the point cloud radius [S1].
    *   Define the point cloud radius $r_{pc}$ as the median distance from each point $p \in P_{obj}$ to the center $c_{pc}$ [S1]:
        $$ r_{pc} = \text{median}_{p \in P_{obj}} (\| p - c_{pc} \|_2) $$
    *   The source notes define radius as "half the point cloud radius" in one context but clarifies "radius is defined as median distance to center" in the detailed description. We follow the detailed description for the calculation of $r_{pc}$ [S1].
    *   Let $r_{mesh}$ be the radius of the mesh bounding box (or a representative radius of the mesh geometry).
    *   Compute scale $s_0$:
        $$ s_0 = \frac{r_{mesh}}{r_{pc}} $$
    *   [G] If $r_{pc} \approx 0$, set $s_0 = 1.0$ to avoid division by zero.

3.  **Initialize Translation:**
    Align the mesh bounding box center $c_{mesh}$ to the point cloud median $c_{pc}$ [S1].
    $$ t_{init} = c_{pc} - s_0 \cdot c_{mesh} $$
    [G] This assumes the mesh is centered at the origin in its local coordinate system. If not, adjust accordingly.

4.  **Align Vertical Axis:**
    Make the object's local vertical axis orthogonal to the room floor normal $R_{floor}$ [S1].
    *   Let $v_{local}$ be the local vertical axis of the object (e.g., $(0,1,0)$).
    *   Compute the rotation $R_{vert}$ that aligns $v_{local}$ with $R_{floor}$ (or the component of $v_{local}$ orthogonal to $R_{floor}$ if strict orthogonality is required, but the text says "orthogonal to the room floor", implying the object's up vector aligns with the floor normal).
    *   [G] Standard rotation alignment: Find rotation $R$ such that $R \cdot v_{local} = R_{floor}$.

5.  **Initialize Rotation (Discrete Search):**
    *   Fix the vertical axis alignment from Step 4.
    *   Define a set of candidate rotations $\mathcal{R}_{cand}$ around the vertical axis (e.g., discrete steps of $15^\circ$ or $30^\circ$).
    *   For each candidate rotation $R_k \in \mathcal{R}_{cand}$:
        *   Render the object mask $W_{render}$ using the current pose $(R_k, t_{init}, s_0)$.
        *   Compute the mean L1 difference between the rendered mask $W_{render}$ and the target segmentation mask $W_{obj}$ [S1]:
            $$ L_{L1}(R_k) = \frac{1}{N} \sum_{pixel} | W_{render}(pixel) - W_{obj}(pixel) | $$
    *   Select the rotation $R_0$ that minimizes this loss:
        $$ R_0 = \arg \min_{R_k \in \mathcal{R}_{cand}} L_{L1}(R_k) $$

6.  **Apply Heuristic Constraints (Optional/Contextual):**
    *   **Floor Contact:** If the distance between the bottom face of the object’s bounding box and the floor is $< 0.1$ AND the distance between the object mask edge and the floor mask edge is $< 20$ pixels, mark the object as "on the floor" [S1].
        *   [G] This flag is used during the subsequent gradient-based optimization phase to apply a floor-distance loss. It does not change the coarse initialization pose but informs the next stage.
    *   **Floating Objects:** If an object is not on the floor, search if its mask edge is within $< 20$ pixels of another object’s edge on the floor [S1].
        *   [G] If so, enforce a constraint to keep the floating object on top of the parent object during optimization. This does not change the coarse initialization pose.

**Pitfalls:**
*   **Scale Initialization:** The method intentionally initializes objects to be smaller than reality (by using median radius which is robust to outliers but smaller than extent-based radius) and allows them to grow during optimization. This makes the optimization better behaved than trying to initialize the scale accurately [S1].
*   **Vertical Axis Assumption:** The heuristic of making the vertical axis orthogonal to the floor is appropriate for most indoor objects but may fail for tilted or irregular objects [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by the discrete rotation search. If $N_{rot}$ is the number of candidate rotations and $T_{render}$ is the time to render a mask, the complexity is $O(N_{rot} \cdot T_{render})$. $T_{render}$ depends on the rasterizer and object complexity.
*   **Memory Complexity:** $O(N_{pc} + N_{mesh})$ to store the point cloud and mesh.
*   **Strengths:** Robust to outliers in point cloud estimation (using median); decouples scale and rotation for easier optimization; heuristic constraints help stabilize differentiable rendering optimization.
*   **Limitations:** Relies on accurate segmentation masks; assumes vertical alignment with floor; discrete search may miss the global optimum if rotation steps are too coarse.
*   **When to use:** When initializing differentiable rendering for indoor scenes with segmented object masks and approximate point clouds.
*   **When not to use:** For objects with significant tilt relative to the floor or when high-precision initial pose is required without subsequent optimization.

## Background
The method was introduced in the context of the PSDR-Room framework, which reconstructs indoor scenes from a single photo using differentiable rendering [S1]. It addresses the challenge of initializing object poses in a scene where only coarse geometric priors (point clouds) and 2D segmentation masks are available. The approach leverages the statistical robustness of median-based statistics for scale and center estimation, combined with a simple image-space metric (L1 mask difference) for rotational initialization.

## Sources
[S1] PSDR-Room: Single Photo to Scene using Differentiable Rendering, https://doi.org/10.1145/3610548.3618165, fetched 2026-11-08
