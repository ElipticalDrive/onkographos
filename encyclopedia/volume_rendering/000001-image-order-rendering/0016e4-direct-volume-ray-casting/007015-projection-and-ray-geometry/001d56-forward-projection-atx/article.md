# Forward Projection (aTx)

Also called: Fan Beam Projection, Parallel Beam Projection, X-ray projection, aTx

## Overview
X-ray projection (also called Forward Projection or aTx) is a volume rendering composition scheme where interpolated samples along a ray are simply summed to produce an image [S1]. It mimics the physical process of X-ray attenuation, where the total signal is the integral of density along the ray path [G]. Unlike alpha blending, which preserves depth cues through opacity accumulation, this method loses depth information, making it suitable for visualizing high-density structures like bones or vessels where relative depth is less critical than total integrated intensity [S1].

## How it works
**Inputs:**
- A 3D volume $V$ (scalar field, e.g., density or attenuation coefficient) [G].
- Ray parameters defining the projection geometry (origin $\mathbf{o}$, direction $\mathbf{d}$, sampling interval $\Delta s$) [G].

**Outputs:**
- A 2D projection image $P$ where each pixel value $P_{ij}$ is the sum of interpolated volume values along the corresponding ray [G].

**Data Structures:**
- The volume $V$ is stored as a 3D array or grid [G].
- The accumulator for each ray is a scalar initialized to $0$ [G].

**Steps:**
1. For each pixel $(i, j)$ in the output image, define the ray $\mathbf{r}(s) = \mathbf{o}_{ij} + s \cdot \mathbf{d}_{ij}$, where $s$ ranges from the near to the far clipping plane [G].
2. Initialize the accumulator $A = 0$ [G].
3. Iterate through samples $k$ along the ray at positions $s_k$:
   a. Compute the 3D world position $\mathbf{p}_k = \mathbf{r}(s_k)$ [G].
   b. Sample the volume value $v_k = V(\mathbf{p}_k)$ using interpolation (e.g., trilinear) [G].
   c. Update the accumulator: $A \leftarrow A + \Delta s \cdot v_k$ [S2, G, unofficial].
4. Assign the final accumulator value to the pixel: $P_{ij} = A$ [G].

**Formula:**
The discrete approximation of the line integral is:
$$
P_{ij} = \sum_{k=1}^{N} v_k \cdot \Delta s
$$
where $v_k$ is the interpolated volume value at sample $k$, and $\Delta s$ is the step size along the ray [G].

**Parameters:**
- $\Delta s$: Sampling interval. Typical values depend on the volume resolution and desired accuracy; often set to the voxel size or a fraction thereof [G].

**Edge Cases and Pitfalls:**
- **Loss of Depth Information:** Since only the sum is stored, occlusion and depth ordering are not represented [S1].
- **Aliasing:** If $\Delta s$ is too large relative to the frequency content of the volume, artifacts may appear [G].
- **Boundary Conditions:** Rays that do not intersect the volume should return $0$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward_projection(volume, ray_origins, ray_directions, step_size):
    """
    volume: 3D list/tuple or array of shape (D, H, W)
    ray_origins: 2D array of shape (H_out, W_out, 3)
    ray_directions: 2D array of shape (H_out, W_out, 3)
    step_size: float, distance between samples
    """
    H_out, W_out, _ = ray_origins.shape
    projection = [[0.0 for _ in range(W_out)] for _ in range(H_out)]
    D, H, W = volume.shape
    max_dist = 1000.0  # Placeholder for actual bounding box intersection logic

    for y in range(H_out):
        for x in range(W_out):
            origin = ray_origins[y][x]
            direction = ray_directions[y][x]
            # Normalize direction
            norm = (direction[0]**2 + direction[1]**2 + direction[2]**2)**0.5
            if norm > 0:
                direction = [d / norm for d in direction]
            else:
                direction = [0.0, 0.0, 0.0]

            accumulator = 0.0
            # Iterate along the ray
            for s in range(0, int(max_dist / step_size)):
                point_x = origin[0] + s * step_size * direction[0]
                point_y = origin[1] + s * step_size * direction[1]
                point_z = origin[2] + s * step_size * direction[2]

                # Check if point is within volume bounds
                if 0 <= point_x < D and 0 <= point_y < H and 0 <= point_z < W:
                    v = trilinear_sample(volume, point_x, point_y, point_z)
                    accumulator += step_size * v

            projection[y][x] = accumulator

    return projection

def trilinear_sample(volume, x, y, z):
    # Standard trilinear interpolation implementation [G]
    ix, iy, iz = int(x), int(y), int(z)
    fx, fy, fz = x - ix, y - iy, z - iz
    # Clamp indices to volume bounds
    ix = max(0, min(ix, volume.shape[0] - 1))
    iy = max(0, min(iy, volume.shape[1] - 1))
    iz = max(0, min(iz, volume.shape[2] - 1))
    # Nearest-neighbor fallback for brevity; replace with full trilinear blend [G]
    return volume[ix][iy][iz]
```

## Subtopics

### [Cylindrical Symmetry Projector](3005-cylindrical-symmetry-projector.md)
A forward and back projector pair designed for objects possessing cylindrical symmetry, serving as a specialized case of the Forward Projection (aTx) method. It exploits rotational invariance to reduce 3D line integrals to a 1D radial problem, distinguishing it from general 3D projectors that handle arbitrary distributions. Choose it when the target volume exhibits axisymmetry and computational efficiency is prioritized over arbitrary geometry support.

### [Analytic 3D Gaussian Ray Tracing for Tomography](3017-analytic-3d-gaussian-ray-tracing-for-tomography.md)
A method that replaces local affine approximations with exact ray tracing to compute line integrals through 3D Gaussian primitives analytically. It directly computes ray-primitive intersections, avoiding the non-overlapping Gaussian assumption and addressing nonlinear corrections like arc-correction in PET scanners. Choose it when quantitative accuracy and geometric flexibility are required over screen-space splatting pipelines.

### [CT Initialization via Fan Beam Ray Transform and FBP](2706-ct-initialization-via-fan-beam-ray-transform-and-fbp.md)
A variant that initializes CT reconstruction by computing a reference image via Fan beam ray transform and filtered back projection (FBP), then creates a sparse sinogram by subsampling and resizing. It is used when preparing initialization data for tomographic pipelines that require sparse or resized projection inputs.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Simple implementation, fast computation, good for visualizing dense structures.
- **Limitations:** No depth information, no transparency effects, sensitive to sampling density.
- **When to use:** When depth cues are not required, and the goal is to visualize the total integrated density (e.g., bone visualization in CT).
- **When not to use:** When transparency, occlusion, or depth perception is important.

## Background
Not given in the sources.

## Sources
[S1] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S2] ApaarRaina/DiffTomo, https://github.com/ApaarRaina/DiffTomo, fetched 2026-10-27 (unofficial)
