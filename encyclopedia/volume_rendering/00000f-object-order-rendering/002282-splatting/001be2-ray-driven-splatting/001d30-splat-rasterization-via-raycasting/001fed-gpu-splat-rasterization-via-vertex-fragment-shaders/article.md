# GPU Splat Rasterization via Vertex/Fragment Shaders

Also called: Point Primitive Splatting, Shader-based Splatting

## Overview
This method implements splat rasterization on GPUs by simulating a dedicated rasterization unit using standard graphics pipeline stages: vertex shaders for transformation and bounding box calculation, point primitives for fragment generation, and fragment shaders for kernel evaluation and depth computation [S1]. It solves the problem of efficiently projecting 3D splats onto the screen while maintaining accurate depth values and filtering weights, optimizing for minimal instruction count in the fragment shader [S1]. It is more efficient than previous GPU splatting implementations because the fragment shader requires only three computational instructions (two MADs and one texture fetch), whereas previous approaches were more expensive [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of splats, each defined by a center position, orientation, and shape parameters (implicitly defining the inverse mapping matrix $\mathbf{N}_k$).
*   **Output:** A rasterized image with accumulated attributes and depth values, where each fragment corresponds to a pixel covered by a splat.

**Data Structures:**
*   **Splat Data:** Each splat $k$ is represented by its inverse mapping matrix $\mathbf{N}_k$ (a $4 \times 4$ matrix) and its center position.
*   **Kernel Texture:** A 2D texture storing the precomputed 2D reconstruction kernel weights $r_0(x)$ [S1].

**Steps:**
1.  **Vertex Shader Processing:**
    *   For each splat, compute the inverse mapping matrix $\mathbf{N}_k$ [S1].
    *   Compute a conservative 2D axis-aligned bounding box (AABB) of the projected splat in screen space [S1].
    *   Set the position of the point primitive to the center of this bounding box.
    *   Set the size of the point primitive to the largest dimension of the bounding box [S1].

2.  **Fragment Generation:**
    *   The GPU hardware rasterizer generates fragments for the point primitive, covering the area of the bounding box [S1].

3.  **Fragment Shader Processing:**
    *   Receive the matrix $\mathbf{N}_k$ transmitted as three column vectors (the fourth is implied or handled separately for depth) [S1].
    *   Perform the matrix multiplication to transform the fragment's screen-space coordinates into the splat's local space. This requires only two MAD (Multiply-Add) instructions [S1].
    *   Retrieve the weight $r_0(x)$ from the 2D kernel texture using a single projective texture fetch based on the transformed coordinates [S1].
    *   Extract the accurate window-space depth value from the fourth component of the matrix product result [S1].
    *   Discard fragments outside the splat using the alpha test unit (based on the weight or a threshold) [S1].
    *   Multiply the fragment's attributes by the weight $r_0(x)$ and accumulate them using the blending unit [S1].

**Formulas:**
*   Let $\mathbf{N}_k$ be the inverse mapping matrix for splat $k$.
*   Let $\mathbf{p}$ be the screen-space position of the fragment.
*   The transformed local coordinate $\mathbf{u}$ is computed as $\mathbf{u} = \mathbf{N}_k \mathbf{p}$ (specifically, the first two components for kernel lookup, and the fourth component for depth) [S1].
*   The weight is $w = r_0(\mathbf{u})$ [S1].
*   The depth is $z = (\mathbf{N}_k \mathbf{p})_4$ [S1].

**Parameters:**
*   Kernel texture resolution: Not given in the sources.
*   Alpha test threshold: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on accurate depth extraction from the matrix product. Precision issues may arise if the matrix values are very large or small [S1].
*   **Boundaries:** The conservative bounding box ensures all fragments are generated, but may include some fragments that are ultimately discarded, leading to wasted computation [S1].
*   **Minification:** In cases of minification and high frequencies, resampling filters may miss pixels leading to aliasing artifacts. This is addressed by band-limiting reconstruction kernels via convolution with a screen-space pre-filter, as in the original EWA splatting framework [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
def vertex_shader(splat):
    # Compute inverse mapping matrix N_k
    N_k = compute_inverse_mapping_matrix(splat)
    
    # Compute conservative 2D AABB of projected splat
    bbox = compute_2d_aabb(N_k, splat.center)
    
    # Set point primitive attributes
    point_position = bbox.center
    point_size = max(bbox.width, bbox.height)
    
    return point_position, point_size, N_k

# Fragment Shader
def fragment_shader(point_position, point_size, N_k, fragment_screen_pos):
    # Transmit N_k as three column vectors (N_k_col1, N_k_col2, N_k_col3)
    # Compute transformed local coordinates using two MAD instructions
    # u = N_k * fragment_screen_pos
    # This involves matrix-vector multiplication
    u_x = mad(N_k_col1.x, fragment_screen_pos.x, mad(N_k_col2.x, fragment_screen_pos.y, mad(N_k_col3.x, fragment_screen_pos.z, N_k_col4.x * fragment_screen_pos.w)))
    u_y = mad(N_k_col1.y, fragment_screen_pos.x, mad(N_k_col2.y, fragment_screen_pos.y, mad(N_k_col3.y, fragment_screen_pos.z, N_k_col4.y * fragment_screen_pos.w)))
    depth = mad(N_k_col1.z, fragment_screen_pos.x, mad(N_k_col2.z, fragment_screen_pos.y, mad(N_k_col3.z, fragment_screen_pos.z, N_k_col4.z * fragment_screen_pos.w))) # Note: Source says 4th component, assuming homogeneous coords
    
    # Retrieve weight from precomputed 2D kernel texture
    weight = texture_fetch(kernel_texture, u_x, u_y)
    
    # Discard if outside splat (alpha test)
    if weight < alpha_threshold:
        discard_fragment()
    
    # Multiply attributes by weight and accumulate
    accumulated_attributes = fragment_attributes * weight
    
    return accumulated_attributes, depth
```
*Note: The pseudocode above simplifies the matrix multiplication. The source specifies that transmitting the matrix as three column vectors allows the multiplication to be done with two MAD instructions. The exact mapping of matrix elements to MAD operations depends on the specific GPU instruction set and how the matrix is packed. The depth is explicitly stated to be the fourth component of the matrix product result.*

## Subtopics

### GPU Splatting and Voxelization
This variant focuses on voxelizing particle data by rendering screen-aligned bounding quads using an OpenGL geometry shader to project particle spheres onto the screen [S2]. For each generated fragment, the particle sphere is intersected with the corresponding camera ray, and all overlapped voxels in the corresponding column are marked using atomic bitwise operations [S2]. For particles with radius < 7 pixels on NVIDIA GTX Titan, a simple splatting CUDA kernel can be used for parallel processing, which projects particles onto the screen and loops over all pixels in an axis-aligned bounding quad sequentially [S2]. CUDA does not expose atomics on textures directly; they are accessible via the 'sured' instruction using inline PTX code [S2]. On GTX 970, pure OpenGL-based rendering is always faster than CUDA splatting [S2].

## Operating characteristics
**Time and Memory Complexity:**
*   The fragment shader complexity is very low, requiring only three computational instructions (two MADs and one texture fetch) [S1].
*   Memory usage depends on the number of splats and the kernel texture size [S1].

**Measured Performance:**
*   On NVIDIA GTX 970, splatting performance for 1024x1024 tile size is 76 ms, compression is 28 ms [S2].
*   On NVIDIA GTX 970, splatting performance for 512x512 tile size is 80 ms, compression is 28 ms [S2].
*   On NVIDIA GTX 970, splatting performance for 256x256 tile size is 182 ms, compression is 32 ms [S2].
*   On NVIDIA GTX 970, splatting performance for 128x128 tile size is 546 ms, compression is 47 ms [S2].
*   On NVIDIA GTX 970, splatting performance for 1024x512 tile size is 58 ms, compression is 28 ms [S2].

**Scaling:**
*   Performance scales with tile size and number of particles [S2].
*   SIMT efficiency is improved when particles are partially sorted by the simulation, grouping particles of approximately equal distance and size [S2].

**Strengths and Limitations:**
*   **Strengths:** High efficiency due to minimal fragment shader instructions [S1]. Simple implementation using standard graphics pipeline stages [S1].
*   **Limitations:** Conservative bounding box may generate unnecessary fragments [S1]. Aliasing artifacts may occur in minification without pre-filtering [S1]. CUDA-based splatting is slower than OpenGL-based rendering on GTX 970 [S2].

**When to Use:**
*   Use when high performance is required and the GPU supports efficient MAD instructions and texture fetching [S1].
*   Use for real-time rendering of point-based models [S1].

**When Not to Use:**
*   Avoid if aliasing artifacts are unacceptable and pre-filtering is not implemented [S1].
*   Avoid if using CUDA for splatting on GTX 970, as OpenGL is faster [S2].

## Background
This method was introduced in the paper "Splat/Mesh Blending, Perspective Rasterization and Transparency for Point-Based Rendering" [S1]. It builds on previous GPU-based splatting implementations [GP03,BK03,ZRB∗04,BSK04,BHZK05] by optimizing the fragment shader to use only three computational instructions [S1]. The approach is part of the broader field of point-based rendering and splatting techniques [S1].

## Sources
[S1] Splat/Mesh Blending, Perspective Rasterization and Transparency for Point-Based Rendering, https://doi.org/10.2312/spbg/spbg06/049-057, fetched 2026-10-26
[S2] Memory-Efficient On-The-Fly Voxelization of Particle Data, https://doi.org/10.2312/pgv.20151150, fetched 2026-10-26
