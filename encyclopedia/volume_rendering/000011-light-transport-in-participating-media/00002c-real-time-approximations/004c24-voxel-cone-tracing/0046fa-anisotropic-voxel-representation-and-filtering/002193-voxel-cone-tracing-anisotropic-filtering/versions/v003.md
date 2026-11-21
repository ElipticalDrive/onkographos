# Voxel Cone Tracing Anisotropic Filtering

Also called: Anisotropic Voxel Mipmaps, Anisotropic Voxels, Anisotropic Voxels for Improved Cone-Tracing, Directional Voxel Mipmaps, Octree-Based Sparse Voxelization anisotropic filtering, Two Red-Green Wall Problem Mitigation

## Overview
Voxel Cone Tracing Anisotropic Filtering addresses the "two red-green wall problem" and opacity under-estimation in standard isotropic voxel cone tracing, where averaging opaque voxels of different colors or mixing opaque/transparent voxels causes light leaking through thin geometry [S1, S4]. It replaces the single isotropic value per octree node with six directional values (along $+X, -X, +Y, -Y, +Z, -Z$) computed via axis-aligned volumetric integration, allowing the cone tracer to detect directional occlusion and prevent light leaks [S3, unofficial][S4]. Unlike its sibling [Anisotropic Voxel Representation](4963-anisotropic-voxel-representation.md), this method is specifically optimized for sparse voxel octrees and cone tracing by sharing the base mipmap level across all directions to reduce memory overhead [S3, S5, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A sparse voxel octree (or 3D texture) representing the scene at full resolution (leaf level), containing color and alpha (opacity) per voxel.
- **Output:** Six 3D texture arrays (or storage buffers), one for each major direction ($+X, -X, +Y, -Y, +Z, -Z$), containing the anisotropic mipmap chain. The base level (leaf) is identical for all directions and can be shared.

**Data Structures:**
- **Voxel Offsets:** An array of 8 `ivec3` coordinates defining the corners of a $2 \times 2 \times 2$ block relative to the block origin:
  $$ \text{voxel\_offsets} = \{ (1,1,1), (1,1,0), (1,0,1), (1,0,0), (0,1,1), (0,1,0), (0,0,1), (0,0,0) \} $$ [S2, unofficial].
- **Directional Textures:** Six `sampler3D` or `image3D` bindings, indexed $0$ to $5$, corresponding to directions $-X, +X, -Y, +Y, -Z, +Z$ respectively [S2, S3, unofficial].
- **Voxel Value:** A `vec4` representing color (RGB) and alpha (A) [S2, unofficial].

**Mipmap Generation (Pre-Integration):**
The anisotropic mipmap chain is generated using a compute shader. For each non-leaf octree cell (not at full resolution), six directional values are computed by performing volumetric integration along each axis [S4].

1.  **Boundary Check:** If the current thread index exceeds the resolution, return [S2, unofficial].
2.  **Calculate Positions:** Determine the write position and the origin of the $2 \times 2 \times 2$ child block [S2, unofficial]:
    ```python
    write_pos = gl_GlobalInvocationID
    block_pos = write_pos * 2
    ```
3.  **Fetch Children:** Fetch the 8 child voxel values from the source texture at the current mip level [S2, unofficial]:
    ```python
    values = fetch_texels(dir, block_pos)
    # values[i] = texelFetch(src_tex3D[dir], block_pos + voxel_offsets[i], mip)
    ```
4.  **Compositing (Volumetric Integration):** For each of the 6 directions, composite the 8 children front-to-back along that specific axis. The alpha blending function is defined as [S2, unofficial]:
    $$ \text{alpha\_blend}(v_0, v_1) = v_0 + (1 - v_0.a) \cdot v_1 $$
    The results for the 6 directions are computed as follows [S2, unofficial]:
    -   **Direction 0 (-X):** Average of compositing pairs along the X-axis (front-to-back in -X direction):
        $$ \frac{\text{blend}(v_0, v_4) + \text{blend}(v_1, v_5) + \text{blend}(v_2, v_6) + \text{blend}(v_3, v_7)}{4} $$
    -   **Direction 1 (+X):** Average of compositing pairs along the X-axis (front-to-back in +X direction):
        $$ \frac{\text{blend}(v_4, v_0) + \text{blend}(v_5, v_1) + \text{blend}(v_6, v_2) + \text{blend}(v_7, v_3)}{4} $$
    -   **Direction 2 (-Y):** Average of compositing pairs along the Y-axis:
        $$ \frac{\text{blend}(v_0, v_2) + \text{blend}(v_1, v_3) + \text{blend}(v_5, v_7) + \text{blend}(v_4, v_6)}{4} $$
    -   **Direction 3 (+Y):** Average of compositing pairs along the Y-axis:
        $$ \frac{\text{blend}(v_2, v_0) + \text{blend}(v_3, v_1) + \text{blend}(v_7, v_5) + \text{blend}(v_6, v_4)}{4} $$
    -   **Direction 4 (-Z):** Average of compositing pairs along the Z-axis:
        $$ \frac{\text{blend}(v_0, v_1) + \text{blend}(v_2, v_3) + \text{blend}(v_4, v_5) + \text{blend}(v_6, v_7)}{4} $$
    -   **Direction 5 (+Z):** Average of compositing pairs along the Z-axis:
        $$ \frac{\text{blend}(v_1, v_0) + \text{blend}(v_3, v_2) + \text{blend}(v_5, v_4) + \text{blend}(v_7, v_6)}{4} $$
5.  **Store Results:** Store the resulting `vec4` for each direction into the corresponding destination texture [S2, unofficial].

**Leaf Level Initialization:**
At the leaf level (full resolution), opacity is scaled per direction based on the surface normal $\vec{n}$ and the direction vector $\vec{d}$ to account for the cross-section of thin surfaces [S3, unofficial]:
$$ \alpha_{dir} = \alpha \cdot \max(0, \text{dot}(\vec{n}, \vec{d})) $$
This ensures that a thin surface viewed edge-on or from behind has near-zero opacity in that direction [S3, unofficial].

**Runtime Sampling:**
During cone tracing, the voxel value is retrieved by linearly interpolating the values from the three closest directions with respect to the trace/view direction $\vec{v}$ [S4][S5, unofficial]. The weights are determined by the squared dot product of the trace direction and the axis direction [S3, unofficial]:
$$ w_i = \text{dot}(\vec{v}, \vec{a}_i)^2 $$
where $\vec{a}_i$ are the three axes most aligned with $\vec{v}$. The final value is the weighted average of the three sampled directional textures [S3, unofficial].

**Parameters:**
-   **anisotropic_strength:** A float parameter (default 0.0) controlling the blend between isotropic and anisotropic results during baking. If 0.0, the behavior is identical to isotropic [S3, unofficial].
-   **Resolution:** The size of the voxel grid [S2, unofficial].
-   **Mip Level:** The current mipmap level being processed [S2, unofficial].

**Edge Cases and Pitfalls:**
-   **Descriptor Limits:** Declaring 6 sampled-image array bindings in the shader may exceed descriptor limits on lower-end/mobile GPUs [S3, unofficial]. A potential fix is making bindings conditional on a specialization constant [S3, unofficial].
-   **Base Level Sharing:** The base level (leaf) is identical for all directions. Storing it 6 times is redundant; it should be shared to reduce memory cost [S5, unofficial].
-   **Leaf Nodes:** Directional values are only computed and used for voxels not located in the last octree level (not at full resolution). Leaf nodes store only the standard single value [S4].

## Operating characteristics
-   **Time Complexity:** Mipmap generation requires processing each octree cell 6 times (once per direction), increasing bake time compared to isotropic generation. The exact factor depends on the number of non-leaf nodes.
-   **Memory Complexity:** Storing directional values for all properties increases memory consumption by 1.5x compared to isotropic [S4][S5, unofficial]. This is because the base level is shared, and only 6 directions $\times$ mip levels 1..N are stored, which is smaller than one full level-0-sized copy [S3, unofficial]. Some sources cite a total storage overhead of 1.75x [S1].
-   **Performance:** The technique delivers interactive to real-time framerates on complex scenes (e.g., Sponza scene, 280K triangles, 9-level octree, 512$^3$ virtual resolution) on hardware like NVIDIA GTX 480 [S4].
-   **Strengths:** Prevents light leaking through thin geometry (walls, floors) and reduces color mixing of opaque voxels [S1, S4].
-   **Limitations:** Higher memory cost and bake time compared to isotropic voxel cone tracing [S5, unofficial]. Potential descriptor limit issues on mobile GPUs [S3, unofficial].

## Background
The technique is based on the method described by Crassin and Green in "Octree-Based Sparse Voxelization" [S3, unofficial]. It was introduced to address quality problems in voxel cone tracing, specifically the "two red-green wall problem" where averaging values in the octree mixes colors of opaque voxels as if they were semi-transparent, causing light leaking [S1, S4]. The implementation details provided in the notes are largely derived from a GitHub repository [S2, unofficial] and a pull request to the Godot engine [S3, unofficial], which cite the original academic work [S1, S4].

## Sources
[S1] Interactive Indirect Illumination Using Voxel Cone Tracing, https://doi.org/10.1111/j.1467-8659.2011.02063.x, fetched 2026-10-26
[S2] latencyhiding/voxel_cone_tracing, https://github.com/latencyhiding/voxel_cone_tracing, fetched 2026-10-26 (unofficial)
[S3] Add anisotropic voxel mipmaps and separate diffuse/reflection bias to VoxelGI, https://github.com/NeoCry/godot_landscape/pull/12, fetched 2026-10-26 (unofficial)
[S4] PDF Interactive Indirect Illumination Using Voxel Cone Tracing, https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf, fetched 2026-10-07
[S5] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-07 (unofficial)
