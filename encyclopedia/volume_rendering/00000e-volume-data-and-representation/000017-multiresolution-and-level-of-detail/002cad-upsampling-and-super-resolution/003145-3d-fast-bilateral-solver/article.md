# 3D Fast Bilateral Solver

Also called: 3D FBS, 3D Bilateral Solver

## Overview
The 3D Fast Bilateral Solver (3D FBS) is a post-processing refinement technique that upsamples low-resolution similarity maps to high resolution while adapting them to the edges of an underlying raw intensity volume [S1]. It restores fine details lost due to the lack of resolution in the initial similarity map $S_L$ [S1]. Unlike 2D propagation methods that require complex topology solving and multi-second inference times, the 3D FBS merges features in 3D directly, offering a fast interactive refinement step that aligns coarse segmentations to exact borders in the raw input [S1].

## How it works
The method takes a low-resolution similarity map $S_L$ and the raw intensity volume $V$ as inputs, and produces a high-resolution, edge-aware similarity map $S_H$ [S1]. The core mechanism extends the Fast Bilateral Solver (BLS) to 3D by adding a z-component to each vertex in the bilateral grid, allowing the solver to consider spatial context in all three dimensions [S1].

### Inputs and Outputs
*   **Input:** Low-resolution similarity map $S_L$ and raw intensity volume $V$ [S1].
*   **Output:** High-resolution similarity map $S_H$ [S1].

### Data Structures
The implementation relies on the standard Fast Bilateral Solver data structures extended to 3D:
1.  **Bilateral Grid:** A 3D grid where each cell stores aggregated similarity values and position information. The grid resolution is determined by the parameters $\sigma$ (spatial) and $\sigma_r$ (range/intensity) [G].
2.  **Similarity Map:** A 3D volume of float values representing the confidence or similarity of a voxel belonging to a structure [S1].

### Parameters
*   **Threshold $\tau$:** Used for cropping. Empirically set to $0.25$ [S1].
*   **$\sigma$ (Spatial):** Controls the window size for blurring in the bilateral grid. Influences the occurrence of block artifacts if input detection is insufficient [S1].
*   **Output Resolution $N_H$:** Typically $256^3$ or $512^3$, depending on the class and actual size of the structure [S1].
*   **Reference Volume:** The raw intensity volume $V$ is used as the reference for edge-awareness [S1].

