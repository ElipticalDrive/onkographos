# Slice-based Solid Voxelization

Also called: Dynamic solid slice function voxelization, GPU-accelerated multi-valued solid voxelization by slice functions, Real-time Solid Voxelization, Solid Slice Function Voxelization, Solid Voxelization by Slice Functions

## Overview
Slice-based Solid Voxelization is a GPU-accelerated algorithm that converts closed surface-based geometric objects into multi-valued solid volumetric models [S1]. Unlike binary voxelization methods that only determine occupancy, this method generates interior material properties (e.g., colors, textures, noise) for every voxel within the object's volume [S1]. It distinguishes itself from its parent [Single-pass GPU Solid Voxelization](3323-single-pass-gpu-solid-voxelization.md) by explicitly supporting diverse, non-uniform interior attributes via dynamic slice functions, and from siblings like [Single-pass Solid Voxelization via XOR Bitmask Accumulation](3325-single-pass-solid-voxelization-via-xor-bitmask-accumulation.md) by avoiding logical XOR operations that result in binary outputs and missing voxels [S1].

## How it works
The method processes the volume slice-by-slice along a specified axis (typically Z), using the GPU's stencil buffer to determine interior parity and a user-defined "slice function" to assign voxel values [S1].

### Inputs and Outputs
*   **Input:**
    *   $O$: A closed surface-based geometric object (mesh).
    *   $N_x, N_y, N_z$: The resolution of the output volume.
    *   $B_{min}, B_{max}$: The 3D bounding box of the object.
    *   $SolidSliceFunc()$: A function defining the value of voxels inside the object.
    *   $L$: A display list or geometry shader program representing $O$.
*   **Output:**
    *   $V$: A 3D texture volume containing the voxelized solid object.

### Data Structures
*   **3D Texture Volume ($V$):** Stores the final voxel data. Dimensions $N_x \times N_y \times N_z$.
*   **Stencil Buffer:** Used as a mask to track surface parity (inside/outside).
*   **Depth Buffer:** Used for standard depth testing during rasterization.
*   **Color Buffer:** Temporarily holds the result of the slice function before masking.

### Parameters
*   **Resolution:** Arbitrary, demonstrated up to $512^3$ [S1].
*   **Stencil Buffer Depth:** 8-bit (sufficient for parity toggling) [S1].
*   **Depth Buffer Depth:** 24-bit [S1].
*   **Slice Function Types:**
    *   `FUNC_SINGLE_COLOR`: Constant color.
    *   `FUNC_COLOR_RAMP`: Color varies with Z.
    *   `FUNC_NOISE_WOOD/MARBLE/ROCK`: Procedural noise functions [S1].
    *   `FUNC_MY_TYPE*`: Customized functions (e.g., translucent jade) [S1].

### Algorithm Steps

1.  **Initialization:**
    *   Set viewport to $(0, 0, N_x, N_y)$.
    *   Set orthogonal projection matrix.
    *   Set background color to $(0, 0, 0, 0)$.
    *   Transform object $O$ so its voxelization orientation aligns with the Z-axis.
    *   Generate display list $L$ for drawing $O$.
    *   Initialize or preprocess $SolidSliceFunc()$.

2.  **Slice Loop:**
    *   Iterate $i$ from $0$ to $N_z - 1$.
    *   Check if slice $i$ is inside the bounding box defined by $B_{min}$ and $B_{max}$.
    *   If inside, call `CreateSolidSlice(L, i, SolidSliceFunc())`.
    *   Insert the resulting slice $S_i$ into volume $V$.

3.  **CreateSolidSlice(L, i, SolidSliceFunc):**
    *   Clear color, stencil, and depth buffers.
    *   Initialize the depth buffer for the current slice plane.
    *   **Parity Check:** Rasterize geometry $L$ clipped by the current slice plane. Use the stencil buffer to toggle a parity flag for front/back faces. This determines which pixels are "inside" the solid object [S1].
    *   **Function Application:** Apply $SolidSliceFunc()$ to the entire slice plane into the color buffer.
    *   **Masking:** Use the stencil buffer as a mask.
        *   If stencil value is 1 (inside), keep the color buffer value.
        *   If stencil value is 0 (outside), clear the color buffer to the background color $(0, 0, 0, 0)$ [S1].
    *   Read the color buffer to obtain the solid slice $S_i$.

4.  **Output:**
    *   Return volume $V$.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solid_voxelization(O, Nx, Ny, Nz, Bmin, Bmax, SolidSliceFunc):
    """
    Main voxelization routine.
    O: Closed surface mesh
    Nx, Ny, Nz: Volume dimensions
    Bmin, Bmax: Bounding box vectors
    SolidSliceFunc: Function returning voxel value for (x, y, z)
    """
    # 1. Setup
    set_viewport(0, 0, Nx, Ny)
    set_orthographic_projection()
    set_background_color(0, 0, 0, 0)
    
    # Transform O to align with Z-axis
    O_transformed = transform_to_z_axis(O)
    L = generate_display_list(O_transformed)
    
    # Initialize slice function if needed
    SolidSliceFunc.preprocess()
    
    # Volume container
    V = empty_3d_texture(Nx, Ny, Nz)
    
    # 2. Slice Loop
    for i in range(Nz):
        # Check if slice is within bounding box
        if is_inside_bounding_box(i, Bmin, Bmax):
            slice_data = create_solid_slice(L, i, SolidSliceFunc)
            V.set_slice(i, slice_data)
            
    return V

