# Sidedness Test

Also called: Angle bound pruning test

## Overview
The Sidedness Test is a geometric pruning test used to determine if a position bounding box and a normal bounding cone can possibly satisfy the refraction conditions for a light transport path [S1]. It ensures that the view vector $V$ lies on the negative side of the shading normal and within the critical angle cone defined by $\arcsin(1/\eta)$ [S1]. It differs from the smooth normal pruning test only in that positions are bounded by a 3D box rather than a 2D triangle; the test logic is otherwise identical [S1].

## How it works
**Inputs:**
*   Position bounding box $B$ (axis-aligned or oriented, defined by min/max extents).
*   Normal bounding cone $C$ (defined by axis $A$ and semi-angle $\alpha$).
*   Light direction vector $L$ (pointing from surface to light).
*   View direction vector $V$ (pointing from surface to viewer).
*   Refraction index ratio $\eta$ (relative to the medium).

**Outputs:**
*   Boolean: `True` if the box/cone pair *might* contain a valid refraction point; `False` if it can be safely pruned.

**Method:**
The test verifies two conditions for any potential point $P$ in $B$ with a shading normal $N$ in $C$:
1.  $L \cdot N > 0$ (Light is on the positive side).
2.  $V \cdot N < 0$ (Viewer is on the negative side).
3.  The angle between $V$ and $N$ is less than the critical angle $\theta_c = \arcsin(1/\eta)$ [S1].

To perform this efficiently without sampling, the algorithm computes bounds on the angles between the cone axis $A$ and the vectors $L$ and $V$, as seen from any point in the position bounding box.

1.  **Compute angular bounds for $L$:**
    Calculate the minimum and maximum possible angles between the cone axis $A$ and the light direction $L$ over all points $P \in B$. Let this range be $[\theta_{L,min}, \theta_{L,max}]$.
    *   This involves finding the extreme angles subtended by the box $B$ relative to the direction $L$ and the axis $A$.
    *   Specifically, compute the angle $\psi_L$ between $A$ and $L$. Then determine the maximum deviation $\delta_L$ caused by the position variation within $B$. The bounds are $[\psi_L - \delta_L, \psi_L + \delta_L]$ clamped to $[0, \pi]$.

2.  **Compute angular bounds for $V$:**
    Similarly, calculate the minimum and maximum possible angles between the cone axis $A$ and the view direction $V$ over all points $P \in B$. Let this range be $[\theta_{V,min}, \theta_{V,max}]$.

3.  **Offset by normal cone semi-angle:**
    The actual shading normal $N$ can deviate from the cone axis $A$ by at most the semi-angle $\alpha$ of the normal bounding cone $C$.
    *   Adjusted range for $L$: $[\theta_{L,min} - \alpha, \theta_{L,max} + \alpha]$.
    *   Adjusted range for $V$: $[\theta_{V,min} - \alpha, \theta_{V,max} + \alpha]$.
    *   Clamp all angles to $[0, \pi]$.

4.  **Check validity ranges:**
    *   **Light Condition:** The light is on the positive side if the angle between $L$ and $N$ is $< \pi/2$. The test passes if the *minimum* possible angle between $L$ and any valid $N$ is $< \pi/2$.
        *   Check: $\theta_{L,min} - \alpha < \pi/2$.
    *   **Viewer Condition:** The viewer is on the negative side if the angle between $V$ and $N$ is $> \pi/2$. The test passes if the *maximum* possible angle between $V$ and any valid $N$ is $> \pi/2$.
        *   Check: $\theta_{V,max} + \alpha > \pi/2$.
    *   **Critical Angle Condition:** The viewer must be within the critical angle cone. The angle between $V$ and $N$ must be $< \theta_c = \arcsin(1/\eta)$. The test passes if the *minimum* possible angle between $V$ and any valid $N$ is $< \theta_c$.
        *   Check: $\theta_{V,min} - \alpha < \arcsin(1/\eta)$.