### Algorithm Steps
1.  **Tri-linear Upsampling:** Upsample the low-resolution similarity map $S_L$ tri-linearly to match the resolution of the raw volume $V$ (or the target resolution $N_H$) [S1].
2.  **Cropping:** Crop the regions where the similarity value $S > \tau$ to discard low-similarity areas [S1].
3.  **3D Bilateral Solver Execution:** Solve for the smoothed high-resolution similarity $S_H$ using the cropped region from $V$ as the reference image for edge-awareness [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fast_bilateral_solver_3d(S_L, V, tau=0.25, target_res=256):
    # Step 1: Tri-linear Upsampling
    S_up = tri_linear_upsample(S_L, target_res)

    # Step 2: Cropping
    mask = []
    for z in range(target_res):
        for y in range(target_res):
            for x in range(target_res):
                if S_up[z][y][x] > tau:
                    mask.append((z, y, x))
    if not mask:
        return S_up

    min_c = [min(c) for c in zip(*mask)]
    max_c = [max(c) for c in zip(*mask)]

    S_crop = extract_subvolume(S_up, min_c, max_c)
    V_crop = extract_subvolume(V, min_c, max_c)

    # Step 3: Solve for smoothed S_H
    S_H_crop = bilateral_solver_3d(S_crop, V_crop)

    S_H = create_volume(target_res, 0.0)
    place_subvolume(S_H, S_H_crop, min_c)
    return S_H

def tri_linear_upsample(volume, target_res):
    # [G] Standard tri-linear interpolation scales volume to target_res
    src_size = len(volume)
    scale = target_res / src_size
    out = create_volume(target_res, 0.0)
    for z in range(target_res):
        for y in range(target_res):
            for x in range(target_res):
                src_x = (x + 0.5) / scale - 0.5
                src_y = (y + 0.5) / scale - 0.5
                src_z = (z + 0.5) / scale - 0.5
                out[z][y][x] = interpolate_trilinear(volume, src_x, src_y, src_z)
    return out

def interpolate_trilinear(volume, x, y, z):
    # [G] Standard tri-linear interpolation
    x0, x1 = int(math.floor(x)), int(math.ceil(x))
    y0, y1 = int(math.floor(y)), int(math.ceil(y))
    z0, z1 = int(math.floor(z)), int(math.ceil(z))
    x0, y0, z0 = max(0, x0), max(0, y0), max(0, z0)
    x1, y1, z1 = min(len(volume)-1, x1), min(len(volume)-1, y1), min(len(volume)-1, z1)
    dx, dy, dz = x - x0, y - y0, z - z0
    v000 = volume[z0][y0][x0]
    v100 = volume[z0][y0][x1]
    v010 = volume[z0][y1][x0]
    v110 = volume[z0][y1][x1]
    v001 = volume[z1][y0][x0]
    v101 = volume[z1][y0][x1]
    v011 = volume[z1][y1][x0]
    v111 = volume[z1][y1][x1]
    c00 = v000 * (1-dx) + v100 * dx
    c10 = v010 * (1-dx) + v110 * dx
    c01 = v001 * (1-dx) + v101 * dx
    c11 = v011 * (1-dx) + v111 * dx
    c0 = c00 * (1-dy) + c10 * dy
    c1 = c01 * (1-dy) + c11 * dy
    return c0 * (1-dz) + c1 * dz

def bilateral_solver_3d(S_ref, V_ref):
    # [G] Standard 3D Fast Bilateral Solver
    grid = build_bilateral_grid(V_ref)
    aggregate_into_grid(grid, S_ref, V_ref)
    smoothed_grid = smooth_grid(grid)
    S_out = project_from_grid(smoothed_grid, V_ref)
    return S_out

def build_bilateral_grid(V_ref):
    # [G] Standard grid construction: maps (x,y,z,r) to aggregated values
    # Intensity r is quantized into bins based on sigma_r
    return {}

def aggregate_into_grid(grid, S_ref, V_ref):
    # [G] Standard aggregation: adds S_ref values to grid cells
    # based on spatial position and intensity in V_ref
    pass

def smooth_grid(grid):
    # [G] Standard smoothing: applies 3D Gaussian blur to aggregated grid values
    return grid

def project_from_grid(grid, V_ref):
    # [G] Standard projection: interpolates values from smoothed grid
    # back to output space based on spatial position and intensity in V_ref
    pass

def create_volume(size, val):
    return [[[val for _ in range(size)] for _ in range(size)] for _ in range(size)]

def extract_subvolume(vol, min_c, max_c):
    return [row[min_c[2]:max_c[2]+1] for row in [slice(min_c[1]:max_c[1]+1) for slice in vol]] # Simplified for brevity, assumes 3D list
    # Correct implementation:
    return [[[vol[z][y][x] for x in range(min_c[2], max_c[2]+1)] for y in range(min_c[1], max_c[1]+1)] for z in range(min_c[0], max_c[0]+1)]

def place_subvolume(target, source, min_c):
    for dz in range(len(source)):
        for dy in range(len(source[0])):
            for dx in range(len(source[0][0])):
                target[min_c[0]+dz][min_c[1]+dy][min_c[2]+dx] = source[dz][dy][dx]
```

### Edge Cases and Pitfalls
*   **Insufficient Detection:** If the structure is not sufficiently detected in the low-resolution similarity map $S_L$, the bilateral solver cannot complete structures far beyond what is detected [S1]. The refined similarity falls off smoothly towards the surface, resulting in block artifacts when rendered as an iso-surface [S1].
*   **Block Artifacts:** Caused by the $\sigma$ parameters of the bilateral solver, which control the window used for blurring. These artifacts appear when the input detection is insufficient [S1].
*   **Connected Components Filter:** Using a connected components filter to discard falsely detected disconnected components fails when target structures are too close together [S1].
*   **Resolution Dependency:** Post-processing time varies with resolution, unlike the initial inference time [S1].

## Operating characteristics
*   **Time Complexity:** The post-processing time varies with resolution. On an Intel i7-8700K, it takes approximately 0.4 seconds for a $256^3$ volume and 5.3 seconds for a $512^3$ volume [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Fast interactive refinement (approx. 0.5 seconds); aligns low-resolution feature maps to raw input, especially at borders; detects fine structures (e.g., fish bones) well and avoids noise present in 1D transfer function segmentation; gets within 1 voxel distance for most fine structures [S1].
*   **Limitations:** Requires sufficient detection of the full structure in $S_L$ to produce surfaces without artifacts; cannot complete structures far beyond what is detected in $S_L$; block artifacts may occur if $\sigma$ parameters are not tuned correctly for insufficient input [S1].
*   **When to Use:** When high visual quality is required for rendering similarity maps that were generated at lower resolutions, and the structure is already reasonably well-detected in the low-resolution map [S1].
*   **When Not to Use:** When the initial similarity map $S_L$ lacks sufficient detection of the structure, as the solver cannot complete it [S1].

## Background
The 3D Fast Bilateral Solver is a 3D extension to the Fast Bilateral Solver (BLS) [12], originally proposed for 2D images. It was introduced in the context of leveraging self-supervised Vision Transformers for segmentation-based transfer function design in volume rendering [S1]. The method was developed to address the low resolution of similarity maps generated by neural network features, allowing for interactive annotation-guided transfer function design with instant feedback [S1]. The source code for the approach is publicly available [S1].

## Sources
[S1] Leveraging Self-Supervised Vision Transformers for Segmentation-Based Transfer Function Design., https://doi.org/10.1109/tvcg.2024.3401755, fetched 2026-10-26
