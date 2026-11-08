# FDK Backprojection Transpose (Adjoint)

Also called: FDK Backprojection Transpose CUDA Kernel

## Overview
The FDK Backprojection Transpose computes the adjoint of the forward FDK projection operator, scattering voxel values from a 3D volume back to 2D projection panels. It solves the problem of computing the gradient of the data fidelity term with respect to the projection data in differentiable tomographic reconstruction. Unlike the forward pass which projects volume to projections, this method accumulates projection values into a buffer using atomic operations, serving as the transpose of the linear system matrix $A$ in the equation $Ax=b$.

## How it works
**Inputs:**
- `volume`: 3D tensor of shape `[batch, 1, depth, height, width]` containing voxel values.
- `matrices`: 4D tensor of shape `[batch, views, 12]` containing camera extrinsics (source and detector positions/orientations).
- `projection_shape`: Tuple `(views, rows, columns)` defining the output projection grid dimensions.
- `box`: Bounding box of the volume (used for ray clipping).

**Outputs:**
- `out`: 4D tensor of shape `[batch, 1, views, rows, columns]` initialized to zero, accumulating the scattered values.

**Data Structures:**
- `ViewColumn`: Struct containing precomputed geometric weights and ray parameters for a specific view.
- `Stencil`: Struct containing 4 indices and 4 weights for bilinear interpolation onto the projection panel.
- `ConeRay`: Struct containing ray origin, direction, and weight.

**Parameters:**
- `TT_FDK_DEPTH`: 4 (number of voxels processed per thread along the z-axis).
- `C`: Channel count (typically 1 for grayscale).

**Algorithm Steps:**

1.  **Initialization**:
    - Detach and convert `matrices` to contiguous float32.
    - Create `out` tensor of shape `[batch, 1, views, rows, columns]` filled with zeros.
    - Extract dimensions: `batch`, `depth`, `height`, `width` from `volume.shape`; `views`, `rows`, `columns` from `projection_shape`.

2.  **Kernel Launch**:
    - For each item in the batch:
        - Launch CUDA kernel `fdk_backproject_transpose_c1` with grid size `(depth, height, width)` and block size `_VOLUME_BLOCK`.
        - Pass arguments: `volume[item, 0]`, `matrices`, `out[item, 0]`, `width`, `height`, `depth`, `views`, `rows`, `columns`.

3.  **Kernel Execution (Per Thread)**:
    - Each thread handles a block of voxels along the z-axis: `z0 = blockIdx.z * TT_FDK_DEPTH`.
    - Thread indices `(x, y)` map to volume coordinates.
    - Load `TT_FDK_DEPTH` voxel values from `volume` at `(x, y, z0...z0+3)`.
    - If all loaded values are zero, skip the view loop.

4.  **Scatter Loop (Per View)**:
    - For each view `v` from `0` to `views-1`:
        - Compute `ViewColumn` parameters from `matrices`.
        - For each of the `TT_FDK_DEPTH` voxels (index `j`):
            - If `value[j] == 0`, continue.
            - Compute ray path through the volume for this voxel.
            - For each of the 4 stencil points (bilinear interpolation corners on the detector):
                - If `s.weight[q] != 0`:
                    - Perform `atomicAdd(out[s.index[q]], s.weight[q] * value[j])`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def fdk_backproject_transpose(volume, matrices, projection_shape, box):
    """
    Computes the adjoint of the FDK projection.
    volume: [B, 1, D, H, W]
    matrices: [B, V, 12]
    projection_shape: (V, R, C)
    box: [x_min, x_max, y_min, y_max, z_min, z_max]
    """
    B, _, D, H, W = volume.shape
    V, R, C = projection_shape
    
    # Initialize output
    out = torch.zeros((B, 1, V, R, C), dtype=torch.float32)
    
    # Precompute matrices if necessary (detached, contiguous, float)
    mats = matrices.detach().float().contiguous()
    
    # Kernel launch logic (conceptual, as CUDA kernel is not pure Python)
    # In practice, this calls the C++/CUDA extension
    for b in range(B):
        # Grid: (D, H, W) / block_size
        # Block: (TX, TY, 1)
        # Arguments: volume[b,0], mats[b], out[b,0], W, H, D, V, R, C, box
        pass 
        
    return out

