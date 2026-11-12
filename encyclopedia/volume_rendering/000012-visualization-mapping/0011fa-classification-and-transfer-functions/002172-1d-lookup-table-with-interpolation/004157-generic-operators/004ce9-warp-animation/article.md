# Warp Animation

Also called: Linear Interpolation between Views, View Configuration Animation

## Overview
Warp Animation is an interactive visualization technique that links two distinct view configurations ($V_1$ and $V_2$) by linearly interpolating the spatial position of data points, allowing users to visually track patterns while switching between representations such as 2D histograms and 3D volume renderings [S1]. It sets itself apart from standard morphing by fixing the shading (color/intensity) of the points during the transition, ensuring that the visual identity of specific data groups remains consistent as they move through intermediate states [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Two view configurations $V_1$ and $V_2$. Each view maps data points to screen or volume coordinates. $V_1$ and $V_2$ may be 2D (e.g., scatterplot, histogram) or 3D (e.g., DVR volume).
*   **Input:** A correspondence mapping between points in $V_1$ and $V_2$.
*   **Input:** A time parameter $t \in [0, 1]$.
*   **Output:** An interpolated view $V(t)$ where each point $p_1 \in V_1$ is mapped to position $p(t)$ with fixed shading $s(p)$.

**Data Structures:**
*   **Point:** A structure containing spatial coordinates $(x, y, z)$ and a shading value $s$ (color or intensity).
*   **View:** A collection of Points. For 3D DVR, points are voxels; for 2D, points are pixels or bins.

**Steps:**
1.  **Initialize Views:** Define the source view $V_1$ and target view $V_2$. Ensure a one-to-one correspondence exists between points $p_1 \in V_1$ and $p_2 \in V_2$ [S1].
2.  **Determine Time Parameter:** The user controls $t$ via horizontal mouse drag while holding the right button. $t=0$ corresponds to $V_1$, $t=1$ to $V_2$ [S1].
3.  **Interpolate Position:** For each point $p_1$ in $V_1$ with corresponding point $p_2$ in $V_2$, compute the new position $p(t)$ using linear interpolation [S1]:
    $$p(t) = (1 - t)p_1 + t p_2$$
    where $p_1$ and $p_2$ are treated as vectors in the display space [S1].
4.  **Fix Shading:** Assign the shading $s(p)$ of the output point to be identical to the shading of the source point $p_1$ (or the original data value associated with $p_1$). For 2D images, $s$ is the pixel color; for 3D volumes, $s$ is the DVR voxel color [S1].
5.  **Handle Depth Ordering (2D Target):** If $V_2$ is a 2D view (e.g., histogram) and $V_1$ is a 3D view (e.g., DVR), use the depth ($z$) values from $V_1$ to determine the rendering order of pixels in $V_2$. Pixels with lower $z$ values in $V_1$ are rendered under pixels with higher $z$ values in $V_2$ [S1].
6.  **Render:** Display the set of points at positions $p(t)$ with shading $s(p)$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def warp_animation(v1_points, v2_points, t):
    """
    Interpolate between two views.
    v1_points: List of Point objects from source view V1.
    v2_points: List of Point objects from target view V2.
    t: Float in [0, 1] representing animation progress.
    Returns: List of Point objects for view V(t).
    """
    v_t_points = []
    
    # Assume v1_points and v2_points are aligned by index
    for p1, p2 in zip(v1_points, v2_points):
        # Step 3: Linear interpolation of position
        # p1 and p2 are vectors (x, y, z)
        pos_t = (1 - t) * p1.position + t * p2.position
        
        # Step 4: Fix shading from source point
        shading = p1.shading
        
        # Create new point
        new_point = Point(position=pos_t, shading=shading)
        v_t_points.append(new_point)
        
    # Step 5: Handle depth ordering if V2 is 2D and V1 is 3D
    # This is a rendering concern, not a data structure change.
    # If rendering to a 2D buffer, sort v_t_points by original z-depth from V1
    # before rasterization.
    
    return v_t_points
```

**Parameters:**
*   $t$: Time parameter controlling animation progress. Range $[0, 1]$ for forward animation ($V_1 \to V_2$). Values outside this range allow reverse animation [S1]. Controlled by horizontal mouse drag [S1].

**Edge Cases and Pitfalls:**
*   **Depth Ordering:** When animating from a 3D view to a 2D view, standard 2D rendering may occlude data incorrectly. The pitfall is ignoring the depth information from the 3D source view. The solution is to use the $z$ values from $V_1$ to sort pixels in $V_2$ during rendering, ensuring proper layering [S1].
*   **Correspondence:** The method assumes a valid correspondence between $V_1$ and $V_2$. If points do not map 1:1, the interpolation may produce meaningless trajectories [S1].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of points, as each point is interpolated independently [S1].
*   **Memory Complexity:** $O(N)$ to store the point sets and intermediate frames [S1].
*   **Strengths:** Preserves mental map by allowing users to track specific data groups across different visual encodings. Reveals patterns in intermediate frames that are not visible in either endpoint view [S1].
*   **Limitations:** Requires a defined correspondence between views. Performance may degrade with very large datasets if real-time interpolation is required [S1].
*   **Usage:** Effective for exploring brain structures (DVR to scatterplot), detecting astronomical outliers (data cube to histogram), and eliminating complex regions in 2D images [S1].

## Background
Warp Animation was introduced in the context of "Color Tunneling" for interactive exploration and selection in volumetric datasets [S1]. It extends a dual-view design originally used for trajectory analysis to the more general case of DVR data exploration [S1]. The technique allows users to animate between any two view configurations, including 2D and 3D views mapping different data attributes [S1].

## Sources
[S1] Color Tunneling: Interactive Exploration and Selection in Volumetric Datasets, https://doi.org/10.1109/pacificvis.2014.61, fetched 2026-10-26
