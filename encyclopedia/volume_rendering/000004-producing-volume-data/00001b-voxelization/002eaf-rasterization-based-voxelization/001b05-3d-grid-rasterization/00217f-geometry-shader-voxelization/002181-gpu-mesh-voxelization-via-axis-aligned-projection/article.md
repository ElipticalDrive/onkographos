# GPU Mesh Voxelization via Axis-Aligned Projection

Also called: Dominant Axis Projection, Dominant Axis Voxelization, Dominant Axis Projection Voxelization, Voxelization with Dominant Axis Projection, GPU Voxelization via Rasterization and Geometry Shader Axis Projection

## Overview
GPU Mesh Voxelization via Axis-Aligned Projection converts triangle meshes into a 3D voxel grid by rasterizing each triangle orthographically along its dominant normal axis [S1, S2, S6, S7, unofficial]. It solves the "missing voxel" artifact inherent in single-direction rasterization, where surfaces viewed edge-on leave gaps in the 3D texture [S1, S7, S9, unofficial]. Unlike [GPU Voxelization Using OpenGL Image Atomics](2544-gpu-voxelization-using-opengl-image-atomics.md) which may require multiple passes or complex atomic logic for depth, and [GPU Mesh Voxelization via AABB-Triangle Intersection](2541-gpu-mesh-voxelization-via-aabb-triangle-intersection.md) which relies on geometric intersection tests, this method uses the GPU's native rasterizer to fill fragments, ensuring maximal coverage per triangle with a single pass [S1, S8, S9, S10, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh (vertices, indices, normals, colors) [S1, S3, unofficial].
*   **Output:** A 3D texture (voxel map) or structured buffer containing voxel colors and occupancy flags [S1, S2, S3, S4, unofficial].

**Data Structures:**
*   **Voxel Grid:** A 3D texture of resolution $W \times H \times D$ (typically powers of 2, e.g., 64, 128, 256) [S4, S7, S9, unofficial].
    *   *Format:* RGBA (e.g., `DXGI_FORMAT_R8G8B8A8_UNORM` or `float4`) [S1, S3, S4, unofficial].
    *   *Content:* RGB stores direct diffuse lighting (Phong model); Alpha stores solidity (0=empty, 1=surface) [S1, unofficial].
*   **Dummy Render Target:** A 2D texture of size $W \times H$ used to trigger fragment generation [S3, S4, S7, unofficial].
*   **Unordered Access View (UAV):** The target for writing voxel data, either the 3D texture directly or a structured buffer [S2, S3, S4, S6, unofficial].
*   **Constant Buffer:** Contains model-to-voxel projection matrices, grid dimensions ($W, H, D$), and strides [S3, S4, unofficial].

**Numbered Steps:**

1.  **Initialization:** Create the 3D voxel texture and clear all values to $(0,0,0,0)$ [S1, S3, unofficial]. Bind a dummy 2D render target and the voxel UAV [S3, S4, unofficial]. Set viewport to $(W, H)$ [S1, S3, unofficial]. Disable depth testing and face culling to ensure all triangles are processed regardless of orientation or occlusion [S1, S6, S10, unofficial].

2.  **Vertex Shader:** Pass input position, normal, and color to the Geometry Shader [S3, S7, unofficial]. Apply the model-to-voxel orthographic projection matrix [S1, S3, unofficial].

3.  **Geometry Shader (Dominant Axis Selection):**
    For each input triangle with vertices $v_0, v_1, v_2$ and positions $p_0, p_1, p_2$:
    a. Calculate the unnormalized face normal (area vector) $\vec{n}$:
       $$ \vec{n} = \text{abs}((p_1 - p_0) \times (p_2 - p_0)) $$
    b. Determine the dominant axis index `ind` by finding the maximum component of $\vec{n}$:
       $$ \text{ind} = \arg\max(\vec{n}_x, \vec{n}_y, \vec{n}_z) $$
       *   If $\vec{n}_x$ is max, `ind` = 0 (Project onto YZ plane) [S1, S7, unofficial].
       *   If $\vec{n}_y$ is max, `ind` = 1 (Project onto XZ plane) [S1, S7, unofficial].
       *   If $\vec{n}_z$ is max, `ind` = 2 (Project onto XY plane) [S1, S7, unofficial].
    c. Reorder vertex components to align the dominant axis with the view direction (Z-axis in NDC) [S1, S5, S7, unofficial]:
       *   If `ind` == 0: Output vertices as $(z, y, x)$ [S1, unofficial].
       *   If `ind` == 1: Output vertices as $(x, z, y)$ [S1, unofficial].
       *   If `ind` == 2: Output vertices as $(x, y, z)$ [S1, unofficial].
    d. Emit the reordered vertices [S1, S7, unofficial].

