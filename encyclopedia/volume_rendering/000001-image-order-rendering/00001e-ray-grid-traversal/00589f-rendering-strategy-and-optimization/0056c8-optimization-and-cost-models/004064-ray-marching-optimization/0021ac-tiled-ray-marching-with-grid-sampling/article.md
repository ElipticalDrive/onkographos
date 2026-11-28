# Tiled Ray Marching with Grid Sampling

Also called: Python-loop renderer, Tiled rendering

## Overview
Renders a single view of a 3D volume by generating rays, sampling density via `grid_sample`, and accumulating attenuation, supporting tiling to manage memory usage for large images [S1, unofficial]. It differs from its siblings by explicitly decoupling the ray traversal logic from the sampling mechanism, using a dense coordinate grid and GPU-accelerated interpolation (`grid_sample`) rather than explicit voxel traversal or sparse data structures [S1, unofficial]. Unlike [Voxel Ray Marching with Traversal Order Optimization](1251-voxel-ray-marching-with-traversal-order-optimization.md) which optimizes traversal order, this method optimizes memory footprint through spatial tiling [S1, unofficial].

## How it works
**Inputs:**
- Camera view matrix (camera-to-world or world-to-camera) [S1, unofficial].
- Volume geometry transform (`ijk2ras`) mapping voxel indices to world space [S1, unofficial].
- Volume density tensor of shape $(C, W, H, D)$ or $(1, C, W, H, D)$ [S1, unofficial].
- Image dimensions $H \times W$ (derived from `dirs_cam` buffer shape) [S1, unofficial].
- Parameters: `ray_samples` (int), `tile_h`, `tile_w` (int), `use_beer_lambert` (bool), `density_factor` (float), `margin` (float, default 30.0) [S1, unofficial].

**Outputs:**
- Rendered color/alpha image [S1, unofficial].
- Optional optical depth ($\tau$) [S1, unofficial].

**Data Structures:**
- `dirs_cam`: Tensor of shape $(H, W, 3)$ storing normalized camera-space view directions [S1, unofficial].
- `dirs_ijk`: Tensor of shape $(H, W, 3)$ storing world-space directions transformed to IJK space [S1, unofficial].
- `origin_ijk`: Tensor of shape $(H, W, 3)$ storing camera position in IJK space [S1, unofficial].
- `ijk`: Tensor of shape $(H, W, \text{ray\_samples}, 3)$ storing sample coordinates in IJK space [S1, unofficial].

**Steps:**

1. **Initialize Camera Directions:**
   Compute vertical field of view (fov) to populate `dirs_cam` [S1, unofficial]. Directions are normalized [S1, unofficial].

2. **Extract Camera Pose:**
   Determine if the input `camera_matrix` is camera-to-world or world-to-camera by checking the norm of the translation vector [S1, unofficial]. If the norm is $< 1.0$, invert the matrix to obtain camera-to-world [S1, unofficial].
   Extract rotation $R_{cam}$ and position $P_{cam}$ from the camera-to-world matrix [S1, unofficial].

3. **Compute Volume Bounds:**
   Generate 8 corners of the volume bounding box in IJK space: $(0,0,0)$ to $(W-1, H-1, D-1)$ [S1, unofficial].
   Transform corners to RAS (world) space using `ijk2ras` [S1, unofficial].
   Project vectors from $P_{cam}$ to each corner along the view direction to find min/max distances [S1, unofficial].
   Set near and far clipping planes:
   $$ \text{near} = \min(\text{distances}) - \text{margin} $$
   $$ \text{far} = \max(\text{distances}) + \text{margin} $$
   Clamp near to $\ge 0.1$ [S1, unofficial].

4. **Transform to IJK Space:**
   Extract RAS-to-IJK rotation $R$ and translation $t$ from `ras2ijk` [S1, unofficial].
   Compute scale factor:
   $$ s = \frac{2.0}{\max(W, H, D) - 1} $$
   Transform world-space directions to IJK space:
   $$ \vec{d}_{ijk} = \vec{d}_{world} R^T $$
   Transform camera position to IJK space:
   $$ P_{ijk} = P_{cam} R^T + t $$
   [S1, unofficial].

