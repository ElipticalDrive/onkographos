# Splat Picking

## Overview
Splat Picking is the method of identifying a specific splat primitive located under a user's cursor at sub-frame latency [S1, unofficial]. It solves the problem of interactive selection in splat-based renderers, where the volume is represented as a set of discrete primitives rather than a continuous grid [S1, unofficial]. It is distinguished from its sibling [Splat Point Raycasting with Pixel Distance Threshold](14939-splat-point-raycasting-with-pixel-distance-threshold.md) by its focus on direct cursor association and sub-frame latency targets, accepting partial success rates in exchange for performance [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The current set of splat primitives, camera projection parameters, and the cursor position in screen space or Normalized Device Coordinates (NDC) [G].
*   **Output:** The identifier of the selected splat, or `null` if no unique splat is found within the threshold [S1, unofficial].

**Data Structures:**
*   Not given in the sources.

**Steps:**
1.  **Project to Screen Space:** Transform each splat's 3D center to 2D screen coordinates using the camera projection matrix [G].
2.  **Distance Check:** Compute the Euclidean distance between the cursor position and each projected splat center [S1, unofficial].
3.  **Threshold Comparison:** Identify splats where the distance is less than or equal to a pixel-distance threshold [S1, unofficial].
4.  **Selection:** If exactly one splat satisfies the threshold, return its identifier; otherwise, return `null` [S1, unofficial].

**Formulas:**
Let $\mathbf{u} = (u_x, u_y)$ be the cursor position in screen space, and $\mathbf{p}'_k = (p'_{kx}, p'_{ky})$ be the screen-space projection of splat $k$'s center. The distance is:
$$ d_k = \sqrt{(u_x - p'_{kx})^2 + (u_y - p'_{ky})^2} $$
A splat $k$ is a candidate if $d_k \leq \tau$, where $\tau$ is the pixel-distance threshold [G].

**Parameters:**
*   **Threshold ($\tau$):** Pixel distance for screen-space containment. Not given in the sources.
*   **Latency Target:** < 10 ms p95 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Partial Success:** Not all cursor positions yield a unique splat hit, particularly in dense regions where splats overlap significantly [S1, unofficial].
*   **Precision:** Floating-point rounding during perspective projection can affect hit accuracy near boundaries [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def splat_picking(splats, camera_matrix, cursor_screen):
    """Identify a specific splat under the cursor at sub-frame latency [S1, unofficial]."""
    best_splat = None
    best_dist = float('inf')
    threshold = 1.0  # Pixel distance threshold [G]
    
    for splat in splats:
        # Project 3D center to 2D screen space [G]
        clip = [
            camera_matrix[0][0]*splat.center.x + camera_matrix[0][1]*splat.center.y + camera_matrix[0][2]*splat.center.z + camera_matrix[0][3],
            camera_matrix[1][0]*splat.center.x + camera_matrix[1][1]*splat.center.y + camera_matrix[1][2]*splat.center.z + camera_matrix[1][3],
            camera_matrix[2][0]*splat.center.x + camera_matrix[2][1]*splat.center.y + camera_matrix[2][2]*splat.center.z + camera_matrix[2][3],
            camera_matrix[3][0]*splat.center.x + camera_matrix[3][1]*splat.center.y + camera_matrix[3][2]*splat.center.z + camera_matrix[3][3]
        ]
        if clip[3] <= 0.0:
            continue  # Behind camera
        inv_w = 1.0 / clip[3]
        screen_center = (clip[0] * inv_w, clip[1] * inv_w)
        
        # Euclidean distance check [S1, unofficial]
        dx = cursor_screen.x - screen_center[0]
        dy = cursor_screen.y - screen_center[1]
        dist = (dx*dx + dy*dy)**0.5
        
        if dist <= threshold and dist < best_dist:
            best_dist = dist
            best_splat = splat.id
            
    return best_splat
```

## Subtopics

### [Splat Point Raycasting with Pixel Distance Threshold](14939-splat-point-raycasting-with-pixel-distance-threshold.md)
A method to identify 3D points from a splat mesh that are close to a camera ray by using a pixel-distance threshold in screen space rather than strict geometric intersection, thereby handling the discrete nature of splats. It differs from strict ray-splat intersection by projecting splat centers to screen space and comparing their Euclidean distance to the cursor, allowing for efficient sub-frame latency picking.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** p95 latency of 5.30 ms was recorded on a 177 K splat scene (`butterfly.spz`) [S1, unofficial].
*   **Strengths:** Meets the sub-frame latency target (< 10 ms p95) [S1, unofficial].
*   **Limitations:** Partial success rate; only 49 out of 200 NDC samples produced a unique-splat hit in testing [S1, unofficial].
*   **When to Use:** Interactive selection where sub-frame latency is critical [S1, unofficial].
*   **When Not to Use:** When precise, unambiguous selection is required for all cursor positions [S1, unofficial].

## Background
Splat Picking was investigated as Hypothesis H1 in the `splatcarve` project, aiming to identify a specific splat under the cursor at sub-frame latency [S1, unofficial]. The approach was validated for snap-to-voxel functionality, though functional success was limited by splat overlap in dense regions [S1, unofficial].

## Sources
[S1] stevekwon211/splatcarve, https://github.com/stevekwon211/splatcarve, fetched 2026-10-27 (unofficial)
