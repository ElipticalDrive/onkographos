# Multi-Volume Rendering with Multiple Textures

Also called: Composición de 4 clases simultáneamente, Visualización de múltiples volúmenes utilizando texturas 3D, Viv multi-channel rendering

## Overview
This method renders multiple distinct volumetric classes simultaneously by storing each class in a separate 3D texture and combining them during fragment shader execution [S2, unofficial]. It solves the problem of visualizing heterogeneous data where different channels require independent transfer functions and opacities [S2, unofficial]. It sets itself apart from single-volume approaches by allowing independent control of each channel's appearance, and from out-of-core techniques by relying entirely on GPU memory residency for all active channels [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** $N$ 3D textures, where each texture stores the scalar field (opacity or density) for one specific class [S2, unofficial].
*   **Input:** A set of $N$ transfer function parameters (colors and opacities) for each class [S2, unofficial].
*   **Input:** Ray origin, direction, step size, and volume bounds for sampling [G].
*   **Output:** A single RGBA color value per sample, representing the accumulated color of all classes along the ray [S2, unofficial].

**Data Structures:**
*   **Textures:** $N$ instances of `sampler3D` (or 3D array equivalents). The internal format is constrained by hardware limits [S2, unofficial].
*   **Uniforms:** An array of color vectors, `tissueColor[N]`, where each element contains the RGB color and alpha opacity for the corresponding class [S2, unofficial].

**Steps:**
1.  **Texture Binding:** Assign an independent texture unit for each class in the rendering context using `glActiveTextureARB` followed by binding the specific 3D texture to the active unit [S2, unofficial].
2.  **Primitive Drawing:** Draw the proxy geometry (e.g., view-aligned slices or quads) that covers the volume of interest [S2, unofficial].
3.  **Sampling:** For each fragment or ray step, sample the scalar value from each of the $N$ textures using the fragment's texture coordinates [S2, unofficial].
4.  **Color Computation:** Multiply the sampled scalar value by the pre-defined color and opacity for that class [S2, unofficial].
5.  **Accumulation:** Sum the resulting color vectors from all classes to produce the final output color [S2, unofficial].

**Formulas:**
Let $C_i = (R_i, G_i, B_i, A_i)$ be the color vector (RGB) and opacity scalar for class $i$.
Let $L_i$ be the sampled scalar value (luminance/opacity) from texture $i$ at the current coordinate.
The contribution color $K_i$ for class $i$ is:
$$ K_i = (R_i \cdot L_i, G_i \cdot L_i, B_i \cdot L_i, A_i \cdot L_i) $$
The final output color $K_{final}$ is:
$$ K_{final} = \sum_{i=1}^{N} K_i $$

