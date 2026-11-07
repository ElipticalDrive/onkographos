# Forward Projection (aTx)

Also called: Fan Beam Projection, Parallel Beam Projection, X-ray projection, aTx

## Overview
Forward projection (also known as X-ray projection or aTx) is a composition scheme in volume rendering where interpolated sample values along a ray are summed to produce a single scalar intensity [S1]. It solves the problem of generating 2D projection images from 3D volumetric data, mimicking the physics of radiographic imaging [S1]. Unlike Maximum Intensity Projection (MIP) which tracks the maximum value, or alpha blending which composites opacity, forward projection accumulates the total attenuation or density along the ray path [S1]. This method clearly represents high-density structures like bones or vessels but loses depth information [S1].

## How it works
**Inputs and Outputs:**
- Input: A 3D volume dataset $V$, camera parameters (origin $O$, direction $D$), and ray traversal parameters (step size $\Delta t$, volume bounds) [G, parent].
- Output: A 2D image where each pixel $(x, y)$ holds a scalar intensity $I_{xy}$ representing the line integral [G, parent].

**Data Structures:**
- Volume $V$: A 3D array of scalar values. Access is performed via trilinear interpolation in normalized texture coordinates $[0, 1]^3$ [G].
- Accumulator: A scalar initialized to $0.0$ (the identity element for summation) [G, parent].

**Steps:**
1. **Ray Setup**: For each pixel, compute the ray origin $O$ and direction $D$. Determine the entry ($t_{start}$) and exit ($t_{end}$) points of the ray intersecting the volume bounds [G, parent].
2. **Initialization**: Set the accumulator $I \leftarrow 0.0$ [G, parent].
3. **Traversal Loop**: Iterate $t$ from $t_{start}$ to $t_{end}$ with step size $\Delta t$:
   a. Compute the 3D position $P(t) = O + tD$.
   b. Convert $P(t)$ to normalized texture coordinates.
   c. Sample the volume using trilinear interpolation to get scalar value $s_i$ [S1].
   d. Update the accumulator: $I \leftarrow I + s_i \cdot \Delta t$ [S1][S2, unofficial]. Source [S2, unofficial] specifies adding `length * image_value` to the sum, where `image_value` is accessed at index `batch_index * channels * height * width + channel_index * height * width + img_i * width + img_j`.
4. **Finalization**: Store $I$ in the output image for the corresponding pixel [G, parent].

**Formulas:**
The intensity $I$ for a ray is approximated as:
$$ I = \sum_{i} s(t_i) \cdot \Delta t_i $$
where $s(t_i)$ is the interpolated volume value at sample point $t_i$, and $\Delta t_i$ is the step size or voxel traversal length [S1][S2, unofficial].

**Parameters:**
- Step size $\Delta t$: Determines sampling density. Typical values depend on volume resolution and desired accuracy [G].
- Threshold: Not typically used for early termination in standard forward projection as it sums all contributions, unlike MIP [S1].