# Conceptual kernel logic for one thread
def thread_kernel_logic(x, y, z0, volume, mats, out, W, H, D, V, R, C, box):
    TT_FDK_DEPTH = 4
    C_channels = 1
    
    # Load voxels
    values = []
    for k in range(TT_FDK_DEPTH):
        z = z0 + k
        if z < D:
            values.append(volume[x, y, z])
        else:
            values.append(0.0)
            
    if all(v == 0.0 for v in values):
        return

    for view in range(V):
        # Extract view parameters from mats
        # This involves computing ray origin, direction, and geometric weights
        # See forward FDK for ray construction details
        ray = compute_cone_ray(mats[view], box)
        
        for j in range(TT_FDK_DEPTH):
            if values[j] == 0.0:
                continue
                
            # Project voxel (x, y, z0+j) to detector
            # This is the inverse of the forward projection's sampling
            # In the adjoint, we scatter the voxel value to the detector pixels
            # that would have contributed to this voxel in the forward pass.
            
            # Compute detector coordinates (u, v) for this voxel
            u, v_coord = project_to_detector(ray, x, y, z0 + j)
            
            # Bilinear interpolation stencil on detector
            stencil = compute_bilinear_stencil(u, v_coord, R, C)
            
            for q in range(4):
                if stencil.weights[q] != 0.0:
                    # Atomic add to the corresponding detector pixel
                    idx = stencil.indices[q]
                    out[0, 0, view, idx // C, idx % C] += stencil.weights[q] * values[j]

def compute_bilinear_stencil(u, v, rows, cols):
    """
    Computes the 4 corners and weights for bilinear interpolation.
    Returns indices and weights.
    """
    u_floor = math.floor(u)
    v_floor = math.floor(v)
    
    dc = u - u_floor
    dr = v - v_floor
    
    # Weights
    w00 = (1.0 - dc) * (1.0 - dr)
    w10 = dc * (1.0 - dr)
    w01 = (1.0 - dc) * dr
    w11 = dc * dr
    
    # Indices (flattened)
    # Check bounds
    indices = []
    weights = []
    
    corners = [
        (u_floor, v_floor),
        (u_floor + 1, v_floor),
        (u_floor, v_floor + 1),
        (u_floor + 1, v_floor + 1)
    ]
    
    ws = [w00, w10, w01, w11]
    
    for i, (cu, cv) in enumerate(corners):
        if 0 <= cu < cols and 0 <= cv < rows:
            indices.append(int(cv * cols + cu))
            weights.append(ws[i])
        else:
            indices.append(-1)
            weights.append(0.0)
            
    return type('Stencil', (), {'indices': indices, 'weights': weights})()
```

**Edge Cases and Pitfalls:**
- **Atomic Contention**: Multiple threads may write to the same projection pixel simultaneously. The implementation uses `atomicAdd`. Note that `float32` addition is not associative, so the order of accumulation matters for numerical consistency with the forward pass. The code relies on the specific execution order of the CUDA kernel to maintain adjoint correctness [S1, unofficial].
- **Boundary Conditions**: Stencil weights are clamped to zero if the interpolated coordinate falls outside the detector bounds `[0, rows) x [0, columns)` [S1, unofficial].
- **Zero Voxel Skip**: If all `TT_FDK_DEPTH` voxels loaded by a thread are zero, the view loop is skipped to save computation [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(B \cdot V \cdot D \cdot H \cdot W)$, where $B$ is batch size, $V$ is views, $D,H,W$ are volume dimensions. Each voxel is processed once per view.
- **Memory Complexity**: $O(B \cdot V \cdot R \cdot C)$ for the output buffer.
- **Performance**: The kernel is optimized for CUDA, using shared memory for view parameters and atomic operations for accumulation. Performance is limited by atomic contention on high-resolution projections or large batch sizes.
- **Strengths**: Efficiently computes the adjoint for differentiable tomography. Integrates seamlessly with PyTorch via custom CUDA kernels.
- **Limitations**: Atomic contention can reduce throughput. Numerical precision issues may arise due to non-associative floating-point addition in `atomicAdd`.

## Background
The FDK Backprojection Transpose is part of the `torchtomo` library, which provides differentiable tomographic reconstruction tools. It implements the adjoint of the FDK (Feldkamp-Davis-Kress) projection operator, enabling gradient-based optimization in tomographic imaging. The method was developed to support end-to-end differentiable pipelines in medical imaging and materials science.

## Sources
[S1] itu-biai/torchtomo, https://github.com/itu-biai/torchtomo, fetched 2026-10-27 (unofficial)