**Parameters:**
*   **Number of Channels ($N$):** Limited by the number of available texture bind points. WebGL 2.0 guarantees a minimum of eight [S1, unofficial]. Older hardware like GeForce FX supports up to four with ARB multitexture or up to sixteen with NV fragment program extensions [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Limits:** The method does not employ out-of-core techniques. It is strictly limited by the available GPU memory [S1, unofficial]. Large-scale datasets may not fit if the number of channels is high [S1, unofficial].
*   **Hardware Limits:** The maximum number of simultaneous textures is hardware-dependent [S2, unofficial].
*   **Runtime Switching:** Viv switches channels at runtime by changing texture bindings, while Neuroglancer requires users to adapt the fragment shader manually for multiple channels [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trilinear_sample(volume, x, y, z):
    # Clamp coordinates to [0, 1]
    x = max(0.0, min(1.0, x))
    y = max(0.0, min(1.0, y))
    z = max(0.0, min(1.0, z))
    
    # Convert to voxel indices
    dx = len(volume) - 1
    dy = len(volume[0]) - 1
    dz = len(volume[0][0]) - 1
    
    ix = int(x * dx)
    iy = int(y * dy)
    iz = int(z * dz)
    
    # Clamp indices to valid range
    ix = min(max(ix, 0), dx)
    iy = min(max(iy, 0), dy)
    iz = min(max(iz, 0), dz)
    
    # Fractional parts for interpolation
    fx = x * dx - ix
    fy = y * dy - iy
    fz = z * dz - iz
    
    # Fetch 8 neighbors
    v000 = volume[ix][iy][iz]
    v100 = volume[min(ix+1, dx)][iy][iz]
    v010 = volume[ix][min(iy+1, dy)][iz]
    v110 = volume[min(ix+1, dx)][min(iy+1, dy)][iz]
    v001 = volume[ix][iy][min(iz+1, dz)]
    v101 = volume[min(ix+1, dx)][iy][min(iz+1, dz)]
    v011 = volume[ix][min(iy+1, dy)][min(iz+1, dz)]
    v111 = volume[min(ix+1, dx)][min(iy+1, dy)][min(iz+1, dz)]
    
    # Interpolate along x
    v00 = v000 * (1-fx) + v100 * fx
    v01 = v010 * (1-fx) + v110 * fx
    v10 = v001 * (1-fx) + v101 * fx
    v11 = v011 * (1-fx) + v111 * fx
    
    # Interpolate along y
    v0 = v00 * (1-fy) + v01 * fy
    v1 = v10 * (1-fy) + v11 * fy
    
    # Interpolate along z
    return v0 * (1-fz) + v1 * fz

def render_multi_volume(origin, direction, step, bounds_min, bounds_max, textures, tissue_colors):
    # Initialize output color
    final_color = [0.0, 0.0, 0.0, 0.0]
    
    # Handle zero direction components to avoid division by zero
    safe_dir = [max(abs(d), 1e-6) * (1 if d >= 0 else -1) for d in direction]
    
    # Compute ray length inside bounds
    t_min = 0.0
    t_max = float('inf')
    for i in range(3):
        t1 = (bounds_min[i] - origin[i]) / safe_dir[i]
        t2 = (bounds_max[i] - origin[i]) / safe_dir[i]
        t_enter = min(t1, t2)
        t_exit = max(t1, t2)
        t_min = max(t_min, t_enter)
        t_max = min(t_max, t_exit)
        
    if t_min >= t_max:
        return final_color
        
    # Ray march
    t = t_min
    while t < t_max:
        # World position
        pos = [origin[i] + direction[i] * t for i in range(3)]
        
        # Convert to texture coordinates [0, 1]
        tex_coord = [(pos[i] - bounds_min[i]) / (bounds_max[i] - bounds_min[i]) for i in range(3)]
        
        # Accumulate contributions from all channels
        for i in range(len(textures)):
            # Sample scalar value from texture i
            sample_val = trilinear_sample(textures[i], tex_coord[0], tex_coord[1], tex_coord[2])
            
            # Compute color contribution: base_color * sample_val
            contrib = [
                tissue_colors[i][0] * sample_val,
                tissue_colors[i][1] * sample_val,
                tissue_colors[i][2] * sample_val,
                tissue_colors[i][3] * sample_val
            ]
            
            # Accumulate (simple addition as per source)
            final_color = [final_color[j] + contrib[j] for j in range(4)]
        
        t += step
        
    return final_color
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of channels $N$ per fragment, as $N$ texture fetches and additions are performed [S2, unofficial].
*   **Memory Complexity:** $O(N \cdot V)$, where $V$ is the volume of the data. All $N$ textures must reside in GPU memory simultaneously [S1, unofficial].
*   **Strengths:** Simple implementation; allows independent transfer functions per class; supports runtime channel switching [S1, unofficial].
*   **Limitations:** Strictly limited by GPU memory capacity; no out-of-core support; hardware-dependent maximum channel count [S1, S2, unofficial].
*   **When to use:** Small to medium-sized datasets where all channels fit in GPU memory and independent class visualization is required [S1, unofficial].
*   **When not to use:** Large-scale datasets requiring out-of-core streaming or when the number of channels exceeds hardware texture unit limits [S1, unofficial].

## Background
The technique originates from early GPU volume rendering research, specifically utilizing OpenGL's multitexturing capabilities. It was documented in the context of visualizing biomedical data with multiple tissue classes (gray matter, muscle, skin, fat, bone) using GeForce FX hardware [S2, unofficial]. It is also referenced in modern web-based volume renderers like Viv and Neuroglancer [S1, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S2] Texture Mapping Based Volume Rendering, https://www.ciroduran.com/files/Tesis.pdf, fetched 2026-10-29 (unofficial)