def create_solid_slice(L, i, SolidSliceFunc):
    """
    Generates a single 2D slice at index i.
    """
    # Clear buffers
    clear_color_buffer()
    clear_stencil_buffer()
    clear_depth_buffer()
    
    # Initialize depth buffer for the slice plane
    init_depth_buffer_for_slice(i)
    
    # Step 1: Parity Check using Stencil Buffer
    # Rasterize geometry L. 
    # The stencil buffer toggles parity for front/back faces.
    # This marks interior pixels with stencil value 1.
    rasterize_geometry(L, stencil_mode=TOGGLE_PARITY)
    
    # Step 2: Apply Slice Function
    # Fill the entire color buffer with the slice function result
    # Note: The function is evaluated for all pixels (x, y) at z=i
    fill_color_buffer_with_function(SolidSliceFunc, z=i)
    
    # Step 3: Masking
    # Use stencil buffer to mask out exterior voxels
    # If stencil == 1 (inside), keep color. Else, clear to background.
    apply_stencil_mask(background_color=(0, 0, 0, 0))
    
    # Step 4: Readback
    slice_data = read_color_buffer()
    
    return slice_data
```

### Formulas
The $i$-th solid slice $S_i$ is described as [S1]:
$$ S_i(x, y) = \{ f(x, y, i) \} \in \mathbb{R}^3 $$
where $x = 1, \dots, N_x$, $y = 1, \dots, N_y$, and $i = 1, \dots, N_z$.

The volumetric object $V$ is the set of slices [S1]:
$$ V(z) = \{ S_i, i = 1, \dots, N_z \} $$

The solid slice function $f_s$ can be a fixed value, $f(x,y)$, $f(x,y,z)$, or a customized function $f_{customized}(x,y,z)$ [S1].

### Edge Cases and Pitfalls
*   **Complex Functions:** Computing complex slice functions (e.g., Perlin noise) takes significantly more time than simple color-based functions [S1].
*   **Anti-aliasing:** The surface boundary of the volumetric object is not generated smoothly in 2D or 3D space, leading to aliasing artifacts [S1].
*   **Performance Scaling:** Running time is directly influenced by the number of slices ($N_z$) due to the slice-based nature of the algorithm [S1].
*   **Memory Swapping:** Unlike older methods that required swapping data between GPU and CPU, this method stores slices directly into 3D texture memory, avoiding this bottleneck [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of slices $N_z$ and the complexity of the slice function. Performance for simple functions (e.g., `FUNC_SINGLE_COLOR`) is highly efficient, taking less than 0.5ms for $512^3$ resolution even for models with 1M polygons (e.g., Buddha) [S1]. Color-ramp functions also perform under 0.5ms for $128^3$ and $256^3$ resolutions [S1].
*   **Memory Complexity:** Requires memory for the 3D texture volume ($N_x \times N_y \times N_z$) and GPU buffers (color, depth, stencil). The method avoids CPU-GPU memory swapping for intermediate slices [S1].
*   **Hardware Requirements:** Tested on Nvidia GeForce 7800 GTX with 256MB texture memory, 8-bit stencil buffer, and 24-bit depth buffer [S1]. Requires support for 3D texture mapping [S1].
*   **Strengths:** Real-time performance; supports multi-valued (non-binary) interiors; slice-independent processing allows for parallelization; supports arbitrary voxelization orientations [S1].
*   **Limitations:** Aliasing at boundaries; performance degrades with complex procedural slice functions; dependent on $N_z$ for runtime [S1].

## Background
This method was introduced by Duoduo Liao and George Washington University in 2008 [S1]. It extends previous stencil-buffer-based solid voxelization techniques [Liao 2001, Liao 2002] by combining them with GPU acceleration and dynamic slice functions. Prior methods, such as Chen and Fang [2000], used logical XOR to generate binary volumes with missing voxels, while Liao and Fang [2002] required drawing the object twice and suffered from memory swapping issues [S1]. This approach was the first to propose dynamic solid slice functions to generate diverse solid volumes with various interior materials (rainbow, marble, wood, etc.) in real time [S1]. It is suitable for applications like volume modeling, collision detection, medical simulation, and 3D printing [S1].

## Sources
[S1] GPU-accelerated multi-valued solid voxelization by slice functions in real time, https://doi.org/10.1145/1477862.1477886, fetched 2026-11-06
