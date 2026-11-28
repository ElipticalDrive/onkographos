# Occlusion Caching

Also called: Baked Volumetric Shadows, Baking Occlusion

## Overview
Occlusion caching is a performance optimization for static volumetric lighting that pre-calculates shadow and ambient occlusion data into volume textures, avoiding expensive per-frame camera renders [S2, unofficial]. It sets itself apart from real-time ray marching by trading runtime compute for a one-time or event-driven baking cost, leveraging the static nature of most cloud volumes and occluders [S2, unofficial].

## How it works
### Inputs and Outputs
*   **Inputs:** Static light position/orientation, static occluder geometry, camera parameters for capture [S1][S2, unofficial].
*   **Outputs:** Occlusion volumes $V_{occl}$ containing spherical harmonic (SH) coefficients, and runtime interpolation weights [S1].

### Data Structures
*   **Occlusion Volumes ($V_{occl}$):** A grid of volumetric cells [S1]. Each cell $v_i^{occl}$ stores principal components of occlusion as SH coefficients $f_o^i$ [S1].
*   **Depth Cubemaps ($\{\hat{D}_p^i\}_{p=1}^6$):** Six depth maps per volume cell, captured from six non-overlapping 90-degree FoV cameras [S1].
*   **Occlusion Cubemaps ($\{\hat{O}_p^i\}_{p=1}^6$):** Derived from depth cubemaps, representing the occlusion state [S1].

### Steps
1.  **Baking Stage (Capture):** For each volume cell $v_i^{occl}$, set up six cameras with 90-degree Field of View (FoV), arranged non-overlappingly around the cell center [S1]. Perform six render passes to obtain depth cubemaps $\{\hat{D}_p^i\}_{p=1}^6$ [S1]. Convert depth cubemaps into occlusion cubemaps $\{\hat{O}_p^i\}_{p=1}^6$ [S1]. Compute the principal components of the occlusion data and store them as SH coefficients $f_o^i$ for the cell [S1].
2.  **Decomposition Stage (Runtime Interpolation):** For a surface point $x$ with normal $n$, identify the eight nearest occlusion volumes $\{v_k\}_{k=1}^8$ to $x$ [S1]. Let $p_k$ be the position vector of volume $v_k$ and $f_k^o$ be its SH coefficients [S1]. Calculate standard trilinear interpolation weights $\{w_k\}_{k=1}^8$ based on the relative position of $x$ within the grid [S1]. Apply masked trilinear interpolation to compute modified weights $\tilde{w}_k$:
    $$
    \tilde{w}_k = 
    \begin{cases} 
    0 & \text{if } (p_k - x) \cdot n \le 0 \\
    w_k & \text{if } (p_k - x) \cdot n > 0 
    \end{cases}
    $$
    This mask ensures that only occlusion from the upper hemisphere defined by $n$ contributes to the ambient occlusion integral [S1]. Normalize the weights:
    $$
    \hat{w}_k = \frac{\tilde{w}_k}{\sum_{j=1}^8 \tilde{w}_j}
    $$
    Interpolate the SH coefficients to obtain the final occlusion coefficients $f_x^o$:
    $$
    f_x^o = \sum_{k=1}^8 \hat{w}_k f_k^o
    $$
    [S1]

### Parameters
*   **Bake Interval:** Controls caching behavior [S2, unofficial]. On Start captures once at startup and re-captures if the light moves or rotates [S2, unofficial]. Every Frame enables continuous capture for dynamic occluders [S2, unofficial].
*   **Shadow Resolution:** Resolution of the capture cubemaps (e.g., 512x512) [S2, unofficial].
*   **Ignore Rotation:** Boolean flag. If enabled, rotating the light does not trigger re-capture; the map is reprojected [S2, unofficial].
*   **Auto Toggle Distance:** Distance threshold beyond which occlusion is disabled for far lights [S2, unofficial].
*   **Check Time Interval:** Frequency of Auto Toggle distance checks [S2, unofficial]. Default: 1 second [S2, unofficial].