4.  **Pixel Shader (Voxel Writing):**
    For each rasterized fragment:
    a. Convert 3D position to voxel grid indices using the formula $i = (p / s + 1) \times r$, where $p$ is the fragment's 3D position, $s$ is the physical scale of the voxel volume (bounding box size), and $r$ is the grid resolution [S2, unofficial]. Apply this component-wise to $x, y, z$.
    b. Calculate the color. Use the **original** (pre-rotation) surface normal for lighting calculations to avoid incorrect Phong shading due to the geometric rotation [S1, unofficial].
    c. Write the color and alpha (1.0) to the voxel at $(u, v, d)$ in the 3D texture or UAV [S1, S4, unofficial].
       *   *Concurrency:* If writing to a structured buffer, use atomic counters or Raster Order Groups to handle simultaneous writes from different triangles [S2, S4, S6, unofficial].
       *   *Overwriting:* If multiple triangles map to the same voxel, the last fragment written wins, or an atomic max on luminance can be used to keep the brightest color [S2, S7, S11, unofficial].

**Pseudocode (Geometry Shader Logic):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Geometry Shader Dominant Axis Selection
# Input: Triangle vertices v0, v1, v2 with world positions p0, p1, p2
# Output: Reordered vertices for rasterization along dominant axis

def process_triangle(v0, v1, v2):
    # 1. Calculate unnormalized normal (area vector)
    edge1 = v1.pos - v0.pos
    edge2 = v2.pos - v0.pos
    normal = cross(edge1, edge2)
    
    # 2. Take absolute values to handle winding order
    abs_normal = abs(normal)
    
    # 3. Determine dominant axis and reorder vertices
    # Mapping from [S1, unofficial]: align dominant axis with view direction (Z in NDC)
    if abs_normal.x >= abs_normal.y and abs_normal.x >= abs_normal.z:
        # X is dominant -> map X to Z. New=(Z, Y, X)
        reorder = lambda p: (p.z, p.y, p.x)
    elif abs_normal.y >= abs_normal.x and abs_normal.y >= abs_normal.z:
        # Y is dominant -> map Y to Z. New=(X, Z, Y)
        reorder = lambda p: (p.x, p.z, p.y)
    else:
        # Z is dominant -> map Z to Z. New=(X, Y, Z)
        reorder = lambda p: (p.x, p.y, p.z)
        
    # 4. Emit reordered vertices
    emit_vertex(reorder(v0.pos), v0.color, v0.normal_original)
    emit_vertex(reorder(v1.pos), v1.color, v1.normal_original)
    emit_vertex(reorder(v2.pos), v2.color, v2.normal_original)

def emit_vertex(pos, color, normal):
    # Standard rasterization step: pass attributes to fragment shader
    pass
```

**Pseudocode (Pixel Shader Logic):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Pixel Shader Voxel Writing
# Input: Fragment attributes (clip_pos, color, original_normal)
# Output: Write to 3D Texture or UAV

def fragment_shader(clip_pos, color, original_normal):
    # 1. Convert Clip Space to Voxel Grid Coordinates
    # Assuming orthographic projection maps [-1, 1] to [0, GridSize]
    ndc = (clip_pos.xyz + 1.0) * 0.5
    
    # 2. Determine Voxel Indices
    # Note: The 'axis' variable is passed from Geometry Shader
    # In a 3D texture write, we need x, y, z indices.
    # The rasterizer provides screen-space (u, v) and depth (z).
    # The dominant axis determines which grid dimension corresponds to the rasterization depth.
    
    # Example mapping for axis=2 (Z dominant, projected to XY):
    # u -> x, v -> y, clip_z -> z
    voxel_x = int(ndc.x * grid_width)
    voxel_y = int(ndc.y * grid_height)
    voxel_z = int(ndc.z * grid_depth)
    
    # Clamp to bounds
    voxel_x = clamp(voxel_x, 0, grid_width - 1)
    voxel_y = clamp(voxel_y, 0, grid_height - 1)
    voxel_z = clamp(voxel_z, 0, grid_depth - 1)
    
    # 3. Calculate Lighting (Phong)
    # Use original_normal, NOT the rotated one
    light_color = compute_phong_lighting(color, original_normal)
    
    # 4. Write to Voxel
    # If using 3D Texture:
    write_voxel(voxel_x, voxel_y, voxel_z, vec4(light_color, 1.0))
    
    # If using Structured Buffer with Atomic Counter (e.g., [S2, unofficial]):
    # voxel_buffer[atomic_counter++] = Voxel(voxel_x, voxel_y, voxel_z, light_color)

def compute_phong_lighting(color, normal):
    # Standard Phong lighting calculation using pre-rotation normal
    return color

def vec4(r, g, b, a):
    return (r, g, b, a)

def write_voxel(x, y, z, data):
    # Standard UAV/Texture write operation
    pass
```

**Parameters:**
*   **Grid Resolution:** $64^3$ to $512^3$ [S1, S7, S9, unofficial].
*   **MSAA:** 8x MSAA is often enabled on the dummy render target to catch thin/edge-on polygons that might otherwise be missed by standard rasterization [S7, unofficial].
*   **Conservative Rasterization:** Optional extension (e.g., `NV_conservative_raster`) to ensure any voxel intersected by the triangle's plane is filled, eliminating holes [S1, S5, S9, unofficial].

