# Clipping Volumes

Also called: CSG Clipping Volumes, Half-space Clipping Volumes

## Overview
Clipping volumes restrict rendering to a specific region of space defined by a set of planar half-spaces, allowing internal structures to be revealed by discarding geometry outside the defined volume [S1]. Unlike static cropping masks, this method constructs a dynamic, convex polyhedral volume (up to 6 planes) that can be combined via intersection or union logic, functioning as a simple Constructive Solid Geometry (CSG) expression [S1]. It is distinct from simple view-frustum culling as it allows arbitrary plane orientations and offsets independent of the camera, and from surface-level clipping as it operates on the volumetric data itself [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of up to 6 clipping planes, where each plane is defined by a normal vector $\hat{N}$ and an offset $O$.
*   **Input:** A combination mode (Intersection or Union) for multiple planes [S1].
*   **Input:** Rendering state flags for visualization (`RenderStateClipping.draw`) and outline rendering [S1].
*   **Output:** Rendered scene containing only voxels/geometry inside the clipping volume [S1].
*   **Output:** Optional outline lines at the intersection of geometry and clipping planes [S1].

**Data Structures**
*   **Clipping Plane:** Defined by a 4D vector $[n_x, n_y, n_z, o]$ or separate components.
    *   $\hat{N} = [n_x, n_y, n_z]$: Unit normal vector ($|\hat{N}| = 1.0$) [S1].
    *   $O$: Offset, representing the negative distance along the normal vector [S1].
*   **Clipping Volume State:**
    *   `planes`: Array of up to 6 plane definitions [S1].
    *   `mode`: Enum for `INTERSECTION` or `UNION` [S1].
    *   `draw`: Boolean flag to visualize planes [S1].
    *   `color`: 4D RGBA vector for plane visualization; undefined by default [S1].
    *   `outlineEnabled`: Boolean flag for outline rendering [S1].

**Formulas**
The signed distance $F(x,y,z)$ from a 3D point $P=(x,y,z)$ to a plane defined by normal $\hat{N}$ and offset $O$ is:
$$ F(x,y,z) = x\hat{N}_x + y\hat{N}_y + z\hat{N}_z - O $$
This is computationally equivalent to a 4D dot product where the input point has a $w$ component of $-1$:
$$ \text{signed\_distance} = \text{vec4.dot}([x, y, z, -1], [n_x, n_y, n_z, o]) $$
[S1]

**Logic and Steps**
1.  **Define Planes:** For each plane $i$, ensure the normal vector is normalized to length 1.0 [S1].
2.  **Compute Signed Distance:** For each sample point (voxel or vertex) $P$, calculate $d_i = F(P)$ for all active planes [S1].
3.  **Determine Outside Status:** A point is considered "outside" a single half-space if $d_i > 0$ [S1].
4.  **Combine Planes:**
    *   **Intersection:** The point is inside the volume if it is inside *all* half-spaces (i.e., $d_i \le 0$ for all $i$). In boolean logic terms, it is outside if `outside_plane_1 && outside_plane_2 && ...` [S1].
    *   **Union:** The point is inside the volume if it is inside *any* half-space (i.e., $d_i \le 0$ for at least one $i$). In boolean logic terms, it is outside if `outside_plane_1 || outside_plane_2 || ...` [S1].
5.  **Render Decision:** Discard points determined to be outside the combined volume [S1].
6.  **Visualization (Optional):**
    *   If `RenderStateClipping.draw` is true, render the planes.
    *   Set `RenderStateClippingPlane.color` to an RGBA vector to make planes visible [S1].
    *   Planes are only drawn when the camera is on the positive side (outside) of the plane [S1].
7.  **Outline Rendering (Optional):**
    *   If enabled, render lines where geometry intersects the clipping planes [S1].
    *   Check availability via `View.deviceProfile` under `DeviceProfile.features` [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def is_inside_volume(point, planes, mode):
    """
    Determines if a point is inside the clipping volume.
    
    Args:
        point: Tuple (x, y, z)
        planes: List of tuples (nx, ny, nz, o)
        mode: 'INTERSECTION' or 'UNION'
        
    Returns:
        Boolean: True if inside, False if outside (clipped)
    """
    x, y, z = point
    outside_flags = []
    
    for (nx, ny, nz, o) in planes:
        # Compute signed distance
        # F(x,y,z) = x*nx + y*ny + z*nz - o
        signed_distance = x * nx + y * ny + z * nz - o
        
        # Positive side is outside
        is_outside = signed_distance > 0
        outside_flags.append(is_outside)
        
    if mode == 'INTERSECTION':
        # Point is outside if it is outside ANY plane
        return not any(outside_flags)
    elif mode == 'UNION':
        # Point is outside if it is outside ALL planes
        return not all(outside_flags)
    else:
        raise ValueError("Invalid mode")

def flip_plane(plane):
    """
    Flips a plane by negating all elements.
    """
    nx, ny, nz, o = plane
    return (-nx, -ny, -nz, -o)
```

**Parameters and Typical Values**
*   **Max Planes:** 6 [S1].
*   **Normal Length:** Must be 1.0 [S1].
*   **Plane Color:** Default is undefined (invisible); typically set to an RGBA vector with alpha < 1.0 for transparency [S1].
*   **Clipping ID:** Planes are assigned object IDs in `ClippingId` in the order they are listed, used for picking [S1].

**Edge Cases and Pitfalls**
*   **Dynamic Objects:** Clipping volumes do not apply to dynamic objects [S1].
*   **Outline Performance:** Outline rendering has significant memory overhead and performance cost per plane due to WebGL2 API limitations [S1]. It may not be available on all devices; check `DeviceProfile.features` [S1].
*   **Plane Visibility:** Clipping planes are only drawn when the camera is on the positive side (outside) of the plane. If the camera moves inside the volume, the planes disappear [S1]. For a box (6 planes), only 3 are visible at any given camera position [S1].
*   **Picking:** To create a plane from a picked object, use `PickSample.normal` and check `PickSampleExt.sampleType` to select surface, edge, or corner [S1].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of samples/vertices and $P$ is the number of planes (max 6). The operation is a simple dot product and boolean check per sample.
*   **Memory Complexity:** $O(P)$ for plane definitions. Outline rendering adds significant memory overhead per enabled plane [S1].
*   **Performance:** Standard clipping is lightweight. Outline rendering incurs a notable performance hit and should be used with caution on weaker devices [S1].
*   **Strengths:** Precise control over visible regions; supports arbitrary convex polyhedra; interactive visualization and picking; useful for orthographic projections to reveal hidden surfaces [S1].
*   **Limitations:** Does not support dynamic objects; outline rendering is not universally supported and is costly; planes are not visible from the inside [S1].
*   **When to Use:** Revealing internal structures obscured by outer shells; interactive exploration of volumes; orthographic views where perpendicular surfaces are invisible [S1].
*   **When Not to Use:** Scenes with dynamic objects; devices with limited WebGL2 support if outlines are required; when non-convex clipping volumes are needed (limited to 6 planes forming convex shapes) [S1].

## Background
Clipping volumes are a feature within the Novorender engine, documented in the Novorender documentation [S1]. The method leverages standard half-space clipping techniques common in computer graphics, adapted for volumetric data. It integrates with the engine's picking and rendering state systems, allowing planes to be treated as pickable objects with specific IDs [S1]. The approach is analogous to Constructive Solid Geometry (CSG) operations, specifically using half-space intersections and unions [S1].

## Sources
[S1] Clipping Volumes | Novorender, https://docs.novorender.com/docs/guides/clipping_volumes, fetched 2026-10-29
