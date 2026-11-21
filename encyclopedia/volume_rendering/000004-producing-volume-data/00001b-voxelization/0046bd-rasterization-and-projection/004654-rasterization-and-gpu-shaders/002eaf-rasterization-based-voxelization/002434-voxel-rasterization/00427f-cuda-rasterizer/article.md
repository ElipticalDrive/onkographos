# CUDA Rasterizer

Also called: CUDA Rasterization Forward Kernel, Custom CUDA Rasterization

## Overview
The CUDA Rasterizer is a custom implementation of forward and backward CUDA kernels for rasterizing triangle meshes into 3D voxel grids and computing gradients for vertex positions. It supports both full-image rendering and batched pixel-query modes, enabling differentiable rendering for neural reconstruction tasks [S1]. It is distinguished from sibling methods by its explicit use of atomic operations for gradient accumulation and its specific grid/block launch configurations optimized for the underlying hardware, rather than relying on standard graphics APIs or sparse voxel structures [S2, S2, unofficial].

## How it works
The method consists of two distinct kernels: a forward kernel for rasterization and interpolation, and a backward kernel for gradient computation.

### Inputs and Outputs
**Forward Kernel Inputs:**
- `volume`: 3D tensor representing the voxel grid.
- `vertices`: Array of 3D vertex coordinates.
- `faces`: Array of face indices (triplets of vertex indices).
- `shape`: Metadata defining the dimensions of the volume.

**Forward Kernel Outputs:**
- Updated `volume` tensor with interpolated values.

**Backward Kernel Inputs:**
- `grad_vertices`: Gradient tensor for vertex positions.
- `volume`: The volume tensor from the forward pass.
- `grad_volume`: Gradient of the loss with respect to the volume.
- `vertices`: Vertex coordinates.
- `faces`: Face indices.
- `shape`: Volume dimensions.

**Backward Kernel Outputs:**
- Updated `grad_vertices` array.

### Data Structures
- **Vertices**: Stored as a list or tensor of 3D coordinates $(x, y, z)$.
- **Faces**: Stored as a list or tensor of triplets, where each triplet contains three integer indices referencing the `vertices` array.
- **Volume**: A 3D tensor (likely $W \times H \times D$) storing scalar or vector values per voxel.
- **Grad Vertices**: A tensor of the same shape as `vertices`, storing partial derivatives $\frac{\partial L}{\partial V_i}$.

### Forward Kernel Steps
1. **Kernel Launch Configuration**:
   The kernel is launched with a grid dimension of $(\lceil W / 32.0 \rceil, \lceil H / 16.0 \rceil, \lceil D / 2.0 \rceil)$ and a block dimension of $(32, 16, 2)$ [S2, unofficial].

2. **Vertex Retrieval**:
   For each face $i$, retrieve the 3D coordinates of its three vertices:
   $$ V_0 = \text{vertices}[\text{faces}[i][0]] $$
   $$ V_1 = \text{vertices}[\text{faces}[i][1]] $$
   $$ V_2 = \text{vertices}[\text{faces}[i][2]] $$

3. **Gradient Computation**:
   For each vertex $V_k$ (where $k \in \{0, 1, 2\}$), compute the gradient contribution by calling `compute_grad`. This function takes:
   - The other two vertices of the face.
   - The current vertex $V_k$.
   - A location $P$ (likely the voxel center or projection point).
   - The current voxel value.
   - The gradient of the voxel.
   - A diagonal parameter.
   
   *Note: The source code contains a TODO indicating `compute_grad` should take `volume` and `grad_volume` at location $P$ as input [S2, unofficial].*

4. **Atomic Gradient Accumulation**:
   The computed gradients $\text{grad}_{V_0}, \text{grad}_{V_1}, \text{grad}_{V_2}$ are atomically added to the corresponding entries in the `grad_vertices` array using `atomicAdd` for each of the three coordinate components (0, 1, 2) [S2, unofficial].

### Backward Kernel Steps
1. **Kernel Launch Configuration**:
   The backward kernel uses the same grid $(\lceil W / 32.0 \rceil, \lceil H / 16.0 \rceil, \lceil D / 2.0 \rceil)$ and block $(32, 16, 2)$ dimensions as the forward kernel [S2, unofficial].

2. **Type Dispatch**:
   The kernel execution is dispatched using `AT_DISPATCH_FLOATING_TYPES` for the type of `grad_vertices` to support different floating-point precisions [S2, unofficial].

