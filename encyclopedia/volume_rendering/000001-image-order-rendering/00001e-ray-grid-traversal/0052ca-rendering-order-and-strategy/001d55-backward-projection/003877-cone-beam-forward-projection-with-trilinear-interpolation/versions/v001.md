# Cone Beam Forward Projection with Trilinear Interpolation

Also called: Cone Beam Forward Projection

## Overview
Computes the projection of a 3D volume onto a 2D detector array for cone beam geometry by ray-casting through the volume [S1, unofficial]. It uses trilinear interpolation to sample the volume along rays defined by source and detector positions [S1, unofficial]. It simulates cone-beam projections by reading volumes in a specific bricked layout with a 1-voxel zero border [S1, unofficial].

## How it works
**Inputs:**
- Volume tensor in standard layout $[B, 1, Z, Y, X]$ [S1, unofficial].
- Cone beam geometry parameters (source positions, detector positions/orientations, pixel size) [S1, unofficial].
- Number of samples per ray $n$ [S1, unofficial].
- Number of channels $C$ (template parameter, typically 1, 2, or 4) [S1, unofficial].

**Outputs:**
- Projection tensor of shape $[B, C, \text{views}, \text{detector\_height}, \text{detector\_width}]$ [S1, unofficial].

**Data Structures:**
- **Layout:** Defined by volume dimensions $X, Y, Z$, detector plane stride, and bricks\_x [S1, unofficial].
- **Volume Storage:** Padded with a 1-voxel zero border on all sides [S1, unofficial]. Stored in bricks of $4 \times 4 \times 4$ voxels [S1, unofficial]. The Python wrapper `_brick` converts standard tensors to this layout $[\text{bricks} \times 64, \text{group}]$ [S1, unofficial].
- **ConeRay:** Contains entry/exit distances, direction components, initial position, step size, and weight [S1, unofficial].

**Steps:**
1. **Brick Conversion:** Convert the input volume from standard layout $[B, 1, Z, Y, X]$ to the bricked layout $[\text{bricks} \times 64, C]$ using the `_brick` function [S1, unofficial].
2. **Kernel Launch:** Launch a CUDA kernel with grid size derived from the projection shape and block size $(8, 16, 1)$ [S1, unofficial]. The kernel is specialized for $C \in \{1, 2, 4\}$ [S1, unofficial].
3. **Thread Indexing:** Each thread handles one detector pixel for one view [S1, unofficial].
   - `column = blockIdx.x * blockDim.x + threadIdx.x`
   - `row = blockIdx.y * blockDim.y + threadIdx.y`
   - `view = blockIdx.z`
   - `ray_index = ((size_t)view * R + row) * W + column` where $R$ is detector height, $W$ is detector width [S1, unofficial].
4. **Initialization:** Initialize accumulator `acc[C]` to $0.0f$ and `weight` to $0.0f$ [S1, unofficial].
5. **Ray Construction:** Construct `ConeRay` from source, detector, and bounding box [S1, unofficial].
   - Calculate ray direction and length [S1, unofficial].
   - Clip ray against bounding box to find entry and exit distances [S1, unofficial].
   - If the ray misses or has non-positive span, set `missed = true` [S1, unofficial].
   - If valid, set `weight = ray.weight` where `ray.weight = span / n` [S1, unofficial].
   - Calculate initial sample position at half-step: `first = ray.entry + 0.5f * ray.weight` [S1, unofficial].
   - Compute initial position `px, py, pz` and step components `sx, sy, sz` [S1, unofficial].
6. **Sampling Loop:** Loop $k$ from $0$ to $n-1$ [S1, unofficial].
   - Calculate sample position:
     $$px = \text{fma}(k \cdot \text{step}, \text{ray.sx}, \text{ray.px})$$
     $$py = \text{fma}(k \cdot \text{step}, \text{ray.sy}, \text{ray.py})$$
     $$pz = \text{fma}(k \cdot \text{step}, \text{ray.sz}, \text{ray.pz})$$
     (Using `__fmaf_rn` for fused multiply-add with round-nearest) [S1, unofficial].
   - Compute integer voxel coordinates:
     $$ix = \lfloor px \rfloor, \quad iy = \lfloor py \rfloor, \quad iz = \lfloor pz \rfloor$$
     (Using `__float2int_rd`) [S1, unofficial].
   - If voxel coordinates changed from previous step ($ix \neq x_0$ or $iy \neq y_0$ or $iz \neq z_0$):
     - Compute cell index and offsets $(ox, oy, oz)$ using a `cell` function [S1, unofficial].
     - Fetch 8 voxel values $v[8][C]$ from the bricked volume [S1, unofficial].
     - Update $x_0, y_0, z_0$ to current $ix, iy, iz$ [S1, unofficial].
   - Compute fractional offsets:
     $$dx = px - ix, \quad dy = py - iy, \quad dz = pz - iz$$ [S1, unofficial].
   - Perform trilinear interpolation:
     - For $C=1$:
       $$acc[0] += (1-dx)(1-dy)(1-dz)v[0] + dx(1-dy)(1-dz)v[1] + (1-dx)dy(1-dz)v[2] + dx dy (1-dz)v[3] + (1-dx)(1-dy)dz v[4] + dx(1-dy)dz v[5] + (1-dx)dy dz v[6] + dx dy dz v[7]$$
       (Implemented using nested `__fmaf_rn` calls) [S1, unofficial].
     - For $C>1$:
       $$wx[2] = \{1-dx, dx\}, \quad wy[2] = \{1-dy, dy\}, \quad wz[2] = \{1-dz, dz\}$$
       Loop $q=0..7$:
       $$w = (wz[q>>2] \cdot wy[(q>>1)\&1]) \cdot wx[q\&1]$$
       $$acc[c] += w \cdot v[q][c] \quad \forall c \in [0, C)$$ [S1, unofficial].
7. **Write Results:**
   $$out[(size_t)c \cdot \text{plane} + \text{ray\_index}] = acc[c] \cdot \text{weight} \quad \forall c < C$$ [S1, unofficial].

**Pitfalls:**
- Memory access is guarded by clamping in the `cell` function: $x = \min(\max(x_0, 0), L.X)$, etc., to prevent out-of-bounds access [S1, unofficial].
- The volume must be padded with a 1-voxel zero border; failure to do so may cause incorrect sampling at boundaries [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N_{rays} \cdot n)$, where $N_{rays}$ is the number of detector pixels times views, and $n$ is samples per ray [S1, unofficial].
- **Memory Complexity:** $O(V)$ for volume storage, where $V$ is the number of voxels including padding and brick overhead [S1, unofficial].
- **Performance:** Measured on RTX 2080 Ti, 512x512, batch 4, 360 views: CUDA exact forward projection takes ~2.196 ms (Parallel Beam) or ~2.720 ms (Fan Beam) [S1, unofficial].
- **Strengths:** Exact discrete adjoint consistency when paired with Backward Projection; efficient GPU implementation using bricked layout [S1, unofficial].
- **Limitations:** Requires specific volume layout (bricked, padded); limited to float32 on CUDA backend [S1, unofficial].

## Background
Developed as part of the TorchTomo library by BIAI Lab [S1, unofficial]. It is designed to provide differentiable forward projection for cone beam CT reconstruction, supporting exact adjoint consistency with the backward projection operator [S1, unofficial].

## Sources
[S1] itu-biai/torchtomo, https://github.com/itu-biai/torchtomo, fetched 2026-10-27 (unofficial)
