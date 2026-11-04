# Parallel Dual Marching Cube


## Overview
Parallel Dual Marching Cube (PDMC) is an intersection-free isosurface extraction method that processes a 3D signed distance field (SDF) grid on the GPU to produce a triangle or quadrilateral mesh [S1, unofficial]. It extends Dual Marching Cubes by parallelizing the cell processing and using a dual mesh topology where connected patches of triangulation in a cell become single vertices, and crossed grid edges yield quads connecting adjacent cells' patch vertices [S1, unofficial]. Unlike standard Marching Cubes which produces triangles, this method produces quadrilaterals, offering better surface quality for smooth fields while maintaining topological correctness in most cases [S1, unofficial]. It is implemented as a CUDA kernel accessible via a PyTorch C++ extension, designed for differentiable rendering pipelines like PaMO [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D signed distance field (SDF) grid tensor `grid` of shape `[dimX, dimY, dimZ]` residing on a CUDA device. The values must be contiguous.
*   **Parameters:**
    *   `isovalue` (scalar): The threshold for extraction. Default: `0.0` [S1, unofficial].
    *   `dtype` (`torch.float32` or `torch.float64`): Precision for vertex coordinates and grid processing [S1, unofficial].
    *   `return_quads` (bool): If `True`, output quad faces `[F, 4]`; if `False`, output triangle faces `[F, 3]` [S1, unofficial].
    *   `normalize` (bool): If `True`, vertices are normalized to `[0, 1]`; if `False`, they remain in `[0, dim-1]` [S1, unofficial].
*   **Outputs:**
    *   `verts`: Tensor of shape `[V, 3]` containing vertex coordinates.
    *   `faces`: Tensor of shape `[F, 3]` or `[F, 4]` containing face indices (starting from 0) [S1, unofficial].

**Data Structures:**
The CUDA kernel uses the following device storage structures [S1, unofficial]:
*   `Vertex<T>`: Struct with fields `x, y, z` of type `T`, stored contiguously.
*   `Quad<T>`: Struct with fields `i, j, k, l` of type `T`, stored contiguously.
*   `Triangle<T>`: Struct with fields `i, j, k` of type `T`, stored contiguously.
*   `Edge<T>`: Struct with fields `i, j` of type `T`, stored contiguously.
*   `CUDualMC` device storage:
    *   `temp_storage`: `IndexType*` for prefix sums.
    *   `first_cell_used`: `IndexType*`.
    *   `used_cell_index`: `IndexType*`.
    *   `used_to_first_mc_vert`: `IndexType*`.
    *   `used_cell_code`: `uint8_t*`.
    *   `used_to_first_mc_patch`: `IndexType*`.
    *   `mc_vert_to_cell`: `IndexType*`.
    *   `mc_vert_to_edge`: `Edge<Vertex<Scalar>>*`.
    *   `mc_vert_type`: `uint8_t*` (values 0-2 indicate x/y/z entering, 3-5 indicate x/y/z exiting) [S1, unofficial].
    *   `quads`: `Quad<IndexType>*`.
    *   `tris`: `Triangle<IndexType>*`.
    *   `first_tris_create`: `IndexType*`.
    *   `first_verts_create`: `IndexType*`.
    *   `verts`: `Vertex<Scalar>*`.

**Linearization Functions:**
To map 3D grid coordinates to 1D indices, the method uses [S1, unofficial]:
$$gA(x, y, z) = z + \text{dims}[2] \times (y + \text{dims}[1] \times x)$$
where $x$ is the slowest-varying dimension and $z$ is the fastest.

The inverse linearization functions are [S1, unofficial]:
$$gX(\text{id}) = \text{id} / (\text{dims}[2] \times \text{dims}[1])$$
$$gY(\text{id}) = (\text{id} / \text{dims}[2]) \% \text{dims}[1]$$
$$gZ(\text{id}) = \text{id} \% \text{dims}[2]$$

