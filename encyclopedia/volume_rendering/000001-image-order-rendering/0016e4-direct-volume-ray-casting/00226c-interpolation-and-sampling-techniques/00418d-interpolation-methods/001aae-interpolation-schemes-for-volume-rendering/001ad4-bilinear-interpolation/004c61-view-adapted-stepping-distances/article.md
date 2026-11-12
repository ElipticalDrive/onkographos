# View Adapted Stepping Distances

Also called: Adaptive Stepping Distances, Adaptive Termination with Bilinear Interpolation, Jump Bilinear, View Adapted Stepping

## Overview
View Adapted Stepping Distances is a volume rendering acceleration technique that aligns sample points with voxel cube faces to enable bilinear interpolation instead of trilinear interpolation, while using adaptive termination to skip empty space [S1]. It differs from its parent [Bilinear Interpolation](858-bilinear-interpolation.md) by dynamically calculating the stepping distance based on the view direction to enforce this alignment, and from [Optimized Trilinear Reconstruction](2437-optimized-trilinear-reconstruction.md) by reducing the number of neighbor lookups from 8 to 4 [S1].

## How it works
**Inputs:** Ray origin $O$, ray direction $D$, volume data set $V$, volume bounds.
**Outputs:** Final pixel color and opacity.

**Data Structures:**
*   **Ray:** Defined by origin $O=(o_x, o_y, o_z)$ and direction $D=(d_x, d_y, d_z)$.
*   **Voxel Data:** Scalar field $f(x,y,z)$.
*   **Fast Voxel Traversal State:** Current voxel coordinates $(x,y,z)$, step sizes $(\Delta x, \Delta y, \Delta z)$, and t-values for next face crossing $(t_x, t_y, t_z)$ [S1].

**Parameters:**
*   **Epsilon ($\epsilon$):** Adaptive termination threshold, typically 0.1 [S1].
*   **Stepping Distance:** Variable, calculated per ray to ensure samples fall on cube faces.

**Steps:**

1.  **Determine Fastest Axis:** Identify the axis along which the ray travels the furthest for a given length. This is the axis with the largest absolute component in the direction vector $D$ [S1]. Let this axis be $k \in \{x, y, z\}$.
2.  **Adjust Ray Start:** Adjust the ray origin to start on the closest cube face perpendicular to the fastest axis $k$. This ensures the first sample lies on a face [S1].
3.  **Scale Stepping Distance:** Scale the stepping distance of the fast voxel traversal algorithm so that the stepping distance along the fastest axis $k$ is exactly 1 unit. This is achieved by dividing the standard step size by $|d_k|$ [S1].
4.  **Align Sample Position:** Alter the fractional part of the sample point position so that it is zero in the fastest axis $k$, while retaining the fractional placement in the other two axes. This ensures every subsequent sample remains on a face perpendicular to $k$ [S1].
5.  **Fast Voxel Traversal:** Perform fast voxel traversal as normal using the adjusted step sizes and t-values. Each sample point is guaranteed to lie on a cube face perpendicular to the fastest axis [S1].
6.  **Adaptive Termination:** For each sample, check if the accumulated opacity exceeds the threshold $\epsilon$. If so, terminate the ray early to skip empty space [S1].
7.  **Bilinear Interpolation:** For each sample, perform bilinear interpolation using the 4 face-neighboring voxels instead of trilinear interpolation using 8 cube vertices [S1]. The formula is:
    $$t_0 = (v_1 - v_0)r_i + v_0$$
    $$t_1 = (v_3 - v_2)r_i + v_2$$
    $$v_{p_j(i)} = (t_1 - t_0)r_j + t_0$$
    where $r_i$ and $r_j$ are the offsets along the two axes on the face perpendicular to the fastest axis [S1].
8.  **Normal Calculation:** Calculate the normal $G = (g_x, g_y, g_z)$ using central differences from the 4 face-neighboring voxels (reducing operations by 50% compared to 8 neighbors) [S1]:
    $$g_x = f(x+1, y, z) - f(x-1, y, z)$$
    $$g_y = f(x, y+1, z) - f(x, y-1, z)$$
    $$g_z = f(x, y, z+1) - f(x, y, z-1)$$
    Normalize the normal:
    $$l = \sqrt{g_x^2 + g_y^2 + g_z^2}$$
    $$g'_x = g_x/l, \quad g'_y = g_y/l, \quad g'_z = g_z/l$$
