# GPU Backprojection with Global Memory Buffer

Also called: GPU Backprojection Kernel Mapping, backproject

## Overview
This method performs cone-beam CT reconstruction by mapping projection weights to volume expansion coefficients using a GPU-accelerated kernel that iterates over projection angles. It solves the inverse problem of tomographic reconstruction by accumulating contributions from filtered projections into a global memory volume buffer. It is distinguished from sibling methods by its use of a single kernel launch per projection angle with a 1D NDRange where each work-item corresponds to one voxel, avoiding complex local memory tiling or image texture sampling [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** A sequence of $P$ filtered projection images, each of size $W \times H$. Geometric parameters: Source-to-Object Distance ($sod$), Source-to-Detector Distance ($sdd$), voxel size ($voxelSize$), and pixel size ($pixelSize$). Volume dimensions $NX \times NX \times NY$.
- **Outputs:** A reconstructed 3D volume buffer of size $NX \times NX \times NY$ of type `float`.

**Data Structures:**
- **Volume Buffer:** A 1D array of size $V = NX \times NX \times NY$ allocated in global memory (`CL_MEM_READ_WRITE`). Initialized to zero.
- **Projection Buffer:** A 1D array of size $W \times H$ allocated in global memory (`CL_MEM_READ_ONLY | CL_MEM_COPY_HOST_PTR`) for the current projection angle.

**Numbered Steps:**
1. Initialize OpenCL context, command queue, and program from source file `kernels/backprojection_buffer.cl` [S1, unofficial].
2. Allocate the volume buffer of size $V \times sizeof(float)$ and fill with zeros [S1, unofficial].
3. Loop over each projection angle $p$ from $0$ to $P-1$:
    a. Calculate the rotation angle: $\theta_p = 2.0 \pi \frac{p}{P}$ [S1, unofficial].
    b. Create the projection buffer containing the $p$-th filtered projection slice [S1, unofficial].
    c. Set kernel arguments: `projBuffer`, `volumeBuffer`, $W$, $H$, $NX$, $NY$, $voxelSize$, $pixelSize$, $sdd$, $sod$, $\theta_p$ [S1, unofficial].
    d. Launch the backprojection kernel with a global size equal to $V$ (1D NDRange) [S1, unofficial].
    e. Release the projection buffer [S1, unofficial].
4. After all projections, read the volume buffer back to the host [S1, unofficial].
5. Scale the reconstructed volume by $\frac{\pi}{P}$ [S1, unofficial].

**Kernel Logic (Per Work-Item):**
Each work-item with global ID $gid$ corresponds to a unique voxel in the volume. The voxel coordinates $(x, y, z)$ are derived from $gid$. The kernel computes the contribution of the current projection to this voxel:
1. Map the voxel $(x, y, z)$ to the detector plane coordinates $(u, v)$ using the cone-beam geometry defined by $sod$, $sdd$, and $\theta_p$ [S1, unofficial].
2. Sample the projection buffer at $(u, v)$ using interpolation (e.g., bilinear) [S1, unofficial].
3. Add the sampled value to the voxel at $gid$ in the `volumeBuffer` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Kernel: backprojection_buffer.cl
# Global ID gid maps to voxel index.
# Dimensions: NX, NY, W, H.
# Parameters: voxelSize, pixelSize, sdd, sod, angle.

def backproject_kernel(proj_buffer, volume_buffer, W, H, NX, NY, voxel_size, pixel_size, sdd, sod, angle):
    gid = get_global_id(0)
    if gid >= NX * NX * NY:
        return

    # 1. Map 1D gid to 3D voxel coordinates (x, y, z)
    # Assuming row-major order: z = gid / (NX*NX), rem = gid % (NX*NX), y = rem / NX, x = rem % NX
    z = gid // (NX * NX)
    rem = gid % (NX * NX)
    y = rem // NX
    x = rem % NX

    # 2. Convert voxel indices to physical coordinates
    # Center of the volume is at (0,0,0)
    x_phys = (x - (NX - 1) / 2.0) * voxel_size
    y_phys = (y - (NX - 1) / 2.0) * voxel_size
    z_phys = (z - (NY - 1) / 2.0) * voxel_size

    # 3. Rotate voxel coordinates by -angle around Y-axis to align with projection frame
    # Rotation matrix for angle theta around Y-axis:
    # x' = x*cos(theta) + z*sin(theta)
    # z' = -x*sin(theta) + z*cos(theta)
    cos_a = cos(angle)
    sin_a = sin(angle)
    
    x_rot = x_phys * cos_a + z_phys * sin_a
    z_rot = -x_phys * sin_a + z_phys * cos_a
    y_rot = y_phys

    # 4. Project to detector coordinates (u, v)
    # Using similar triangles: u = x_rot * (sdd / sod)
    # v = y_rot * (sdd / sod)
    # Note: z_rot is used to determine if the voxel is within the cone beam, 
    # but for simple backprojection into the volume, we assume valid geometry.
    scale = sdd / sod
    u = x_rot * scale
    v = y_rot * scale

    # 5. Map detector coordinates to pixel indices
    # Center of detector is at (W/2, H/2)
    u_idx = u / pixel_size + W / 2.0
    v_idx = v / pixel_size + H / 2.0

    # 6. Sample projection buffer
    # Check bounds
    if u_idx < 0 or u_idx >= W or v_idx < 0 or v_idx >= H:
        weight = 0.0
    else:
        # Bilinear interpolation or nearest neighbor
        # For simplicity, using nearest neighbor or clamped access
        u_int = int(round(u_idx))
        v_int = int(round(v_idx))
        # Clamp to valid range
        u_int = max(0, min(W - 1, u_int))
        v_int = max(0, min(H - 1, v_int))
        weight = proj_buffer[v_int * W + u_int]

    # 7. Accumulate into volume buffer
    volume_buffer[gid] += weight

# Host-side loop
def reconstruct_gpu_buffer(projections, P, W, H, NX, NY, voxel_size, pixel_size, sdd, sod):
    # 1. Initialize OpenCL context, queue, program
    # 2. Allocate volume buffer (CL_MEM_READ_WRITE) of size NX*NX*NY, fill with 0
    volume_buffer = allocate_buffer(NX * NX * NY, 0.0)
    
    # 3. Loop over projections
    for p in range(P):
        angle = 2.0 * PI * p / P
        # 4. Create projection buffer with p-th projection
        proj_buffer = allocate_buffer(W * H, projections[p])
        
        # 5. Set kernel arguments
        # 6. Launch kernel with global size NX*NX*NY
        launch_kernel(proj_buffer, volume_buffer, W, H, NX, NY, voxel_size, pixel_size, sdd, sod, angle)
        
        # 7. Release projection buffer
        release_buffer(proj_buffer)
        
    # 8. Read volume buffer back to host
    result = read_buffer(volume_buffer)
    
    # 9. Scale by PI / P
    result = [val * PI / P for val in result]
    
    return result
```

**Parameters:**
- $P$: Total number of projection angles [S1, unofficial].
- $NX, NY$: Volume dimensions. Volume size is $NX \times NX \times NY$ [S1, unofficial].
- $W, H$: Projection image dimensions [S1, unofficial].
- $voxelSize$: Physical size of a voxel [S1, unofficial].
- $pixelSize$: Physical size of a pixel [S1, unofficial].
- $sdd$: Source-to-Detector Distance [S1, unofficial].
- $sod$: Source-to-Object Distance [S1, unofficial].
- Typical work-item count: $128 \times 128 \times 128 = 2,097,152$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point accumulation errors may occur if $P$ is very large. [G] Use double precision for intermediate geometric calculations if necessary.
- **Boundaries:** Voxel projections falling outside the detector bounds $(0 \le u < W, 0 \le v < H)$ must be handled to avoid out-of-bounds memory access. [G] Clamp or check bounds before sampling.
- **Degenerate Input:** If $sod = 0$ or $sdd = 0$, division by zero occurs. [G] Validate inputs before kernel launch.

## Subtopics

### [Forward Projection](4740-forward-projection.md)
The adjoint operator to backprojection, mapping volume to projections.

### [GPU Backprojection using OpenCL Buffers](5110-gpu-backprojection-using-opencl-buffers.md)
Uses OpenCL buffers for both input and output, similar to this method but potentially with different kernel optimization strategies.

### [GPU Backprojection using OpenCL Image Textures](5111-gpu-backprojection-using-opencl-image-textures.md)
Uses image textures for projection data, leveraging hardware interpolation and caching.

## Operating characteristics
- **Time Complexity:** $O(P \cdot V)$, where $P$ is the number of projections and $V$ is the number of voxels. Each voxel is processed once per projection.
- **Memory Complexity:** $O(V + W \cdot H)$. The volume buffer is stored in global memory, and one projection buffer is allocated and reused.
- **Performance:** The method is efficient for moderate resolution volumes due to the simplicity of the kernel and lack of complex synchronization. However, global memory bandwidth may become a bottleneck for very large volumes or high-resolution projections. [S1, unofficial]
- **Strengths:** Simple implementation, no need for local memory management, easy to debug.
- **Limitations:** High global memory traffic as each voxel reads from the projection buffer. Not optimized for cache coalescing as effectively as tiled approaches.
- **When to use:** Suitable for prototyping, moderate resolution CT reconstruction, and when implementation simplicity is prioritized over maximum performance.
- **When not to use:** For very large volumes or real-time applications where memory bandwidth is a critical constraint.

## Background
This method is part of the GPU-accelerated cone-beam CT reconstruction pipeline described in the `ct-reconstruction-gpu` repository. It serves as a baseline GPU implementation for backprojection, contrasting with more optimized versions using local memory or image textures. The approach follows the standard FDK (Feldkamp-Davis-Kress) algorithm structure, where filtered projections are backprojected into a volume.

## Sources
[S1] hardik0001/ct-reconstruction-gpu, https://github.com/hardik0001/ct-reconstruction-gpu, fetched 2026-10-27 (unofficial)