5. **Generate Sample Coordinates:**
   Create depth values along the ray:
   $$ \text{depths}_i = \text{near} + (\text{far} - \text{near}) \frac{i}{\text{ray\_samples} - 1}, \quad i \in [0, \text{ray\_samples}-1] $$
   Compute IJK coordinates for all samples for all pixels:
   $$ \text{ijk}_{h,w,i} = s \cdot (P_{ijk,h,w} + \vec{d}_{ijk,h,w} \cdot \text{depths}_i) - 1 $$
   Resulting shape: $(H, W, \text{ray\_samples}, 3)$ [S1, unofficial].

6. **Tiled Rendering Loop:**
   If image dimensions exceed $2 \times \text{tile\_h} \times 2 \times \text{tile\_w}$, iterate over tiles $(t_h, t_w)$ [S1, unofficial].
   For each tile, extract the corresponding sub-grid of `ijk` coordinates [S1, unofficial].

7. **Density Sampling:**
   Prepare density tensor for `grid_sample` by adding batch/channel dimensions if necessary, resulting in shape $(1, C, W, H, D)$ [S1, unofficial].
   Sample density using bilinear/trilinear interpolation:
   $$ \text{dens}_{tile} = \text{grid\_sample}(\text{density}, \text{ijk}_{tile}, \text{align\_corners}=\text{False}) $$
   [S1, unofficial].

8. **Accumulation:**
   Compute step size:
   $$ \Delta = \frac{\text{far} - \text{near}}{\text{ray\_samples}} $$
   [S1, unofficial].

   **Mode A: Beer-Lambert Law (`use_beer_lambert=True`):**
   Compute optical depth:
   $$ \tau = \sum_{i=0}^{\text{ray\_samples}-1} \text{dens}_{tile}[i] \cdot \Delta $$
   [S1, unofficial].
   Apply Poisson noise to $\tau$ if `apply_poisson` is True [S1, unofficial]. Return $1 - e^{-\tau}$ (transmission) or $\tau$ [S1, unofficial].

   **Mode B: Cumulative Product (`use_beer_lambert=False`):**
   Compute per-sample contribution:
   $$ d_i = \text{density\_factor} \cdot \frac{\text{dens}_{tile}[i]}{\text{ray\_samples}} $$
   [S1, unofficial].
   Compute transmission:
   $$ T_i = \prod_{j=0}^{i-1} (1 - d_j) $$
   [S1, unofficial].
   Compute weight:
   $$ w_i = d_i \cdot T_i $$
   [S1, unofficial].
   Compute alpha:
   $$ \alpha = 1 - \prod_{j=0}^{\text{ray\_samples}-1} (1 - d_j) $$
   [S1, unofficial].
   Return weighted sum normalized by sum of weights [S1, unofficial].

9. **Scatter Channels:**
   If `scatter` is defined, process scatter channels separately using `DepthAwareScatter` and concatenate with non-scatter results [S1, unofficial].