9.  **Compositing:** Apply the compositing formula:
    $$o_{out} = (1 - o_i)o_{in} + o_i$$
    for $i = 1, \dots, n$, where $n$ is the number of samples, $o_i$ is the opacity at sample $i$, and $o_{in}$ and $o_{out}$ are the opacities coming into and going out of sample $i$. Note that the interval length does not appear in this formula [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def view_adapted_stepping(ray_origin, ray_direction, volume_data, volume_bounds):
    # 1. Determine fastest axis
    abs_d = abs(ray_direction)
    fastest_axis_index = argmax(abs_d)
    fastest_axis = ['x', 'y', 'z'][fastest_axis_index]
    
    # 2 & 3. Adjust ray start and scale stepping distance
    # Calculate interval size based on view angle
    # Interval size varies between 1 (axis-aligned) and sqrt(3) (45 degrees to each axis)
    interval_size = 1.0 / abs_d[fastest_axis_index]
    
    # Adjust origin to start on closest face perpendicular to fastest axis
    # [G] Implementation detail: Use fast voxel traversal entry point calculation
    # adjusted by the interval size scaling.
    current_voxel, t_max, step = fast_voxel_traversal_setup(
        ray_origin, ray_direction, volume_bounds, interval_size
    )
    
    # 4. Align sample position
    # Ensure fractional part of position in fastest axis is 0
    sample_pos = ray_origin + t_max * ray_direction
    sample_pos[fastest_axis_index] = floor(sample_pos[fastest_axis_index])
    
    accumulated_opacity = 0.0
    final_color = (0.0, 0.0, 0.0)
    
    while not ray_terminated(accumulated_opacity, epsilon=0.1):
        # 5. Fast Voxel Traversal continues automatically in loop condition/update
        
        # 6. Adaptive Termination check happens here implicitly in loop condition
        
        # 7. Bilinear Interpolation
        # Identify the 4 face-neighboring voxels based on current voxel and fastest axis
        # [G] Determine which face of the current voxel the sample lies on
        # based on the fractional coordinates in the non-fastest axes.
        v0, v1, v2, v3 = get_face_neighbors(current_voxel, fastest_axis, sample_pos)
        
        # Offsets in the two axes on the face
        r_i = sample_pos[non_fastest_axis_1] - floor(sample_pos[non_fastest_axis_1])
        r_j = sample_pos[non_fastest_axis_2] - floor(sample_pos[non_fastest_axis_2])
        
        # Bilinear interpolation formula
        t0 = (v1 - v0) * r_i + v0
        t1 = (v3 - v2) * r_i + v2
        value = (t1 - t0) * r_j + t0
        
        # 8. Normal Calculation using 4 neighbors
        gx = volume_data[current_voxel.x + 1, current_voxel.y, current_voxel.z] - \
             volume_data[current_voxel.x - 1, current_voxel.y, current_voxel.z]
        gy = volume_data[current_voxel.x, current_voxel.y + 1, current_voxel.z] - \
             volume_data[current_voxel.x, current_voxel.y - 1, current_voxel.z]
        gz = volume_data[current_voxel.x, current_voxel.y, current_voxel.z + 1] - \
             volume_data[current_voxel.x, current_voxel.y, current_voxel.z - 1]
        
        l = sqrt(gx**2 + gy**2 + gz**2)
        if l > 0:
            gx /= l
            gy /= l
            gz /= l
            
        # [G] Compute shading and opacity from value and normal
        opacity = compute_opacity(value, gx, gy, gz)
        
        # 9. Compositing
        final_color = (1.0 - opacity) * final_color + opacity * compute_color(value, gx, gy, gz)
        accumulated_opacity += opacity * (1.0 - accumulated_opacity)
        
        # Advance ray
        current_voxel, t_max, step = fast_voxel_traversal_step(current_voxel, t_max, step)
        sample_pos = ray_origin + t_max * ray_direction
        # Re-align to face
        sample_pos[fastest_axis_index] = floor(sample_pos[fastest_axis_index])
        
    return final_color
```

## Subtopics

### Adaptive Termination
A technique to skip empty space by stopping the ray when accumulated opacity exceeds a threshold. It reduces computation in transparent regions. Choose when the volume has significant empty space. [S1]

### Fast Voxel Traversal
The underlying traversal algorithm (Amanatides and Woo) used to efficiently step through voxels. It is essential for this method to maintain the face-alignment property. [S1]

## Operating characteristics
*   **Time Complexity:** Reduces computational cost by approximately 50% per sample due to fewer operations (bilinear vs trilinear) and fewer neighbor lookups (4 vs 8) [S1]. Normal calculation operations are also reduced by 50% [S1].
*   **Memory Complexity:** Avoids the extra storage required for precomputed normals (e.g., 200MBytes for a $256^3$ dataset) by calculating normals on the fly [S1].
*   **Measured Performance:** On a 133MHz Dec Alpha with Hydrogen data set at angle 45,45,45, Standard Ray Termination time is 113.16, Jump is 69.18, Bilinear is 39.10, Gain is 26.6% [S1]. On the CThead data set at angle 45,45,45, Standard Ray Termination time is 365.85, Jump is 276.57, Bilinear is 141.71, Gain is 36.9% [S1]. Overall computation time reduction is about 30%-40% [S1]. On the CThead dataset, the method achieved a 30%-40% reduction in computation time compared to standard methods, and was over 7 times faster than the method without optimizations [S1].
*   **Scaling:** The method is scalable, with time increasing with image size, unlike the Template method which is constant time but produces coarse images [S1].
*   **Strengths:** Excellent balance between image quality and computation time; very little extra computation to initiate; works for arbitrary viewing directions [S1].
*   **Limitations:** The interval size varies with viewing angle, being 1 for axis-aligned and $\sqrt{3}$ for 45-degree angles, which affects the number of samples taken [S1]. Using larger interval sizes ($m > 1$) makes the volume appear more transparent than with interval size 1, because the compositing formula depends on sample count $n$, not interval length [S1].
*   **When to use:** When computational speed is critical and slight transparency changes due to interval size variation are acceptable. [S1]
*   **When not to use:** When strict opacity preservation across all viewing angles is required, or when precomputed normals are already available and memory is not a constraint. [S1]

## Background
The method was presented by Mark W. Jones from the Department of Computer Science, University of Wales Swansea [S1]. It addresses the high computational cost of volume rendering, particularly the dominance of normal calculations and trilinear interpolation [S1]. The method builds on the fast voxel traversal algorithm of Amanatides and Woo [S1].

## Sources
[S1] Volume Rendering Using View Adapted Stepping Distances, https://zenodo.org/records/14186166, fetched 2026-10-26
