# Cone Beam Forward Projection with Trilinear Interpolation

Also called: Cone Beam Forward Projection, ConeBeam Projector (General 3D)

## Overview
Computes the projection of a 3D volume onto a 2D detector array for cone beam geometry by integrating voxel values along explicit source-to-detector segments [S1, unofficial]. It differs from its parent [Backward Projection](1491-backward-projection.md) by performing the forward ray-cast operation, and from siblings like [GPU Backprojection](5843-gpu-backprojection.md) by accumulating values into the detector space rather than distributing them into the volume. It supports circular and helical trajectories, flat and curved detectors, and per-view source offsets [S1, unofficial].

## How it works
**Inputs:**
- `volume`: 3D tensor of shape `[B, 1, Z, Y, X]` (batch, channel, depth, height, width).
- `source_positions`: Tensor of shape `[views, 3]`.
- `detector_positions`: Tensor of shape `[views, rows, columns, 3]`.
- `n_samples`: Integer, number of samples per ray (e.g., 192) [S1, unofficial].

**Outputs:**
- `projections`: Tensor of shape `[B, 1, views, rows, columns]` [S1, unofficial].

**Data Structures:**
The volume is stored in a specific bricked layout with a 1-voxel zero border on all sides [S1, unofficial].
- **Bricks**: Volumes are stored in bricks of $4 \times 4 \times 4$ voxels.
- **Layout**: The Python wrapper `_brick` converts standard tensors to this layout. The layout struct contains fields `X`, `Y`, `Z` (volume dimensions), `plane` (stride for detector plane), and `bricks_x` (number of bricks in X direction) [S1, unofficial].
- **Storage**: The internal storage layout is `[bricks * 64, group]` where group corresponds to the number of channels (1, 2, or 4) [S1, unofficial].

**Algorithm:**
The method is implemented as a CUDA kernel launched per view. The kernel iterates over detector pixels and integrates along the ray.

1.  **Thread Indexing**: Each thread handles one detector pixel for one view.
    ```python
    column = blockIdx.x * blockDim.x + threadIdx.x
    row = blockIdx.y * blockDim.y + threadIdx.y
    view = blockIdx.z
    ray_index = ((size_t)view * R + row) * W + column
    ```
    Where $R$ is rows and $W$ is columns [S1, unofficial].

2.  **Initialization**: Initialize accumulator array `acc[C]` to $0.0f$ and `weight` to $0.0f$ [S1, unofficial].

3.  **Ray Construction**: Construct a `ConeRay` from source position, detector element position, and the volume bounding box.
    - Calculate ray direction and length.
    - Clip the ray against the volume box to find entry and exit fractions.
    - If the ray misses the volume or has non-positive span, set `missed = true`.
    - If valid, set `weight = ray.weight` (span / $n$) [S1, unofficial].

4.  **Sampling Loop**: Loop $k$ from $0$ to $n-1$:
    a.  **Sample Position**: Calculate the 3D position $P_k$ along the ray.
        ```python
        # kf is (k + 0.5f) / n
        px = __fmaf_rn(kf, ray.sx, ray.px)
        py = __fmaf_rn(kf, ray.sy, ray.py)
        pz = __fmaf_rn(kf, ray.sz, ray.pz)
        ```
        Where `ray.px, py, pz` are the starting positions and `sx, sy, sz` are the step increments [S1, unofficial].

    b.  **Voxel Coordinates**: Compute integer voxel coordinates.
        ```python
        ix = __float2int_rd(px)
        iy = __float2int_rd(py)
        iz = __float2int_rd(pz)
        ```
        Using round-down mode to ensure consistent cell indexing [S1, unofficial].

    c.  **Fetch Voxel Values**: If voxel coordinates changed ($ix \neq x_0 \mid\mid iy \neq y_0 \mid\mid iz \neq z_0$):
        - Compute cell index and fractional offsets $(ox, oy, oz)$ using a `cell` function.
        - Clamp coordinates to prevent out-of-bounds access: $x = \min(\max(x_0, 0), l.X)$ [S1, unofficial].
        - Fetch 8 voxel values $v[8][C]$ from the bricked volume layout. The indices $q=0..7$ correspond to the corners of the voxel cell containing $P_k$ [S1, unofficial].

    d.  **Trilinear Interpolation**:
        - Compute fractional offsets:
          ```python
          dx = px - (float)ix
          dy = py - (float)iy
          dz = pz - (float)iz
          ```
        - For $C=1$:
          ```python
          acc[0] += trilinear_interpolate(v, dx, dy, dz)
          ```
        - For $C>1$:
          ```python
          wx = [1-dx, dx]
          wy = [1-dy, dy]
          wz = [1-dz, dz]
          for q in 0..7:
              w = (wz[q>>2] * wy[(q>>1)&1]) * wx[q&1]
              for c in 0..C-1:
                  acc[c] += w * v[q][c]
          ```
          Note: The source code uses `__fmaf_rn` for fused multiply-add with round-nearest [S1, unofficial].

