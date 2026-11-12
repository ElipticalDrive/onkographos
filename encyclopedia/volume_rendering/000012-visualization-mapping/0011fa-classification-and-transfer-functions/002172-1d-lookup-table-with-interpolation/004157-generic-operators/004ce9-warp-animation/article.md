# Warp Animation

Also called: Linear Interpolation between Views, View Configuration Animation

## Overview
Warp Animation is an interactive visualization technique that links two distinct view configurations ($V_1$ and $V_2$) by linearly interpolating the spatial position of each data point, allowing users to visually track patterns and preserve their mental map while switching between representations [S1]. It sets itself apart from static multi-view coordination by generating a dense sequence of intermediate frames where data patterns become more visible than in either endpoint, effectively "tunneling" through the data space [S1]. Unlike standard morphing which may alter appearance, it fixes the shading of points to ensure visual consistency of specific data groups during the transition [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Two view configurations $V_1$ and $V_2$. Each view maps data points to screen or volume coordinates. $V_1$ is the source view, $V_2$ is the target view.
*   **Input:** A time parameter $t \in [0, 1]$ controlling the interpolation progress.
*   **Output:** An interpolated view $V(t)$ where each point $p$ has a new position $p(t)$ and fixed shading $s(p)$.

**Data Structures:**
The method operates on a set of data points. For each point $i$, the following attributes are required:
*   $p_{1,i}$: Position vector in view $V_1$ (e.g., 2D screen coordinates or 3D volume coordinates).
*   $p_{2,i}$: Position vector in view $V_2$.
*   $s_i$: Shading value (scalar or color) associated with the point.

**Numbered Steps:**
1.  **Pairing:** Identify the correspondence between every point $p_{1,i}$ in $V_1$ and its counterpart $p_{2,i}$ in $V_2$. The correspondence is implicit in the data attribute mapping; points representing the same underlying data entity are paired.
2.  **Interpolation:** For each point $i$, compute the interpolated position $p_i(t)$ using linear interpolation:
    $$ p_i(t) = (1 - t)p_{1,i} + t p_{2,i} $$
3.  **Shading Assignment:** Assign the shading $s_i$ to the point at $p_i(t)$. The shading is fixed and does not interpolate; it remains the value from the original data point.
    *   If $V$ is a 2D image, $s_i$ is the pixel color.
    *   If $V$ is a 3D volume, $s_i$ is the voxel color determined by Direct Volume Rendering (DVR) [S1].
4.  **Depth Ordering (2D Target):** If the target view $V_2$ is 2D (e.g., a histogram or scatterplot) and the source $V_1$ is 3D (DVR), use the depth ($z$) values from $V_1$ to sort the pixels in $V_2$. Pixels with lower $z$ values are rendered under pixels with higher $z$ values to resolve occlusion correctly in the interpolated view [S1].
5.  **Rendering:** Render the set of points $\{ (p_i(t), s_i) \}$ for all $i$.

**Parameters:**
*   $t$: The time parameter. Controlled by horizontal mouse drag. Range $[0, 1]$ for forward animation ($V_1 \to V_2$). Values outside this range allow reverse animation or extrapolation [S1].

**Pitfalls:**
*   **Occlusion in 2D Targets:** When animating from a 3D DVR view to a 2D histogram, simple 2D rendering may hide underlying data. The depth values from the 3D source must be preserved and used for painter's algorithm sorting in the 2D target to ensure correct visual layering [S1].
*   **Point Correspondence:** The method assumes a stable one-to-one mapping between points in $V_1$ and $V_2$. If the data selection changes between views (e.g., brushing removes points), the correspondence must be handled carefully, typically by keeping the set of points constant and allowing their positions to move, or by handling missing points as transparent [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def warp_animation(points_v1, points_v2, t):
    """
    Compute the interpolated view V(t) between V1 and V2.
    
    Args:
        points_v1: List of dicts, each with keys 'pos' (vector), 'shade' (value), 'depth' (scalar, optional).
        points_v2: List of dicts, same structure as points_v1.
        t: Float, interpolation factor [0, 1].
        
    Returns:
        List of dicts representing the interpolated view V(t).
    """
    # Ensure points are paired by index. Assumes len(points_v1) == len(points_v2)
    # and that index i corresponds to the same data entity in both views.
    
    interpolated_points = []
    
    for p1, p2 in zip(points_v1, points_v2):
        # Step 2: Linear interpolation of position
        # p(t) = (1 - t)*p1 + t*p2
        pos_t = (1 - t) * p1['pos'] + t * p2['pos']
        
        # Step 3: Fixed shading
        # Use shade from the original point (they are identical for the same entity)
        shade_t = p1['shade']
        
        # Store depth for sorting if needed
        depth_t = p1.get('depth', 0.0)
        
        interpolated_points.append({
            'pos': pos_t,
            'shade': shade_t,
            'depth': depth_t
        })
        
    # Step 4: Depth ordering if target is 2D and depth is relevant
    # This step is typically handled by the renderer, but the data must be prepared.
    # If rendering to a 2D canvas, sort by depth ascending.
    interpolated_points.sort(key=lambda p: p['depth'])
    
    return interpolated_points
```

## Operating characteristics
*   **Time Complexity:** $O(N)$ per frame, where $N$ is the number of data points. The interpolation is a simple vector addition and scalar multiplication per point.
*   **Memory Complexity:** $O(N)$ to store the positions and shading of all points in both views.
*   **Strengths:** Preserves user's mental map by providing continuous visual feedback; reveals patterns hidden in static views by "tunneling" through occlusions; flexible between 2D and 3D views [S1].
*   **Limitations:** Requires a stable correspondence between points in $V_1$ and $V_2$; performance may degrade with very large $N$ if real-time animation is required; does not interpolate the data values themselves, only their spatial representation [S1].
*   **When to use:** When exploring complex datasets where switching between different visual encodings (e.g., 3D volume vs. 2D histogram) causes loss of context; when trying to identify outliers or structures that are only visible in specific projections [S1].
*   **When not to use:** When the data selection changes dynamically between views in a way that breaks point correspondence; when high precision in data value interpolation is required (this method only interpolates position) [S1].

## Background
Warp Animation was introduced in the context of "Color Tunneling" for interactive exploration and selection in volumetric datasets [S1]. It extends a dual-view design originally used for trajectory analysis to the more general case of DVR data exploration [S1]. The technique allows users to animate between any two view configurations, including 2D and 3D views, by linearly warping the positions of data points while keeping their shading fixed [S1]. It has been applied to exploring brain structures, detecting astronomical outliers, and eliminating complex regions in 2D images [S1].

## Sources
[S1] Color Tunneling: Interactive Exploration and Selection in Volumetric Datasets, https://doi.org/10.1109/pacificvis.2014.61, fetched 2026-10-26