### Edge Cases and Pitfalls
*   **Dynamic Occluders:** If occluders move independently of the light (e.g., a fan), "On Start" baking will produce stale shadows [S2, unofficial]. Use "Every Frame" or event-driven updates [S2, unofficial].
*   **Rotation:** Without "Ignore Rotation," rotating a static light triggers a full re-capture, increasing cost. Enable this flag for lights with attached occluders (e.g., lamp cages) [S2, unofficial].
*   **Visibility:** Capture can be suspended when the light volume is not visible to any camera to save resources [S2, unofficial].
*   **Point Light Orientation:** For "Point Light Half Sphere To Camera" mode, re-capture only occurs if the camera direction changes by more than 45 degrees [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bake_occlusion_cells(volume_cells):
    occlusion_volumes = {}
    for i, cell in enumerate(volume_cells):
        # Six non-overlapping 90-degree FoV cameras around cell center [S1]
        depth_cubemaps = []
        for p in range(6):
            # Capture depth for face p [S1]
            # Sources do not specify the exact camera setup or rendering pipeline [G]
            depth_cubemaps.append(render_depth_pass(cell.center, p, fov=90.0))
        # Convert depth cubemaps to occlusion cubemaps [S1]
        # Sources do not specify the exact conversion formula [G]
        occlusion_cubemaps = convert_depth_to_occlusion(depth_cubemaps)
        # Store principal components of occlusion into SH coefficients [S1]
        # Sources do not specify the exact SH projection method [G]
        sh_coeffs = compute_sh_coefficients(occlusion_cubemaps)
        occlusion_volumes[i] = sh_coeffs
    return occlusion_volumes

def decompose_occlusion(point_x, normal_n, occlusion_volumes, grid):
    # Find eight nearest volumes {v_k} for point x [S1]
    # Standard grid neighbor lookup [G]
    nearest_indices = find_8_nearest(point_x, grid)
    total_weight = 0.0
    interpolated_sh = [0.0] * 15  # Standard SH basis size [G]

    for k in range(8):
        idx = nearest_indices[k]
        p_k = occlusion_volumes[idx].position
        f_k = occlusion_volumes[idx].sh_coefficients

        # Standard trilinear weight w_k [S1]
        # Standard trilinear interpolation calculation [G]
        w_k = compute_trilinear_weight(point_x, p_k, grid)

        # Masked trilinear interpolation [S1]
        if dot(p_k - point_x, normal_n) <= 0:
            w_tilde_k = 0.0
        else:
            w_tilde_k = w_k

        total_weight += w_tilde_k
        for j in range(len(interpolated_sh)):
            interpolated_sh[j] += w_tilde_k * f_k[j]

    # Normalize weights [S1]
    if total_weight > 0:
        for j in range(len(interpolated_sh)):
            interpolated_sh[j] /= total_weight

    return interpolated_sh
```

## Subtopics

### Masked-Trilinear Interpolation for AO
A variant of the interpolation step used in the decomposition phase [S1]. Standard trilinear interpolation weights $\{w_k\}$ are modified to $\{\tilde{w}_k\}$ by zeroing out contributions from volumes where the vector from the sample point to the volume center has a non-positive dot product with the surface normal ($ (p_k - x) \cdot n \le 0 $) [S1]. This ensures that ambient occlusion is only calculated from the upper hemisphere relative to the surface normal, preventing incorrect shadowing from below [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S2, unofficial].
*   **Memory Complexity:** Not given in the sources [S2, unofficial].
*   **Performance:** Eliminates per-frame shadow ray marching for static lights, significantly reducing GPU load [S2, unofficial].
*   **Strengths:** Ideal for static scenes with static occluders (windows, street lamps) [S2, unofficial]. Event-driven updates handle occasional changes (doors opening) efficiently [S2, unofficial].
*   **Limitations:** Not suitable for highly dynamic scenes with moving occluders unless "Every Frame" baking is used, which negates performance gains [S2, unofficial].
*   **When to Use:** Static volumetric lights with static or slowly moving occluders [S2, unofficial].
*   **When Not to Use:** Real-time dynamic occlusion requirements without acceptable latency for re-baking [S2, unofficial].

## Background
Occlusion caching in volumetric rendering was introduced to address the high computational cost of real-time shadow calculation for volumetric lights [S2, unofficial]. The technique leverages the fact that many volumetric lights and their occluders are static in typical scenes [S2, unofficial]. The method of using SH coefficients to store occlusion data and masked trilinear interpolation for decomposition is detailed in the GS-IR paper [S1]. The Unity URP implementation provides practical settings and runtime behaviors for game engines [S2, unofficial].

## Sources
[S1] GS-IR: 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.1109/cvpr52733.2024.02045, fetched 2026-10-26
[S2] Volumetric Lighting in Unity URP: Shadows, Translucency ..., https://kronnect.com/company/blog/volumetric-lighting-unity-urp-guide/, fetched 2026-10-28 (unofficial)