5.  **Result:**
    Return `True` if all three checks pass. Otherwise, return `False`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sidedness_test(position_box, normal_cone_axis, normal_cone_semi_angle, L, V, eta):
    """
    position_box: Object with method get_extreme_angles(axis, direction)
                  returning (min_angle, max_angle) for the box relative to axis and direction.
    normal_cone_axis: Vector A
    normal_cone_semi_angle: float alpha
    L: Vector light direction
    V: Vector view direction
    eta: float refraction index ratio
    """
    alpha = normal_cone_semi_angle
    theta_c = math.asin(1.0 / eta)
    
    # Step 1: Compute angular bounds for L relative to cone axis A
    # The box defines the range of positions P. We need the angle between A and L
    # as seen from P. However, the standard formulation [S1] implies computing
    # the angle between the axis A and the vector L, then bounding the deviation
    # due to the position box.
    # Let psi_L be the angle between A and L.
    # Let delta_L be the max angular deviation of the vector (L_from_P) from L
    # due to P moving within the box.
    # Note: The source [S1] says "bounds on the angles between the axis of the shading 
    # normal bounding cone and L and V as seen from any point in the position bounding box".
    # This usually means the angle between the fixed axis A and the vector from P to L.
    
    # Simplified interpretation for implementation:
    # 1. Calculate angle between Cone Axis A and Light L: psi_L
    psi_L = math.acos(max(-1.0, min(1.0, dot(normalize(normal_cone_axis), normalize(L)))))
    
    # 2. Calculate max deviation delta_L caused by position box B
    # This is the maximum angle between the vector (L - P) and L for P in B.
    # Or more commonly, the max angle between the vector from P to L and the vector from P0 to L.
    # For a box, this is computed by finding the point P in B that maximizes the angle.
    # Let's assume a helper function exists for this geometric query.
    delta_L = compute_max_position_angle_deviation(position_box, L)
    
    # 3. Calculate angle between Cone Axis A and View V: psi_V
    psi_V = math.acos(max(-1.0, min(1.0, dot(normalize(normal_cone_axis), normalize(V)))))
    
    # 4. Calculate max deviation delta_V caused by position box B
    delta_V = compute_max_position_angle_deviation(position_box, V)
    
    # Step 2 & 3: Offset by normal cone semi-angle
    # Range for angle(L, N): [psi_L - delta_L - alpha, psi_L + delta_L + alpha]
    # Range for angle(V, N): [psi_V - delta_V - alpha, psi_V + delta_V + alpha]
    
    min_angle_L_N = max(0.0, psi_L - delta_L - alpha)
    max_angle_L_N = min(math.pi, psi_L + delta_L + alpha)
    
    min_angle_V_N = max(0.0, psi_V - delta_V - alpha)
    max_angle_V_N = min(math.pi, psi_V + delta_V + alpha)
    
    # Step 4: Check conditions
    
    # Condition 1: L on positive side (angle < pi/2)
    # Possible if min_angle_L_N < pi/2
    light_positive = min_angle_L_N < (math.pi / 2.0)
    
    # Condition 2: V on negative side (angle > pi/2)
    # Possible if max_angle_V_N > pi/2
    viewer_negative = max_angle_V_N > (math.pi / 2.0)
    
    # Condition 3: V within critical angle (angle < theta_c)
    # Possible if min_angle_V_N < theta_c
    within_critical = min_angle_V_N < theta_c
    
    return light_positive and viewer_negative and within_critical

def compute_max_position_angle_deviation(box, direction):
    """
    Computes the maximum angle between the vector (D - P) and D for P in box.
    This is equivalent to the max angle between the vector from P to the source
    and the fixed direction D.
    [G] Implementation detail: This involves finding the point P in the box
    that is furthest from the line defined by the direction, or using
    bounding sphere approximations. For an AABB, this can be computed by
    checking the corners and edges.
    """
    # Placeholder for the specific geometric calculation of angular deviation
    # from a box. The source [S1] states this is "cheap to compute".
    # A common approach is to bound the distance from the box to the line
    # containing the direction vector.
    return 0.0 # Not given in the sources.
```

**Edge Cases and Pitfalls:**
*   **Precision:** When angles are close to $\pi/2$ or $\theta_c$, floating-point errors may lead to incorrect pruning. Use epsilon comparisons.
*   **Degenerate Box:** If the position box is a point, $\delta_L$ and $\delta_V$ are zero.
*   **Large Normal Variation:** If $\alpha$ is large, the bounds become loose, and the test may fail to prune. However, the test is still cheap [S1].
*   **Total Internal Reflection:** If $\eta < 1$, $\theta_c$ is valid. If $\eta \ge 1$, $\theta_c$ is $\pi/2$ or undefined (no TIR). The formula $\arcsin(1/\eta)$ handles $\eta > 1$. If $\eta \le 1$, the critical angle condition is always true (or irrelevant if no TIR).

## Subtopics

## Operating characteristics
*   **Time Complexity:** $O(1)$ per node in the acceleration structure. The computation involves basic trigonometric functions and comparisons.
*   **Memory Complexity:** $O(1)$ auxiliary space.
*   **Performance:** The test is described as "cheap to compute" [S1]. It allows pruning many subtrees even if they have large normal bounding cones [S1].
*   **Strengths:** It is independent of the specific shading normals, relying only on the bounding cone axis and semi-angle. It handles 3D position boxes effectively.
*   **Limitations:** It is a necessary condition, not sufficient. It may return `True` for boxes that do not contain valid points, leading to further refinement.
*   **When to use:** Use during the traversal of the acceleration structure (e.g., BVH or KD-tree) for refractive media to quickly discard regions that cannot contribute to the light path.

## Background
The Sidedness Test is part of the method for single scattering in refractive media with triangle mesh boundaries, introduced by Walter et al. [S1]. It builds on techniques from [Walter et al. 2005] for pruning sub-triangles in the smooth normal case, extending them to 3D position bounding boxes for refractive boundaries.

## Sources
[S1] Single Scattering in Refractive Media with Triangle Mesh Boundaries, https://doi.org/10.1145/1531326.1531398, fetched 2026-10-26