**Edge Cases and Pitfalls:**
*   **Lighting Errors:** Using the rotated normal for lighting calculations results in incorrect shading. The original normal must be preserved and passed through [S1, unofficial].
*   **Missing Voxels:** Without MSAA or conservative rasterization, thin triangles or those viewed nearly edge-on (even with dominant axis projection) may fail rasterization tests, leaving holes [S7, S9, unofficial].
*   **Overwriting:** Multiple triangles may map to the same voxel. The result depends on rasterization order. Using atomic operations or depth testing (if enabled) can control this [S2, S7, unofficial].
*   **Precision:** Floating-point precision in coordinate conversion can cause artifacts at high resolutions. Use integer arithmetic where possible for index calculation [G].

## Subtopics

### [GPU-based Conservative Voxelization](2580-gpu-based-conservative-voxelization.md)
Uses hardware conservative rasterization to guarantee voxel coverage for intersecting triangles, eliminating holes without MSAA. Choose when high fidelity is required and hardware supports it.

### [GPU Mesh Voxelization via AABB-Triangle Intersection](2541-gpu-mesh-voxelization-via-aabb-triangle-intersection.md)
Uses geometric intersection tests rather than rasterization. Choose when exact geometric accuracy is needed regardless of rasterization artifacts.

### [GPU Voxelization Using OpenGL Image Atomics](2544-gpu-voxelization-using-opengl-image-atomics.md)
Uses atomic operations on image buffers. Choose when direct buffer access is preferred over 3D textures.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of triangles. Each triangle is processed once in the geometry shader and rasterized once [S8, S12, unofficial].
*   **Memory Complexity:** $O(W \times H \times D)$ for the voxel grid [S1, unofficial].
*   **Performance:** Real-time performance (10+ fps) for scenes with millions of triangles at $256^3$ resolution on modern GPUs [S8, S12, unofficial]. On an NVIDIA GTX 770, $512^3$ voxelization took ~11.7 ms [S9, unofficial].
*   **Strengths:** Single-pass rasterization; leverages GPU hardware efficiency; simple implementation [S1, S8, unofficial].
*   **Limitations:** Requires MSAA or conservative rasterization for hole-free results; overwriting artifacts if not managed; limited to convex/concave meshes without complex handling [S7, S9, unofficial].
*   **When to Use:** Real-time global illumination (Voxel Cone Tracing), fast pre-computation for volumetric effects [S9, unofficial].
*   **When Not to Use:** When exact geometric boundary representation is required (use AABB intersection); when memory bandwidth is extremely limited and sparse structures are preferred [G].

## Background
The method originates from the need for efficient voxelization for Voxel Cone Tracing (VCT) [S1, S6, unofficial]. Key references include "Real-Time Voxelization for Complex Models" (Michael et al., 2010) [S8, unofficial], "Octree-Based Sparse Voxelization Using the GPU Hardware Rasterizer" (Cyril et al., 2011) [S10, unofficial], and "An Accurate Method for Voxelizing Polygon Meshes" (Huang et al., 1998) [S10, unofficial].

## Sources
[S1] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
[S2] The beautiful hack of voxelization, https://www.youtube.com/watch?v=cnH9gwrgpu8, fetched 2026-10-27 (unofficial)
[S3] nustxujun/AfterHumanDeclined, https://github.com/nustxujun/AfterHumanDeclined, fetched 2026-10-26 (unofficial)
[S4] michschw/GpuPro3Vox, https://github.com/michschw/GpuPro3Vox, fetched 2026-10-26 (unofficial)
[S5] 3D-Mesh-Voxelizer - GitHub, https://github.com/Talkative-Banana/3D-Mesh-Voxelizer, fetched 2026-10-26 (unofficial)
[S6] kakashidinho/VoxelConeTracingMetal, https://github.com/kakashidinho/VoxelConeTracingMetal, fetched 2026-10-26 (unofficial)
[S7] Voxel Cone Tracing Renderer - Graphics Programming Portfolio, https://otaviopeixoto1.github.io/portfolio/vctgi/, fetched 2026-10-27 (unofficial)
[S8] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
[S9] https://developer.nvidia.com/content/basics-gpu-voxelization, https://developer.nvidia.com/content/basics-gpu-voxelization, fetched 2026-10-28 (unofficial)
[S10] PDF K14190_text - Icare3D, https://icare3d.org/research/OpenGLInsights-SparseVoxelization.pdf, fetched 2026-10-29 (unofficial)
[S11] 【Unity】GPUで3Dモデルをリアルタイムにボクセル化する, https://qiita.com/seyakara/items/b635939f6c6106a15d40, fetched 2026-10-29 (unofficial)
[S12] (PDF) Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
[S13] Voxelization using the GPU hardware rasterizer | Alexandre Pestana, https://www.alexandre-pestana.com/voxelization-using-gpu-hardware-rasterizer/, fetched 2026-10-29 (unofficial)
[S14] GPU上でのvoxel構築手法 - shikihuiku - WordPress.com, https://shikihuiku.wordpress.com/2012/08/02/gpu%E4%B8%8A%E3%81%A7%E3%81%AEvoxel%E6%A7%8B%E7%AF%89%E6%89%8B%E6%B3%95/, fetched 2026-10-30 (unofficial)