**Algorithm Steps:**
1.  **Degeneracy Check:** Check if the input grid values are entirely above or below the isovalue. If `grid.min() >= isovalue` or `grid.max() <= isovalue`, return empty vertex tensor `[0, 3]` and face tensor `[0, 4]` (or `[0, 3]` if `return_quads` is False) to skip degenerate cases [S1, unofficial].
2.  **Padding:** Pad the input SDF grid with a 1-voxel border filled with `isovalue + 1` on all six sides to handle boundary conditions during extraction [S1, unofficial]. The padded grid has shape `[dimX+2, dimY+2, dimZ+2]`.
3.  **Kernel Launch:** Pass the padded grid and isovalue to the CUDA kernel. The kernel populates device buffers for vertices, quads, and triangles using the `CUDualMC` storage structures. Each thread block or thread processes one or more cells, determining intersections and generating dual mesh vertices and faces [S1, unofficial].
4.  **Coordinate Offset:** Subtract 1 from all output vertex coordinates to offset the padding added in step 2 [S1, unofficial].
5.  **Normalization (Optional):** If `normalize` is enabled, divide vertex coordinates by `(dimX-1, dimY-1, dimZ-1)` to map them to the `[0, 1]` range. Otherwise, keep them in `[0, dim-1]` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def parallel_dual_marching_cube(grid: torch.Tensor, isovalue: float = 0.0, return_quads: bool = False, normalize: bool = True, dtype: torch.dtype = torch.float32) -> tuple:
    """
    Extracts an isosurface from a 3D SDF grid using Parallel Dual Marching Cube.
    
    Args:
        grid: 3D tensor of shape (dimX, dimY, dimZ) on CUDA device.
        isovalue: Scalar threshold.
        return_quads: If True, returns quad faces; else triangle faces.
        normalize: If True, normalizes vertices to [0, 1].
        dtype: Precision type.
        
    Returns:
        verts: Tensor of shape (V, 3).
        faces: Tensor of shape (F, 3) or (F, 4).
    """
    # Check if grid is on CUDA device and contiguous
    if not grid.is_cuda or not grid.is_contiguous():
        raise ValueError("Input grid must be contiguous and reside on a CUDA device.")

    # Step 1: Degeneracy Check
    if grid.min() >= isovalue or grid.max() <= isovalue:
        return torch.zeros((0, 3), dtype=dtype, device=grid.device), torch.zeros((0, 4), dtype=torch.int32, device=grid.device)

    dimX, dimY, dimZ = grid.shape
    
    # Step 2: Padding
    # Pad with isovalue + 1 on all sides
    padded_grid = F.pad(grid, (1, 1, 1, 1, 1, 1), "constant", isovalue + 1)
    
    # Step 3: Kernel Launch
    # The actual CUDA kernel call is handled by the C++ extension.
    # Here we simulate the interface described in the source.
    # In practice, this calls the compiled CUDMCFloat or CUDMCDouble forward method.
    # verts_raw, quads, tris = _C.CUDualMC.forward(padded_grid, isovalue)
    # For the purpose of this article, we assume the kernel produces raw vertices and faces.
    # Note: The source code shows the kernel returns verts, quads, tris.
    # We must select between quads and tris based on return_quads.
    # The source code returns quads and tris separately, and the Python wrapper
    # likely combines or selects them. The source example shows:
    # verts, quads, tris = self.func.apply(grid, isovalue)
    # And then returns verts and either quads or tris.
    
    # Simulating the kernel output (implementation detail hidden in C++ extension)
    # verts_raw, quads, tris = cuda_kernel(padded_grid, isovalue)
    
    # Since we cannot run the CUDA kernel here, we describe the post-processing.
    # The kernel outputs vertices in the padded coordinate system.
    
    # Step 4: Coordinate Offset
    # Subtract 1 from all output vertex coordinates
    # verts = verts_raw - 1
    
    # Step 5: Normalization
    # if normalize:
    #     verts = verts / (torch.tensor([dimX, dimY, dimZ], dtype=verts.dtype, device=grid.device) - 1)
    
    # Return placeholder for the sake of valid Python structure, 
    # as the core logic is in the CUDA kernel.
    # In a real implementation, the C++ extension handles the kernel launch and memory management.
    raise NotImplementedError("This is a description of the method. The actual CUDA kernel is in the C++ extension.")

# Note: The actual implementation uses a PyTorch C++ extension.
# The Python wrapper in the source code is:
# class DMC(nn.Module):
#     def __init__(self, dtype=torch.float32):
#         super().__init__()
#         self.dtype = dtype
#         if dtype == torch.float32:
#             dmc = _C.CUDMCFloat()
#         elif dtype == torch.float64:
#             dmc = _C.CUDMCDouble()
#         ...
#     def forward(self, grid, isovalue=0.0, return_quads=False, normalize=True):
#         if grid.min() >= isovalue or grid.max() <= isovalue:
#             return torch.zeros((0, 3), dtype=self.dtype, device=grid.device), torch.zeros((0, 4), dtype=torch.int32, device=grid.device)
#         dimX, dimY, dimZ = grid.shape
#         grid = F.pad(grid, (1, 1, 1, 1, 1, 1), "constant", isovalue+1)
#         verts, quads, tris = self.func.apply(grid, isovalue)
#         verts = verts - 1
#         if normalize:
#             verts = verts / (torch.tensor([dimX, dimY, dimZ], dtype=verts.dtype, device=grid.device) - 1)
#         if return_quads:
#             return verts, quads
#         else:
#             return verts, tris
```

## Subtopics

### [Dual Marching Cubes with GPU Mesh Simplification](2139-dual-marching-cubes-with-gpu-mesh-simplification.md)
Extends PDMC by adding a GPU-based mesh simplification step to reduce vertex count and improve mesh quality. It uses an asymptotic decider to resolve ambiguities and outputs a quadrilateral mesh stored in a halfedge data structure. Choose this when mesh size reduction is critical for downstream applications like optimization or rendering, while maintaining topological correctness.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of cells in the grid, $O(N)$, where $N = dimX \times dimY \times dimZ$, as each cell is processed in parallel by the GPU kernel.
*   **Memory Complexity:** Linear with respect to the number of vertices and faces generated. The device storage requires buffers for temporary prefix sums and output arrays.
*   **Performance:** Measured performance depends on GPU hardware and grid resolution. The method is designed for high-performance extraction on modern GPUs using CUDA.
*   **Strengths:** Intersection-free, parallel processing on GPU, supports both triangle and quad outputs, differentiable (via PyTorch integration).
*   **Limitations:** Requires CUDA-capable GPU, input grid must be contiguous and on device, limited to 3D grids.
*   **When to use:** When extracting isosurfaces from large 3D SDF grids on the GPU with a need for intersection-free results and potential mesh simplification.
*   **When not to use:** When running on CPU-only systems, or when the input grid is not a signed distance field or does not reside on the GPU.

## Background
Parallel Dual Marching Cube was introduced as part of the PaMO (Parallel Mesh Optimization) project by Oh et al. in 2025 [S1, unofficial]. It is an intersection-free iso-surface extraction module used in [PaMO](https://github.com/SarahWeiii/pamo). The method builds on the Dual Marching Cubes approach, adapting it for parallel execution on the GPU. It is implemented as a PyTorch C++ extension, allowing it to be integrated into differentiable rendering pipelines. The source code is available on GitHub as `seonghunn/pdmc` [S1, unofficial].

## Sources
[S1] seonghunn/pdmc, https://github.com/seonghunn/pdmc, fetched 2026-10-27 (unofficial)
