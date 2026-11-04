# Voxel Cone Tracing Anisotropic Filtering

Also called: Anisotropic Voxel Mipmaps, Anisotropic Voxels, Anisotropic Voxels for Improved Cone-Tracing, Directional Voxel Mipmaps, Octree-Based Sparse Voxelization anisotropic filtering, Two Red-Green Wall Problem Mitigation

## Overview
Voxel Cone Tracing Anisotropic Filtering addresses the "two red-green wall problem" in voxel cone tracing, where isotropic averaging of opaque voxels with different colors or opacities in upper octree levels causes light leaking by mixing materials as if they were semi-transparent [S1, S4]. It stores six directional copies of color and alpha (along ±X, ±Y, ±Z axes) instead of a single isotropic average, allowing the cone tracer to detect directional occlusion (e.g., thin walls) and preventing light leaks through thin geometry [S3, unofficial][S4]. Unlike its siblings, it explicitly preserves directional visibility information during mipmap generation to mitigate the artifacts inherent in standard isotropic pre-integration [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Isotropic voxel mipmap chain (source textures), octree resolution, current mipmap level.
*   **Output:** Six anisotropic voxel mipmap chains (destination textures), one for each major direction (+X, -X, +Y, -Y, +Z, -Z) [S2, S5, unofficial].

**Data Structures:**
*   **Voxel Offsets:** An array of 8 `ivec3` values defining the corners of a 2x2x2 block relative to the block origin: `{ (1,1,1), (1,1,0), (1,0,1), (1,0,0), (0,1,1), (0,1,0), (0,0,1), (0,0,0) }` [S2, unofficial].
*   **Storage:** Six `image3D` textures (RGBA8) for the output anisotropic chains [S2, unofficial]. At runtime, six `sampler3D` bindings are added to the cone tracer shader [S3, unofficial].
*   **Parameters:** `anisotropic_strength` (float, default 0.0) controls the blend between isotropic and anisotropic results during baking; the run-time tracer only samples anisotropic textures if this value > 0 [S3, unofficial].

**Algorithm Steps:**

1.  **Initialization (Leaf Level):** For leaf nodes (full resolution), compute directional values by compositing the 8 children (or the leaf's own data if it represents a surface) front-to-back along each of the 6 axes. At the leaf level, scale the opacity per direction by the dot product of the surface normal and the direction vector: $opacity_{dir} = opacity \times \max(0, \text{dot}(\text{normal}, \text{dir}))$. This accounts for the fact that radiance is view-independent, but a thin surface has near-zero cross-section when viewed edge-on or from behind [S3, unofficial].

2.  **Mipmap Generation (Internal Levels):** For each octree cell at a given mipmap level, generate 6 directional values. This is done by performing a step of volumetric integration in depth for each direction. Specifically, for a 2x2x2 block of child voxels, average the 4 directional values to get the resulting value for one given direction [S1, S4]. The averaging is performed by compositing the 8 child voxels front-to-back along the specific axis [S3, unofficial].

3.  **Directional Compositing:** For each of the 6 directions, fetch the 8 child texels and composite them. The order of compositing depends on the direction:
    *   **-X:** Composite from +X to -X.
    *   **+X:** Composite from -X to +X.
    *   **-Y:** Composite from +Y to -Y.
    *   **+Y:** Composite from -Y to +Y.
    *   **-Z:** Composite from +Z to -Z.
    *   **+Z:** Composite from -Z to +Z.

4.  **Averaging:** After compositing the 8 children into 4 pairs (or directly averaging the result of the integration step), average the results to produce the final directional value for the parent voxel [S1, S4].

5.  **Runtime Sampling:** During cone tracing, retrieve the voxel value by linearly interpolating the values from the three closest directions with respect to the view direction. The 3 axis-matching directional textures are blended weighted by the squared dot product of the trace direction and the axis direction: $weight = \text{dot}(\text{direction}, \text{axis})^2$ [S3, unofficial][S4].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader for Mipmap Generation
# Local size: 8x8x8
# Uniforms: src_tex3D[6] (sampler3D), dest_tex3D[6] (image3D), resolution (int), mip (int)

# Voxel offsets for 2x2x2 block
voxel_offsets = [
    (1,1,1), (1,1,0), (1,0,1), (1,0,0),
    (0,1,1), (0,1,0), (0,0,1), (0,0,0)
]

def alpha_blend(v0, v1):
    # Standard alpha compositing: v0 is behind v1
    return v0 + ((1.0 - v0.a) * v1)

def fetch_texels(dir_idx, block_pos, mip_level):
    values = [vec4(0.0)] * 8
    for i in range(8):
        values[i] = texelFetch(src_tex3D[dir_idx], block_pos + voxel_offsets[i], mip_level)
    return values

# Direction compositing orders (front-to-back)
# Index 0: -X, 1: +X, 2: -Y, 3: +Y, 4: -Z, 5: +Z
# Offsets are ordered such that for -X, index 0 is at x=1 (far) and index 3 is at x=0 (near)
# The specific order in the source [S2, unofficial] implies:
# Dir 0 (-X): values[0] (1,1,1) ... values[3] (1,0,0) are "back", values[4] (0,1,1) ... values[7] (0,0,0) are "front"
# The formula in [S2, unofficial] shows: alpha_blend(values[0], values[4]) ...
# This implies values[0] is behind values[4].
# Let's define the compositing pairs explicitly based on [S2, unofficial] formulas.

def compute_aniso_mip(write_pos, block_pos, mip_level):
    # Fetch all 8 children from the isotropic source (or previous aniso level if iterative, but usually from isotropic base)
    # Note: [S2, unofficial] fetches from src_tex3D[dir], which implies the source is the isotropic chain or the previous aniso chain.
    # Standard implementation [S3, unofficial] composites from the 8 children of the octree node.
    # We assume src_tex3D contains the isotropic mipmap chain for the children.
    
    # Fetch children from the isotropic source texture (assuming src_tex3D[0] is the isotropic source for all directions for simplicity in fetch, 
    # or we fetch from the specific direction's source if building iteratively. 
    # [S2, unofficial] uses src_tex3D[dir] for fetch, which is slightly ambiguous if src_tex3D are the aniso outputs. 
    # However, [S3, unofficial] says "compositing its 8 children", implying the children's data. 
    # Let's assume we fetch the 8 children from the isotropic source texture (binding 0) for all directions.
    
    children = fetch_texels(0, block_pos, mip_level) # Fetch from isotropic source
    
    # Direction 0: -X
    # Formula [S2, unofficial]: (alpha_blend(values[0], values[4]) + alpha_blend(values[1], values[5]) + alpha_blend(values[2], values[6]) + alpha_blend(values[3], values[7])) / 4
    # This groups (0,4), (1,5), (2,6), (3,7).
    # 0:(1,1,1), 4:(0,1,1) -> X goes 1->0. Correct for -X.
    res_x_neg = (alpha_blend(children[0], children[4]) + 
                 alpha_blend(children[1], children[5]) + 
                 alpha_blend(children[2], children[6]) + 
                 alpha_blend(children[3], children[7])) / 4.0
    imageStore(dest_tex3D[0], write_pos, res_x_neg)

    # Direction 1: +X
    # Formula [S2, unofficial]: (alpha_blend(values[4], values[0]) + alpha_blend(values[5], values[1]) + alpha_blend(values[6], values[2]) + alpha_blend(values[7], values[3])) / 4
    # 4:(0,1,1), 0:(1,1,1) -> X goes 0->1. Correct for +X.
    res_x_pos = (alpha_blend(children[4], children[0]) + 
                 alpha_blend(children[5], children[1]) + 
                 alpha_blend(children[6], children[2]) + 
                 alpha_blend(children[7], children[3])) / 4.0
    imageStore(dest_tex3D[1], write_pos, res_x_pos)

    # Direction 2: -Y
    # Formula [S2, unofficial]: (alpha_blend(values[0], values[2]) + alpha_blend(values[1], values[3]) + alpha_blend(values[5], values[7]) + alpha_blend(values[4], values[6])) / 4
    # 0:(1,1,1), 2:(1,0,1) -> Y goes 1->0. Correct for -Y.
    res_y_neg = (alpha_blend(children[0], children[2]) + 
                 alpha_blend(children[1], children[3]) + 
                 alpha_blend(children[5], children[7]) + 
                 alpha_blend(children[4], children[6])) / 4.0
    imageStore(dest_tex3D[2], write_pos, res_y_neg)

    # Direction 3: +Y
    # Formula [S2, unofficial]: (alpha_blend(values[2], values[0]) + alpha_blend(values[3], values[1]) + alpha_blend(values[7], values[5]) + alpha_blend(values[6], values[4])) / 4
    # 2:(1,0,1), 0:(1,1,1) -> Y goes 0->1. Correct for +Y.
    res_y_pos = (alpha_blend(children[2], children[0]) + 
                 alpha_blend(children[3], children[1]) + 
                 alpha_blend(children[7], children[5]) + 
                 alpha_blend(children[6], children[4])) / 4.0
    imageStore(dest_tex3D[3], write_pos, res_y_pos)

    # Direction 4: -Z
    # Formula [S2, unofficial]: (alpha_blend(values[0], values[1]) + alpha_blend(values[2], values[3]) + alpha_blend(values[4], values[5]) + alpha_blend(values[6], values[7])) / 4
    # 0:(1,1,1), 1:(1,1,0) -> Z goes 1->0. Correct for -Z.
    res_z_neg = (alpha_blend(children[0], children[1]) + 
                 alpha_blend(children[2], children[3]) + 
                 alpha_blend(children[4], children[5]) + 
                 alpha_blend(children[6], children[7])) / 4.0
    imageStore(dest_tex3D[4], write_pos, res_z_neg)

    # Direction 5: +Z
    # Formula [S2, unofficial]: (alpha_blend(values[1], values[0]) + alpha_blend(values[3], values[2]) + alpha_blend(values[5], values[4]) + alpha_blend(values[7], values[6])) / 4
    # 1:(1,1,0), 0:(1,1,1) -> Z goes 0->1. Correct for +Z.
    res_z_pos = (alpha_blend(children[1], children[0]) + 
                 alpha_blend(children[3], children[2]) + 
                 alpha_blend(children[5], children[4]) + 
                 alpha_blend(children[7], children[6])) / 4.0
    imageStore(dest_tex3D[5], write_pos, res_z_pos)

# Main dispatch
if gl_GlobalInvocationID.x < resolution and gl_GlobalInvocationID.y < resolution and gl_GlobalInvocationID.z < resolution:
    write_pos = ivec3(gl_GlobalInvocationID)
    block_pos = write_pos * 2
    compute_aniso_mip(write_pos, block_pos, mip)
```

**Edge Cases and Pitfalls:**
*   **Opacity Under-estimation:** If a 2x2x2 block is half-filled with opaque voxels and half with fully transparent ones, the resulting averaged voxel may be half-transparent, causing light leaking for large cones [S1, S4].
*   **Descriptor Limits:** The 6 new sampled-image array bindings are always declared in the shader, which may exceed descriptor limits on lower-end/mobile GPUs [S3, unofficial].
*   **Leaf Level Normal Dependency:** The leaf-level scaling requires surface normals. If normals are not available or accurate, the directional opacity scaling may be incorrect [S3, unofficial].
*   **Memory Cost:** Storing directional values for all properties increases memory consumption by 1.5x compared to isotropic [S4]. The base level of the mipmap is identical for all directions and can be shared, reducing the cost from 6x to 1.5x [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** Mipmap generation is performed once during baking. The compute shader processes each voxel in constant time, proportional to the number of directions (6) and children (8). Runtime sampling involves interpolating between 3 directions, which is a constant-time operation per voxel sample.
*   **Memory Complexity:** Increases memory usage by approximately 1.5x compared to isotropic voxel cone tracing [S4]. This is because the base level is shared, and only levels 1..N are stored for the 6 directions [S3, S5, unofficial].
*   **Performance:** Delivers interactive to real-time framerates on complex scenes (e.g., Sponza, 280K triangles, 512^3 virtual resolution) on hardware like NVIDIA GTX 480 [S4].
*   **Strengths:** Significantly reduces light leaking through thin geometry (walls, floors) by preserving directional occlusion information [S3, unofficial].
*   **Limitations:** Higher memory cost and potential descriptor limit issues on mobile GPUs [S3, unofficial]. The improvement in visual quality may be marginal in some scenes compared to the cost [S5, unofficial].

## Background
The technique is based on the method described by Crassin and Green in "Octree-Based Sparse Voxelization" [S3, unofficial]. It was introduced to address the limitations of isotropic voxel cone tracing, specifically the "two red-green wall problem" [S1, S4]. The implementation details provided in the notes are derived from an unofficial GitHub repository [S2, unofficial] and a pull request to the Godot engine's VoxelGI implementation [S3, unofficial], as well as the original paper [S1, S4] and a student report [S5, unofficial].

## Sources
[S1] Interactive Indirect Illumination Using Voxel Cone Tracing, https://doi.org/10.1111/j.1467-8659.2011.02063.x, fetched 2026-10-26
[S2] latencyhiding/voxel_cone_tracing, https://github.com/latencyhiding/voxel_cone_tracing, fetched 2026-10-26 (unofficial)
[S3] Add anisotropic voxel mipmaps and separate diffuse/reflection bias to VoxelGI, https://github.com/NeoCry/godot_landscape/pull/12, fetched 2026-10-26 (unofficial)
[S4] PDF Interactive Indirect Illumination Using Voxel Cone Tracing, https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf, fetched 2026-10-27
[S5] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