5.  **Accumulation**: After the loop, write results to the output buffer.
    ```python
    out[(size_t)c * plane + r] = acc[c] * weight
    ```
    Where `plane` is the stride for the detector plane and `r` is the linear index of the pixel [S1, unofficial].

**Parameters:**
- `C`: Template parameter for number of channels (1, 2, or 4) [S1, unofficial].
- `n_samples`: Integer, number of samples per ray. Typical value: 192 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Out-of-bounds**: Memory access is guarded by clamping in the `cell` function to prevent out-of-bounds access when the ray is near the volume boundary [S1, unofficial].
- **Zero Border**: The volume must be padded with a 1-voxel zero border on all sides to handle boundary conditions correctly during interpolation [S1, unofficial].
- **Precision**: Uses `__fmaf_rn` (fused multiply-add, round-nearest) for numerical stability [S1, unofficial].

## Subtopics

### [Differentiable Forward Projection (Mesh)](13343-differentiable-forward-projection-mesh.md)
Computes the differentiable forward projection of 3D triangle meshes onto 2D detector arrays for both parallel and cone-beam geometries. It differs from volume-based approaches (like its parent [Cone Beam Forward Projection with Trilinear Interpolation](8247-cone-beam-forward-projection-with-trilinear-interpolation.md)) by operating directly on mesh topology (vertices and faces) and utilizing a custom CUDA rasterizer to integrate attenuation coefficients along rays, enabling gradient backpropagation through the projection process.

### [3D Siddon's Method for Intercept Matrix Calculation](14467-3d-siddon-s-method-for-intercept-matrix-calculation.md)
Calculates the exact length of intersection between X-ray rays and voxels in a 3D grid to construct the intercept matrix $A$ for Cone Beam Computed Tomography (CBCT) reconstruction. Unlike direct reconstruction algorithms like FDK, which bypass the explicit calculation of $A$, this approach computes $A$ to enable studies of its properties and the approximation of its inverse for volume reconstruction. It is specifically designed for flat-panel detector cone beam geometry and relies on a parametric ray formulation to determine voxel boundaries.

### Image Scanning for Tomography
A method to generate tomographic projections (scans) from a source image by integrating pixel values along lines at various angles and offsets [S2, unofficial].
- **Steps**:
  1. Calculate sampling steps: $y\_step = 2.0 / base\_width$; $x\_step = 2.0 / base\_height$.
  2. Calculate pixel center offsets: $y\_offset = (image.height - 1.0) / 2.0$; $x\_offset = (image.width - 1.0) / 2.0$.
  3. Iterate over image pixels: for $y\_idx$ in $0..height$, for $x\_idx$ in $0..width$:
     $y = (y\_idx - y\_offset) * y\_step$;
     $x = (x\_idx - x\_offset) * x\_step$;
     call $value\_fn(x, y, pixel\_value)$.
- **Parameters**:
  - `rays`: Number of projection rays (samples per angle), typically set to 35 in tests [S2, unofficial].
  - `angles`: Number of projection angles, typically set to 40 in tests [S2, unofficial].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of detector pixels, views, and samples per ray ($O(V \cdot R \cdot W \cdot n)$).
- **Memory Complexity**: Linear with respect to the volume size and detector size. The bricked layout optimizes memory access patterns for the GPU [S1, unofficial].
- **Measured Performance**: On an RTX 2080 Ti with a $256^3$ phantom, 360 views, and $384 \times 384$ detectors:
  - Forward projection: 0.26s
  - Adjoint: 0.33s
  - Forward with geometry gradient: 0.40s
  - PyTorch path (CPU/CUDA fallback): 207s (Forward), 232s (Adjoint), OOM (Geometry Gradient) [S1, unofficial].
- **Strengths**: The CUDA backend uses runtime-compiled kernels for float32, storing nothing per sample, and is significantly faster than the PyTorch path. It supports exact adjoint consistency [S1, unofficial].
- **Limitations**: The PyTorch path (`backend="torch"`) integrates trilinearly in chunks, retains interpolation stencils for autograd, but fits only small scans due to memory constraints. The CUDA backend does not support higher-order geometry gradients out of the box (requires PyTorch path for higher orders) [S1, unofficial].
- **When to use**: Use the CUDA backend for performance-critical applications where first-order geometry gradients are sufficient. Use the PyTorch path for differentiable geometry requiring higher-order gradients or when NVRTC is unavailable [S1, unofficial].

## Background
Developed by the BIAI Lab as part of the TorchTomo library, this method is part of a suite of differentiable tomography operators [S1, unofficial]. It supports various geometries including parallel, fan, cone, and tomosynthesis beams. The implementation allows for differentiable geometry, enabling calibration and optimization of source and detector positions [S1, unofficial].

## Sources
[S1] itu-biai/torchtomo, https://github.com/itu-biai/torchtomo, fetched 2026-10-27 (unofficial)
[S2] simon-frankau/rediscovering-tomography, https://github.com/simon-frankau/rediscovering-tomography, fetched 2026-10-26 (unofficial)