**Edge Cases and Pitfalls:**
- **Loss of Depth Information**: The method sums all values along the ray, making it impossible to distinguish between a single high-density object and multiple low-density objects [S1].
- **Precision**: Accumulating many small values can lead to floating-point precision issues if the volume is large or values are very small [G].
- **Boundary Conditions**: Rays missing the volume should be handled by skipping the loop or setting intensity to background [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_ray_bounds(origin, direction, bounds_min, bounds_max):
    t_min = float('-inf')
    t_max = float('inf')
    for i in range(3):
        if direction[i] == 0.0:
            if origin[i] < bounds_min[i] or origin[i] > bounds_max[i]:
                return None, None
        else:
            t1 = (bounds_min[i] - origin[i]) / direction[i]
            t2 = (bounds_max[i] - origin[i]) / direction[i]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
    if t_min > t_max:
        return None, None
    return t_min, t_max

def trilinear_interpolate(volume, pos, dims):
    vx = pos[0] * (dims[0] - 1)
    vy = pos[1] * (dims[1] - 1)
    vz = pos[2] * (dims[2] - 1)
    x0, y0, z0 = int(math.floor(vx)), int(math.floor(vy)), int(math.floor(vz))
    x1, y1, z1 = min(x0 + 1, dims[0] - 1), min(y0 + 1, dims[1] - 1), min(z0 + 1, dims[2] - 1)
    x0, y0, z0 = max(0, x0), max(0, y0), max(0, z0)
    wx, wy, wz = vx - x0, vy - y0, vz - z0
    c00 = volume[x0][y0][z0] * (1 - wx) + volume[x1][y0][z0] * wx
    c01 = volume[x0][y0][z1] * (1 - wx) + volume[x1][y0][z1] * wx
    c10 = volume[x0][y1][z0] * (1 - wx) + volume[x1][y1][z0] * wx
    c11 = volume[x0][y1][z1] * (1 - wx) + volume[x1][y1][z1] * wx
    r0 = c00 * (1 - wy) + c10 * wy
    r1 = c01 * (1 - wy) + c11 * wy
    return r0 * (1 - wz) + r1 * wz

def forward_projection(volume, camera, bounds_min, bounds_max, dims, step_size):
    h, w = camera.screen_height, camera.screen_width
    image = [[0.0 for _ in range(w)] for _ in range(h)]
    for y in range(h):
        for x in range(w):
            ray_origin, ray_direction = camera.cast_ray(x, y)
            t_start, t_end = compute_ray_bounds(ray_origin, ray_direction, bounds_min, bounds_max)
            if t_start is None or t_start >= t_end:
                continue
            accumulator = 0.0
            t = t_start
            while t < t_end:
                pos = [ray_origin[i] + t * ray_direction[i] for i in range(3)]
                val = trilinear_interpolate(volume, pos, dims)
                accumulator += val * step_size
                t += step_size
            image[y][x] = accumulator
    return image
```

## Subtopics

### [Analytic 3D Gaussian Ray Tracing for Tomography](3017-analytic-3d-gaussian-ray-tracing-for-tomography.md)
This method replaces local affine approximations with exact ray tracing to compute line integrals through 3D Gaussian primitives analytically. It addresses quantitative accuracy degradation and geometric inflexibility in tomographic reconstruction, such as nonlinear arc-corrections in PET scanners. Unlike splatting-based pipelines that project into 2D screen space, this approach directly computes ray-primitive intersections, avoiding the assumption of non-overlapping Gaussians.

### CT Initialization via Fan Beam Ray Transform and FBP
This variant initializes CT reconstruction by computing a reference image via Fan beam ray transform and filtered back projection (FBP), then creates a sparse sinogram by subsampling and resizing [S3, unofficial]. Steps include: 1. Converting input to CPU numpy array. 2. Initializing sinogram, sparse_sinogram, and ori_img arrays with shape [batch, 720, 720]. 3. Computing sinogram via Fan_ray_trafo. 4. Computing reference image ori_img via Fan_FBP. 5. Storing sinogram max value. 6. Subsampling sinogram along the first axis with stride $r$. 7. Resizing to [720, 720] using cv2.resize [S3, unofficial]. Parameter $r$ is the subsampling stride. Output arrays have shape [batch, 720, 720] and dtype float32 [S3, unofficial].

### Cylindrical Symmetry Projector
A forward/back projector pair designed for objects with cylindrical symmetry [S4, unofficial]. The Abel Transform is a special case of this projector applied to parallel-beam geometries [S4, unofficial]. It supports arbitrary 3D detector shifts and non-equispaced projection angles [S4, unofficial].

## Operating characteristics
- **Performance**: Rapid generation of projection images, accelerated by graphics hardware [S1].
- **Strengths**: Clearly represents high-density structures like bones or vessels; computationally efficient [S1].
- **Limitations**: Loses depth information; only a small fraction of data (the sum) is used, which can be misleading for complex structures [S1]. Static projections lack depth cues [S1].

## Background
Forward projection is a fundamental technique in medical imaging and volume visualization, originating from the physics of X-ray attenuation [S1]. It is widely used in 3D CT and MR angiography for rapid visualization of vessels, tumors, or bones [S1]. The technique is often implemented in differentiable frameworks for tomographic reconstruction, such as DiffTomo and DGR, which utilize CUDA for parallelization over output space samples [S2, S3, S4, unofficial].

## Sources
[S1] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S2] ApaarRaina/DiffTomo, https://github.com/ApaarRaina/DiffTomo, fetched 2026-10-27 (unofficial)
[S3] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
[S4] PDF Differentiable Forward Projector for X-ray Computed Tomography, https://differentiable.xyz/papers-2023/paper_48.pdf, fetched 2026-10-29 (unofficial)
