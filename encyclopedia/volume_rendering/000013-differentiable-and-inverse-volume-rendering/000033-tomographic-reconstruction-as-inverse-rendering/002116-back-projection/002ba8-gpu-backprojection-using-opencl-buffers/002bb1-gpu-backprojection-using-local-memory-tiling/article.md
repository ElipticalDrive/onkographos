# GPU Backprojection using Local-Memory Tiling

Also called: GPU Backprojection with Local Memory Tiling, GPU-Local

## Overview
A GPU-accelerated tomographic reconstruction variant that optimizes memory access patterns by tiling the Z-axis reconstruction loop using OpenCL local memory [S1, unofficial]. It reduces global memory bandwidth pressure by reusing projection data across threads within a work-group, distinguishing it from the parent [GPU Backprojection using OpenCL Buffers](5110-gpu-backprojection-using-opencl-buffers.md) which relies solely on global memory buffers for projection storage [S1, unofficial].

## How it works
**Inputs:**
- `projections`: `std::vector<float>` of size $P \times W \times H$ [S1, unofficial].
- `P`: Number of projections [S1, unofficial].
- `W`, `H`: Detector width and height in pixels [S1, unofficial].
- `NX`, `NY`: Volume dimensions ($NX \times NX \times NY$) [S1, unofficial].
- `voxelSize`: Voxel size in mm [S1, unofficial].
- `pixelSize`: Pixel size in mm [S1, unofficial].
- `sdd`: Source-to-detector distance in mm [S1, unofficial].
- `sod`: Source-to-object distance in mm [S1, unofficial].

**Outputs:**
- `std::vector<float>` representing the reconstructed volume of size $NX \times NX \times NY$ [S1, unofficial].

**Data Structures:**
- **Volume Buffer**: Global memory buffer of size $NX \times NX \times NY$ floats, initialized to zero [S1, unofficial].
- **Projection Buffer**: Global memory buffer containing pre-filtered projections [S1, unofficial].
- **Local Memory Buffers**: Two local memory buffers (`col0`, `col1`) of size $H$ floats each, used to cache projection columns for interpolation [S1, unofficial].

**Parameters:**
- `TILE_Z`: Tile size along the Z-axis, defined as 64 [S1, unofficial].
- **Local Memory Requirement**: $2 \times H \times \text{sizeof(float)}$ bytes [S1, unofficial].

**Steps:**
1. **Pre-filtering**: Apply the FDK pre-filter (ramp filter and cosine weighting) on the CPU using `applyFDKPreFilter` before GPU execution [S1, unofficial].
2. **Device Check**: Check the device's local memory size. If insufficient, the code prints info but proceeds without explicit fallback [S1, unofficial].
3. **Kernel Loading**: Load the kernel source from `kernels/backprojection_local.cl` [S1, unofficial].
4. **Volume Allocation**: Allocate the volume buffer and fill it with zeros [S1, unofficial].
5. **NDRange Calculation**:
   - Local size: $(1, 1, \text{TILE\_Z})$ [S1, unofficial].
   - Global Z size: Round up $NY$ to the nearest multiple of `TILE_Z`: $gz = ((NY + \text{TILE\_Z} - 1) / \text{TILE\_Z}) \times \text{TILE\_Z}$ [S1, unofficial].
   - Global X, Y sizes: $NX$ [S1, unofficial].
6. **Projection Loop**: Loop over projections $p$ from $0$ to $P-1$ [S1, unofficial].
   - Calculate the projection angle based on $p$ and $P$ [G].
   - Create the projection buffer and set kernel arguments. Arguments 2 and 3 are local memory buffers (`col0`, `col1`) passed as size in bytes (address is `nullptr`) [S1, unofficial].
   - Launch the kernel with the calculated 3D NDRange (global size, local size) [S1, unofficial].
7. **Post-processing**: Multiply all volume voxels by $\pi / P$ [S1, unofficial].

