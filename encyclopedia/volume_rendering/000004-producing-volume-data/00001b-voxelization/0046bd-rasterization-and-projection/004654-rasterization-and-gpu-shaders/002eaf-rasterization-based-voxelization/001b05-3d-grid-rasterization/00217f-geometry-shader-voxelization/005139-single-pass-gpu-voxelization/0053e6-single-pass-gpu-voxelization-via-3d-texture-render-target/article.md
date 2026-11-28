# Single-Pass GPU Voxelization via 3D Texture Render Target

Also called: Geometry Shader Voxelization via 3D Texture, Real-time GPU-Based Voxelization, Voxelization via 3D Texture Render Target, slicing-based voxelization using geometry shader

## Overview
This method transforms a surface-represented scene into a high-resolution voxel grid in a single GPU pass by using the geometry shader to duplicate geometry for intersecting slices and a 3D volume texture as the direct render target [S2, unofficial]. It solves the inefficiency of multi-pass slicing algorithms and the projection artifacts or fixed-resolution limitations of grid encoding methods [S2, unofficial]. Unlike encoding methods that induce errors, this approach uses a clipping plane algorithm to avoid projection artifacts [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A mesh defined by triangles (or quadrilaterals) with vertex positions, colors, and normals.
*   **Output:** A 3D volume texture where each texel stores volumetric existence, color, surface normal, transmission coefficient, and refraction index [S1, S2, unofficial].

**Data Structures:**
*   **3D Volume Texture:** The primary storage for voxel data. The size is adjustable based on hardware limitations and application needs [S2, unofficial].
*   **Texel Layout:** Each texel stores:
    *   Existence (implicit via render target or explicit flag) [S2, unofficial].
    *   Color [S1, S2, unofficial].
    *   Surface Normal [S1, S2, unofficial].
    *   Transmission Coefficient [S1, unofficial].
    *   Refraction Index [S1, unofficial].
    *   Absorption Coefficient: Computed according to the number of surface drawings (rasterizations) in each voxel during voxelization [S1, unofficial].

**Parameters:**
*   **Voxel Resolution:** Determined by the dimensions of the 3D texture [S2, unofficial].
*   **Slice Thickness:** Used to determine which slices intersect a triangle [S2, unofficial].
*   **Absorption Coefficient:** Derived from the count of surface intersections per voxel [S1, unofficial].

**Steps:**
1.  **Vertex Shader:** Compute the positions of triangle vertices in camera coordinates [S2, unofficial].
2.  **Geometry Shader:**
    *   Determine which slices along the z-axis intersect the triangle [S2, unofficial].
    *   Duplicate the triangle for each intersecting slice [S2, unofficial].
    *   Rasterize the duplicated triangles in the pixel shader [S2, unofficial].
3.  **Pixel Shader:**
    *   Rasterize the triangle onto the 3D texture render target [S2, unofficial].
    *   Store surfacial data (normal, color) and volumetric data (transmittance) in the corresponding texels [S2, unofficial].
    *   Update the absorption coefficient based on the number of surface drawings in each voxel [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
def vertex_shader(position, normal, color):
    # Compute positions in camera coordinates
    camera_pos = transform_to_camera(position)
    return camera_pos, normal, color

# Geometry Shader
def geometry_shader(triangle_vertices):
    # triangle_vertices: list of 3 camera-space positions
    z0 = triangle_vertices[0].z
    z1 = triangle_vertices[1].z
    z2 = triangle_vertices[2].z
    
    # Determine slice range
    # Note: 'thickness' is a parameter representing slice depth
    max_slice = max(z0, z1, z2) / thickness
    min_slice = min(z0, z1, z2) / thickness
    
    # Duplicate triangle for each intersecting slice
    for slice_z in range(int(min_slice), int(max_slice) + 1):
        # Adjust triangle vertices to lie on the slice plane or pass through
        # The geometry shader handles assembly and construction
        # Duplicate the triangle for rasterization in the pixel shader
        yield triangle_vertices, slice_z

# Pixel Shader
def pixel_shader(triangle_vertices, slice_z, normal, color):
    # Rasterize triangle onto 3D texture at slice_z
    # Store color, normal, transmission coefficient, refraction index
    # Update absorption coefficient based on number of surface drawings
    write_to_3d_texture(slice_z, triangle_vertices, color, normal)
```

**Edge Cases and Pitfalls:**
*   **Deformable Models:** This method handles deformable models without preprocessing or sorting, overcoming limitations of previous encoding methods [S2, unofficial].
*   **Primitive Types:** While boundary models use triangles, the algorithm can be extended to handle quadrilaterals without major modification [S2, unofficial].
*   **Hardware Limitations:** The size of the 3D texture is adjustable but constrained by hardware limitations [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Single GPU pass, making it efficient for real-time applications [S2, unofficial].
*   **Memory Complexity:** Depends on the size of the 3D texture, which is adjustable [S2, unofficial].
*   **Performance:** Capable of real-time voxelization of highly complex scenes [S2, unofficial].
*   **Strengths:** Avoids projection artifacts, handles deformable models without preprocessing, allows adjustable resolution, and stores rich surfacial and volumetric data [S2, unofficial].
*   **Limitations:** Limited by hardware constraints on 3D texture size [S2, unofficial].
*   **When to Use:** Real-time applications requiring transparent shadow, transmittance, and refraction effects [S2, unofficial].
*   **When Not to Use:** When hardware does not support 3D texture render targets or geometry shaders [S2, unofficial].

## Background
This method was proposed by Hsu-Huai Chang and Yu-Chi Lai in their paper "Real-time GPU-Based Voxelization and Applications" [S2, unofficial]. It leverages newly available GPU functionalities, specifically the geometry shader and 3D volume texture as render targets, to overcome limitations of previous slicing-based and encoding-based voxelization methods [S2, unofficial]. The technique is used for rendering transparent shadow, transmittance, and refraction effects in real-time applications [S2, unofficial].

## Sources
[S1] Real-time Realistic Voxel-based Rendering, https://doi.org/10.2312/pe.pg.pg2011short.001-006, fetched 2026-10-27 (unofficial)
[S2] Real-time GPU-Based Voxelization and Applications, https://minds.wisc.edu/items/e8fdcf0a-3edc-4603-82ef-f91f70900877, fetched 2026-10-29 (unofficial)
