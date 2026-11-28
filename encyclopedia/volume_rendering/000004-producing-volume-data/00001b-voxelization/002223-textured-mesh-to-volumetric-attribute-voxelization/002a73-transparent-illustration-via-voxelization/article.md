# Transparent Illustration via Voxelization

Also called: Semitransparent voxel rendering, Transparent Illustration via Multivalued Voxelization

## Overview
Transparent Illustration via Voxelization is a technique for rendering complex, deformable geometric models semi-transparently within volumetric scenes [S1, unofficial][S2, unofficial]. It solves the visualization problem by storing multivalued data (normals, texture coordinates, material IDs) in a voxel worksheet and integrating this data into a slicing-based volume rendering pipeline [S1, unofficial][S2, unofficial]. The method builds on [Textured Mesh to Volumetric Attribute Voxelization](2760-textured-mesh-to-volumetric-attribute-voxelization.md) by extending volumetric attribute formation to support multivalued storage for transparent illustration.

## How it works
**Inputs:** A complex deformable geometric model, texture maps, a voxelization engine, and a slicing-based volume rendering system [S1, unofficial][S2, unofficial].
**Outputs:** A semi-transparent visual representation of the model integrated into a volumetric scene [S1, unofficial][S2, unofficial].
**Data Structures:** The core data structure is the "worksheet" (voxel grid) [S1, unofficial][S2, unofficial]. Each occupied voxel stores a combination of vector quantized normals, texture coordinates, and material identifications [S1, unofficial][S2, unofficial].
**Steps:**
1. **Multivalued Voxelization:** Store combinations of vector quantized normals, texture coordinates, and material identifications in the worksheet [S1, unofficial][S2, unofficial].
2. **Integration:** Integrate the voxelization engine into a slicing-based volume rendering pipeline [S1, unofficial][S2, unofficial].
3. **Lighting:** Enable lighting in the rendering stage to show the appearance of the model [S1, unofficial][S2, unofficial].
4. **Rendering:** Render volume data and voxelized slices orderly in 3D space to show complex deformable geometric models semitransparently [S1, unofficial][S2, unofficial].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transparent_illustration(model, texture_maps, voxel_grid, camera):
    # Step 1: Multivalued Voxelization
    for triangle in model.triangles:
        for voxel in voxel_grid.intersected(triangle):
            voxel_grid[voxel].normal = vector_quantize(triangle.normal)
            voxel_grid[voxel].tex_coord = interpolate_texture_coords(triangle, voxel)
            voxel_grid[voxel].material_id = triangle.material_id

    # Step 2: Integration with Volume Rendering
    # The sources do not specify the exact slicing algorithm implementation.
    slices = voxel_grid.extract_slices(camera.view_direction)

    # Step 3: Lighting
    # The sources do not specify the exact lighting computation.
    for slice in slices:
        for voxel in slice:
            if voxel.is_occupied:
                voxel.appearance = compute_lit_color(voxel.normal, voxel.tex_coord, voxel.material_id)

    # Step 4: Rendering
    # The sources do not specify the exact rendering call or blending mode.
    slices.sort(key=lambda s: s.depth)
    for slice in slices:
        for voxel in slice:
            if voxel.is_occupied:
                blend_into_framebuffer(voxel.appearance)
```
**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Deformable Models:** The method is specifically designed for complex deformable models, implying the voxelization must be efficient enough to handle dynamic updates [S1, unofficial][S2, unofficial].
*   **Blending Order:** Slices must be rendered orderly in 3D space to ensure correct semi-transparent blending [S1, unofficial][S2, unofficial].

## Operating characteristics
*   **Strengths:** Simplifies transparent illustration of complex deformable models; integrates seamlessly with slicing-based volume rendering [S1, unofficial][S2, unofficial].
*   **Use Cases:** Surgical planning, intra-operation navigation, and radiation therapy [S1, unofficial][S2, unofficial].

## Background
Transparent illustration has been an open problem in the graphics community [S1, unofficial][S2, unofficial]. This technique proposes that the problem becomes simple if rapid voxelization is feasible [S1, unofficial][S2, unofficial]. It is based on the multivalued voxelization algorithm [S1, unofficial][S2, unofficial]. The approach is promising for applications such as surgical planning, intra-operation navigation, and radiation therapy [S1, unofficial][S2, unofficial].

## Sources
[S1] Real-time Voxelization for Complex Models, https://wo1fsea.github.io/files/About_NavMesh/P2_Real-time_Voxelization_for_Complex_Models.pdf, fetched 2026-10-29 (unofficial)
[S2] Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