**Formulas:**
- $t = (x - 0.5 \cdot (NX - 1)) \cdot \text{voxelSize}$ [G]
- $z_m = (z - 0.5 \cdot (NY - 1)) \cdot \text{voxelSize}$ [S1, unofficial]
- $\text{invU} = \text{sdd} / (\text{sdd} + t)$ [G]
- $u_{\text{proj}} = \text{sdd} \cdot \text{invU} \cdot t$ [S1, unofficial]
- $v_{\text{proj}} = \text{sdd} \cdot \text{invU} \cdot z_m$ [S1, unofficial]
- $u_{\text{idx}} = 0.5 \cdot (W - 1) - u_{\text{proj}} / \text{pixelSize}$ [S1, unofficial]
- $v_{\text{idx}} = 0.5 \cdot (H - 1) - v_{\text{proj}} / \text{pixelSize}$ [S1, unofficial]
- $\text{weight} = (\text{sod} \cdot \text{sod}) \cdot (\text{invU} \cdot \text{invU})$ [S1, unofficial]
- Bilinear interpolation of projection values at $(u_{\text{idx}}, v_{\text{idx}})$ using cached columns [S1, unofficial]
- $\text{volume}[x \cdot NX \cdot NY + y \cdot NY + z] \mathrel{+}= \text{interpolated} \cdot \text{weight}$ [S1, unofficial]
- $\text{volume}[i] \mathrel{*}= \pi / P$ for all $i$ [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reconstructGPU_Local(projections, P, W, H, NX, NY, voxelSize, pixelSize, sdd, sod):
    TILE_Z = 64
    filtered_projections = applyFDKPreFilter(projections, P, W, H, voxelSize, sdd, pixelSize)
    volume = [0.0] * (NX * NX * NY)
    local_size = (1, 1, TILE_Z)
    gz = ((NY + TILE_Z - 1) // TILE_Z) * TILE_Z
    global_size = (NX, NX, gz)
    kernel_source = load_kernel_source('kernels/backprojection_local.cl')
    kernel = compile_kernel(kernel_source)
    
    for p in range(P):
        angle = (p / P) * 2.0 * 3.14159265358979323846
        proj_buffer = create_buffer(filtered_projections, p, W, H)
        vol_buffer = create_buffer(volume)
        col0_size = H * 4
        col1_size = H * 4
        
        set_kernel_arg(kernel, 0, vol_buffer)
        set_kernel_arg(kernel, 1, proj_buffer)
        set_kernel_arg(kernel, 2, col0_size)
        set_kernel_arg(kernel, 3, col1_size)
        set_kernel_arg(kernel, 4, P)
        set_kernel_arg(kernel, 5, W)
        set_kernel_arg(kernel, 6, H)
        set_kernel_arg(kernel, 7, NX)
        set_kernel_arg(kernel, 8, NY)
        set_kernel_arg(kernel, 9, voxelSize)
        set_kernel_arg(kernel, 10, pixelSize)
        set_kernel_arg(kernel, 11, sdd)
        set_kernel_arg(kernel, 12, sod)
        set_kernel_arg(kernel, 13, angle)
        
        launch_kernel(kernel, global_size, local_size)
        volume = read_buffer(vol_buffer)
        
    scale = 3.14159265358979323846 / P
    for i in range(len(volume)):
        volume[i] *= scale
    return volume
```

**Kernel Logic (Conceptual):**
The kernel iterates over X and Y coordinates [S1, unofficial]. For each voxel $(x, y, z)$, it calculates the projection coordinates $(u, v)$ on the detector [S1, unofficial]. It uses local memory to cache two columns of the projection image (`col0` and `col1`) to facilitate bilinear interpolation of the projection value at $(u, v)$ [S1, unofficial]. The interpolated value is weighted by $(\text{sod} \cdot \text{sod}) \cdot (\text{invU} \cdot \text{invU})$ and accumulated into the volume [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Local Memory Overflow**: If $2 \times H \times \text{sizeof(float)}$ exceeds the device's local memory limit, the behavior is undefined or the kernel may fail. The source checks this but proceeds without fallback [S1, unofficial].
- **Boundary Conditions**: Projection coordinates outside $[0, W-1] \times [0, H-1]$ are skipped [S1, unofficial].
- **Precision**: Uses `float` for volume accumulation. Large numbers of projections may lead to precision loss [G].

## Operating characteristics
- **Time Complexity**: $O(P \cdot NX \cdot NX \cdot NY)$, same as CPU backprojection, but optimized for GPU parallelism [G].
- **Memory Complexity**: $O(P \cdot W \cdot H + NX \cdot NX \cdot NY + H)$ for global buffers and local memory [S1, unofficial].
- **Performance**: Reduces global memory bandwidth pressure by reusing projection data in local memory [S1, unofficial].
- **Strengths**: Efficient memory access pattern for Z-axis reconstruction [S1, unofficial].
- **Limitations**: Requires sufficient local memory ($2 \cdot H \cdot 4$ bytes). No fallback if local memory is insufficient [S1, unofficial].

## Background
This method is part of a GPU-accelerated CT reconstruction toolkit, specifically optimizing the backprojection step using OpenCL local memory tiling [S1, unofficial]. It builds on the FDK algorithm and CPU-based pre-filtering [S1, unofficial].

## Sources
[S1] hardik0001/ct-reconstruction-gpu, https://github.com/hardik0001/ct-reconstruction-gpu, fetched 2026-10-27 (unofficial)