3. **Gradient Propagation**:
   The kernel computes gradients for vertex positions based on the input `grad_volume`, `volume`, `vertices`, and `faces`. The specific mathematical derivation for the gradient of the rasterization/interpolation operation is encapsulated within the kernel logic, relying on the `compute_grad` logic or its inverse [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Forward Kernel Pseudocode
def cuda_rasterizer_forward(volume, vertices, faces, shape):
    # Grid: (ceil(W/32), ceil(H/16), ceil(D/2))
    # Block: (32, 16, 2)
    
    # Initialize grad_vertices if needed for backward pass later
    grad_vertices = zeros_like(vertices)
    
    for each face i in faces:
        # Retrieve vertices
        idx0, idx1, idx2 = faces[i]
        V0 = vertices[idx0]
        V1 = vertices[idx1]
        V2 = vertices[idx2]
        
        # Determine voxel/point P and current voxel value
        # (Details of P calculation are not explicitly provided in sources)
        P = compute_point_from_face(V0, V1, V2, shape)
        voxel_val = volume[P]
        
        # Compute gradients for each vertex
        # Note: compute_grad signature is partially defined in sources
        grad_V0 = compute_grad(V1, V2, V0, P, voxel_val, grad_volume[P], diagonal_param)
        grad_V1 = compute_grad(V0, V2, V1, P, voxel_val, grad_volume[P], diagonal_param)
        grad_V2 = compute_grad(V0, V1, V2, P, voxel_val, grad_volume[P], diagonal_param)
        
        # Atomic addition to grad_vertices
        atomic_add(grad_vertices[idx0], grad_V0)
        atomic_add(grad_vertices[idx1], grad_V1)
        atomic_add(grad_vertices[idx2], grad_V2)
        
    return volume, grad_vertices

# Backward Kernel Pseudocode
def cuda_rasterizer_backward(grad_vertices, volume, grad_volume, vertices, faces, shape):
    # Grid and Block same as forward
    # Dispatch based on grad_vertices dtype
    
    # The backward pass propagates gradients from grad_volume back to grad_vertices
    # The exact formula depends on the forward interpolation/rasterization logic
    # which is not fully detailed in the provided text, but involves:
    # 1. Identifying affected voxels/vertices
    # 2. Computing partial derivatives dL/dV_i using chain rule through the rasterization function
    
    for each face i in faces:
        idx0, idx1, idx2 = faces[i]
        V0 = vertices[idx0]
        V1 = vertices[idx1]
        V2 = vertices[idx2]
        
        # Compute gradients based on grad_volume and geometry
        # (Specific implementation details of the backward pass are not fully provided)
        # The source indicates the kernel takes grad_vertices as input, 
        # suggesting it might accumulate or update existing gradients.
        
        # Placeholder for the specific backward computation logic
        # which would mirror the forward interpolation but in reverse
        pass
        
    return grad_vertices
```

### Parameters and Edge Cases
- **Grid/Block Dimensions**: Fixed to $(\lceil W / 32.0 \rceil, \lceil H / 16.0 \rceil, \lceil D / 2.0 \rceil)$ and $(32, 16, 2)$ respectively [S2, unofficial].
- **Degenerate Meshes**: The surrounding system detects when the mesh becomes degenerate (e.g., vertices becoming colinear) and re-meshes, although this rarely occurs in practice [S1].
- **Precision**: The backward kernel supports floating-point types via `AT_DISPATCH_FLOATING_TYPES` [S2, unofficial].

## Subtopics

### CUDA Rasterization Backward Kernel
A variant of the CUDA Rasterizer that specifically handles the backward pass for gradient computation. It shares the same launch configuration (grid and block dimensions) as the forward kernel but takes `grad_vertices`, `volume`, `grad_volume`, `vertices`, `faces`, and `shape` as inputs. It uses `AT_DISPATCH_FLOATING_TYPES` to handle different data types for `grad_vertices` [S2, unofficial].

## Operating characteristics
- **Performance**: Training using this rasterizer takes between 1-3 minutes on an RTX A6000 Ada GPU [S1].
- **Complexity**: The complexity is tied to the number of faces and the volume dimensions, with atomic operations potentially introducing contention in high-density scenarios.
- **Strengths**: Custom CUDA implementation allows for fine-grained control over memory access and parallelism, supporting both full-image rendering and batched pixel-query modes [S1].
- **Limitations**: The reliance on atomic operations for gradient accumulation can be a bottleneck if many faces project to the same voxel or if there is high overlap. The specific grid/block configuration is fixed, which may not be optimal for all aspect ratios or resolutions.

## Background
The CUDA Rasterizer is part of the "Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction" framework [S1]. It is implemented in the `cvlab-epfl/voxel2mesh` repository [S2, unofficial]. The method was developed to enable differentiable rendering in neural reconstruction pipelines, allowing gradients to flow from the rendered voxel output back to the mesh vertices.

## Sources
[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] cvlab-epfl/voxel2mesh, https://github.com/cvlab-epfl/voxel2mesh, fetched 2026-10-27 (unofficial)
