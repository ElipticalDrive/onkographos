# Voxel DDA Face Entry/Exit UV Mapping

Also called: Face Entry/Exit UV Mapping

## Overview
This method computes the normalized entry and exit UV coordinates of a ray within a specific voxel during Digital Differential Analyzer (DDA) traversal. It solves the problem of geometric discontinuity in terrain rendering by providing precise surface parameterization for each voxel face, enabling smooth interpolation across voxel boundaries. It sets itself apart from standard DDA variants, such as [Midpoint UV Calculation](14669-midpoint-uv-calculation.md), by integrating geometric UV extraction directly into the traversal step to support surface smoothing logic, rather than merely tracking voxel indices or using midpoint approximations.

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $(O_x, O_y, O_z)$, ray direction $(D_x, D_y, D_z)$, current voxel bounds $(V_{min}, V_{max})$, and the entry face mask indicating which face the ray enters.
*   **Output:** Normalized UV coordinates $(u, v)$ representing the position on the voxel face where the ray enters or exits.

**Data Structures:**
*   **Entry Mask:** A boolean or integer flag indicating the entry face (X, Y, or Z).
*   **UV Coordinates:** Floating-point values in the range $[0, 1]$.

**Steps:**
1.  **Determine Entry Face:** Check the entry mask to identify which face (X, Y, or Z) the ray intersects first.
2.  **Calculate Local Coordinates:** Compute the intersection point $(P_x, P_y, P_z)$ on the entry face.
3.  **Normalize to UV:** Map the intersection point to the $[0, 1]$ range based on the voxel dimensions.
4.  **Interpolate for Smoothing:** If entering from the "left" or "back" (negative axis direction), set the start UV to the back/left face and interpolate toward the center. If entering from the "right" or "front" (positive axis direction), set the start UV to the front/right face and interpolate toward the center. This ensures continuity across voxel edges.

**Formulas:**
For a ray entering through the X-face (left or right):
$$ u = \frac{P_y - V_{min,y}}{V_{max,y} - V_{min,y}} $$
$$ v = \frac{P_z - V_{min,z}}{V_{max,z} - V_{min,z}} $$
The interpolation direction depends on the sign of $D_x$. If $D_x < 0$ (entering from left), the UVs are oriented from the back/left face toward the center. If $D_x > 0$ (entering from right), they are oriented from the front/right face toward the center. Similar logic applies for Y and Z faces.

**Parameters:**
*   **Precision:** Floating-point types are mandatory.

**Edge Cases and Pitfalls:**
*   **Integer Division:** Using integer division for UV coordinate calculations causes terrain vertices to fail to connect across voxel edges. All coordinate math must use floating-point types [S1, unofficial].
*   **Directionality:** Ensure the interpolation direction matches the ray's entry direction. For example, if entering from the left, the UVs should start at the back face and go to the center, not the front [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_face_uv(ray_origin, ray_direction, voxel_min, voxel_max, entry_face):
    # Calculate intersection point on the entry face
    if entry_face == 'X':
        # Intersect with X face
        t = (voxel_min.x if ray_direction.x < 0 else voxel_max.x - ray_origin.x) / ray_direction.x
        # Note: The source implies a specific logic for 'left' vs 'right' entry
        # If entering from left (ray_direction.x < 0), start at back/left face
        # If entering from right (ray_direction.x > 0), start at front/right face
        # This logic is applied to the UV interpolation direction
        pass
    elif entry_face == 'Y':
        t = (voxel_min.y if ray_direction.y < 0 else voxel_max.y - ray_origin.y) / ray_direction.y
        pass
    elif entry_face == 'Z':
        t = (voxel_min.z if ray_direction.z < 0 else voxel_max.z - ray_origin.z) / ray_direction.z
        pass
    
    # Calculate intersection point
    intersection = ray_origin + ray_direction * t
    
    # Normalize to UV
    # Use floating-point division
    u = (intersection.y - voxel_min.y) / (voxel_max.y - voxel_min.y)
    v = (intersection.z - voxel_min.z) / (voxel_max.z - voxel_min.z)
    
    # Adjust UV orientation based on entry direction for smoothing
    # If entering from negative axis, interpolate from back/left to center
    # If entering from positive axis, interpolate from front/right to center
    # This ensures continuity across voxel boundaries
    
    return u, v
```

## Subtopics

### [Voxel Entry/Exit UV Calculation for DDA Raymarching](22274-voxel-entry-exit-uv-calculation-for-dda-raymarching.md)
This method computes the normalized entry and exit UV coordinates of a ray within a voxel during Digital Differential Analyzer (DDA) traversal, enabling smooth terrain rendering by determining if the ray passes through voxel content or exits. It sets itself apart from standard DDA by integrating geometric UV extraction directly into the traversal step to support surface smoothing logic, rather than merely tracking voxel indices.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel face intersection.
*   **Memory Complexity:** $O(1)$ for storing UV coordinates.
*   **Strengths:** Enables smooth terrain rendering by ensuring geometric continuity across voxel boundaries.
*   **Limitations:** Requires careful handling of floating-point precision to avoid artifacts.
*   **When to Use:** When smooth terrain rendering is required in voxel-based environments.
*   **When Not to Use:** When simple voxel indexing is sufficient and surface smoothing is not needed.

## Background
This method originates from techniques used in voxel-based terrain rendering, specifically in the context of DDA raymarching for smooth surface generation. It addresses the challenge of maintaining geometric continuity across voxel boundaries, which is critical for realistic terrain visualization.

## Sources
[S1] Smooth terrain with voxel DDA raymarching, https://www.youtube.com/watch?v=CxoC0E7inUg, fetched 2026-10-25 (unofficial)