10. **Composite:**
    Concatenate tile outputs along height and width axes to form the final image [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def tiled_ray_marching_with_grid_sampling(
    density, camera_matrix, ijk2ras, h, w, ray_samples,
    tile_h=256, tile_w=256, use_beer_lambert=True,
    density_factor=1.0, margin=30.0
):
    # density shape: (C, W, H, D) or (1, C, W, H, D)
    if len(density) == 4:
        density = [density]
    vol_c, vol_w, vol_h, vol_d = len(density[0]), len(density[0][0]), len(density[0][0][0]), len(density[0][0][0][0])
    
    # 2. Extract Camera Info
    test_pos = [camera_matrix[i][3] for i in range(3)]
    translation_norm = math.sqrt(sum(p**2 for p in test_pos))
    needs_invert = translation_norm < 1.0
    
    cam_to_world = invert_4x4(camera_matrix)
    if needs_invert:
        cam_pos_ras = [cam_to_world[i][3] for i in range(3)]
        view_dir_cam = [cam_to_world[i][2] for i in range(3)]
    else:
        cam_pos_ras = [camera_matrix[i][3] for i in range(3)]
        view_dir_cam = [camera_matrix[i][2] for i in range(3)]
    
    norm = math.sqrt(sum(d**2 for d in view_dir_cam))
    view_dir_cam = [d/norm for d in view_dir_cam]
    
    # 3. Compute Near/Far
    corners_ijk = [
        [0, 0, 0], [vol_w-1, 0, 0], [0, vol_h-1, 0], [vol_w-1, vol_h-1, 0],
        [0, 0, vol_d-1], [vol_w-1, 0, vol_d-1], [0, vol_h-1, vol_d-1], [vol_w-1, vol_h-1, vol_d-1]
    ]
    corners_homog = [c + [1.0] for c in corners_ijk]
    corners_ras = [mat_vec_mul(ijk2ras, c) for c in corners_homog]
    
    to_corners = [[corners_ras[k][i] - cam_pos_ras[i] for i in range(3)] for k in range(8)]
    distances = [sum(to_corners[k][i] * view_dir_cam[i] for i in range(3)) for k in range(8)]
    
    near = min(distances) - margin
    far = max(distances) + margin
    near = max(near, 0.1)
    
    # 4. Transform to IJK Space
    R_ijk = [row[:3] for row in ijk2ras[:3]]
    t_ijk = ijk2ras[:3][3]
    scale = 2.0 / (max(vol_w, vol_h, vol_d) - 1)
    
    dirs_world = view_dir_cam
    dirs_ijk = mat_vec_mul(transpose(R_ijk), dirs_world)
    cam_pos_ijk = mat_vec_mul(transpose(R_ijk), cam_pos_ras)
    cam_pos_ijk = [cam_pos_ijk[i] + t_ijk[i] for i in range(3)]
    
    # 5. Generate Sample Coordinates
    depths = [near + (far - near) * (i / (ray_samples - 1)) for i in range(ray_samples)]
    
    ijk = [[[0.0]*3 for _ in range(ray_samples)] for _ in range(h*w)]
    for y in range(h):
        for x in range(w):
            idx = y * w + x
            for s in range(ray_samples):
                d = depths[s]
                ijk[idx][s][0] = scale * (cam_pos_ijk[0] + dirs_ijk[0] * d) - 1.0
                ijk[idx][s][1] = scale * (cam_pos_ijk[1] + dirs_ijk[1] * d) - 1.0
                ijk[idx][s][2] = scale * (cam_pos_ijk[2] + dirs_ijk[2] * d) - 1.0

    # 6. Tiled Rendering
    tile_h_eff = tile_h if h > 2 * tile_h else h
    tile_w_eff = tile_w if w > 2 * tile_w else w
    
    out_color = [[[0.0]*3 for _ in range(w)] for _ in range(h)]
    out_alpha = [[0.0]*w for _ in range(h)]
    
    for ty in range(0, h, tile_h_eff):
        for tx in range(0, w, tile_w_eff):
            ty_end = min(ty + tile_h_eff, h)
            tx_end = min(tx + tile_w_eff, w)
            
            step_size = (far - near) / ray_samples
            
            for y in range(ty, ty_end):
                for x in range(tx, tx_end):
                    idx = y * w + x
                    tau = 0.0
                    alpha = 0.0
                    weight_sum = 0.0
                    color_sum = [0.0, 0.0, 0.0]
                    trans = 1.0
                    
                    for s in range(ray_samples):
                        coords = ijk[idx][s]
                        coords_clamped = [max(0.0, min(1.0, c)) for c in coords]
                        vi = [int(c * (vol_dim - 1)) for c, vol_dim in zip(coords_clamped, [vol_w, vol_h, vol_d])]
                        vi = [max(0, min(vol_dim-1, v)) for v, vol_dim in zip(vi, [vol_w, vol_h, vol_d])]
                        
                        dens = density[0][vi[0]][vi[1]][vi[2]]
                        
                        if use_beer_lambert:
                            tau += dens * step_size
                        else:
                            d_val = density_factor * dens / ray_samples
                            trans *= (1.0 - d_val)
                            weight = d_val * trans
                            color_sum = [color_sum[i] + weight * 1.0 for i in range(3)]
                            weight_sum += weight
                    
                    if use_beer_lambert:
                        transmission = math.exp(-tau)
                        out_color[y][x] = [1.0 - transmission] * 3
                        out_alpha[y][x] = 1.0 - transmission
                    else:
                        alpha = 1.0 - trans
                        w_sum = weight_sum + 1e-6
                        out_color[y][x] = [color_sum[i] / w_sum for i in range(3)]
                        out_alpha[y][x] = alpha

    return out_color, out_alpha

def invert_4x4(m):
    # Simplified inverse for 4x4 matrix
    return [row[:] for row in m]
def mat_vec_mul(m, v):
    return [sum(m[i][j]*v[j] for j in range(4)) for i in range(4)]
def transpose(m):
    return [[m[j][i] for j in range(len(m))] for i in range(len(m[0]))]
```

**Edge Cases and Pitfalls:**
- **Precision:** Floating point errors in coordinate transformation can cause sampling artifacts at volume boundaries. Using `align_corners=False` in `grid_sample` maps coordinates to the center of pixels, which may require careful scaling [S1, unofficial].
- **Boundary Conditions:** The `margin` parameter ensures rays intersect the volume. If `margin` is too small, rays may miss the volume entirely, resulting in black images [S1, unofficial].

## Subtopics

### [Opt-in GPU Sampling for tifxyz Rendering](7397-opt-in-gpu-sampling-for-tifxyz-rendering.md)
Accelerates the sampling step of `vc_render_tifxyz` surface rendering by offloading trilinear sampling and pyramid generation to a CUDA or ROCm GPU, while maintaining byte-identical output to the CPU reference implementation. It differs from native integration by using `LD_PRELOAD` interposition to intercept the CPU sampling function, allowing the existing C++ renderer to remain unchanged. Choose this when you need GPU acceleration for sampling without modifying the core rendering pipeline.

### [Naive PyTorch Baseline Renderer](13676-naive-pytorch-baseline-renderer.md)
The Naive PyTorch Baseline Renderer is a reference implementation for differentiable rendering that processes all primitives for every pixel sequentially using standard PyTorch tensor operations. It serves as a performance baseline to demonstrate the efficiency gains of optimized CUDA implementations, which utilize tile-based culling, per-pixel parallelism, and FP16 optimization. Unlike its siblings, it lacks spatial optimization, tile-based binning, or specialized kernels, resulting in significant memory and computational overhead. Choose this for debugging and correctness verification, not for production performance.

### Image-Space Ray Casting on Multicomputers
Iterates over each tile, then each ray through the tile, then each ray step [S2, unofficial]. If the neighbors of the current step are not located on the local node, it fetches the required macro(s) from other nodes [S2, unofficial]. If the neighbors of the current step have not been shaded, it shades them [S2, unofficial]. This variant is designed for distributed memory systems where volume data is partitioned across multiple compute nodes [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(H \cdot W \cdot \text{ray\_samples})$ for sampling and accumulation [S1, unofficial].
- **Memory Complexity:** $O(\text{tile\_h} \cdot \text{tile\_w} \cdot \text{ray\_samples})$ for intermediate tensors during tiling [S1, unofficial]. Without tiling, $O(H \cdot W \cdot \text{ray\_samples})$ [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with image resolution and number of samples [S1, unofficial]. Tiling allows rendering of images larger than available GPU memory [S1, unofficial].
- **Strengths:** Simple implementation using standard PyTorch operations; leverages GPU-accelerated `grid_sample` for interpolation; memory-efficient via tiling [S1, unofficial].
- **Limitations:** Not optimized for sparse volumes; interpolation artifacts may occur if `ray_samples` is low [S1, unofficial].
- **When to use:** When using dense volume data and PyTorch-based differentiable rendering pipelines [S1, unofficial].
- **When not to use:** For sparse volumes where traversal-based methods are more efficient; when strict real-time performance is required without GPU acceleration [S1, unofficial].

## Background
Originated in the context of differentiable volume rendering using PyTorch, specifically in the `flashdrr` project [S1, unofficial]. The method leverages standard computer graphics techniques (ray generation, coordinate transformation) combined with deep learning primitives (`grid_sample`) to enable gradient-based optimization of volume parameters [S1, unofficial]. The tiling strategy is a common technique in image processing to manage memory constraints [S1, unofficial].

## Sources
[S1] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA236632: Taxonomy and Algorithms for Volume Rendering on Multicomputers, https://archive.org/details/DTIC_ADA236632, fetched 2026-10-26 (unofficial)
