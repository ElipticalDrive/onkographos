# Instancing with Procedural/Shader Generation

Also called: Instanced Voxel Rendering with Palette Textures, Palette-based Voxel Rendering

## Overview
Instancing with Procedural/Shader Generation renders a voxel model by submitting a single unit cube mesh as an instanced primitive, where each instance represents a voxel [S1, unofficial]. It solves the problem of efficiently rendering large numbers of voxels by decoupling the geometric representation (one cube) from the spatial and material data, which are provided via per-instance attributes (offsets) and procedural sampling (palette textures) [S1, unofficial]. It differs from [Naive Voxel Rendering](1103-naive-voxel-rendering.md) by avoiding the generation of unique vertex buffers for every voxel, and from [Binary Volume Rendering (Opacity Thresholding)](1015-binary-volume-rendering-opacity-thresholding.md) by explicitly rasterizing geometry for every occupied voxel rather than discarding data below a threshold [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel grid (e.g., `.vly` format) containing occupied voxel indices and associated color/material IDs [S1, unofficial].
*   **Output:** A rasterized image of the voxel model with Phong shading [S1, unofficial].

**Data Structures:**
1.  **Instance Buffer (VBO):** A contiguous array of floats bound to `GL_ARRAY_BUFFER` with `GL_STATIC_DRAW` [S1, unofficial].
    *   **Layout:** Each instance consists of exactly 5 floats: `(offsetX, offsetY, offsetZ, u, v)` [S1, unofficial].
    *   **Size:** `instanceCount * 5 * 4` bytes [S1, unofficial].
    *   **Fields:**
        *   `offsetX`, `offsetY`, `offsetZ`: World-space or model-space translation for the voxel center [S1, unofficial].
        *   `u`, `v`: Texture coordinates for sampling the palette texture [S1, unofficial].
2.  **Palette Texture:** A 2D texture containing base colors, sampled using the instance `u, v` coordinates [S1, unofficial].
3.  **Base Geometry:** A single unit cube mesh (vertices and indices) [S1, unofficial].

**Numbered Steps:**
1.  **Data Preparation:** Iterate through the voxel grid. For each occupied voxel at grid index $(i, j, k)$:
    *   Calculate the world position. Note coordinate system mismatches: if the source format uses Z-up and the renderer uses Y-up, swap Y and Z components during offset calculation [S1, unofficial].
    *   Determine the `u, v` coordinates based on the voxel's material ID or color index.
    *   Pack `(offsetX, offsetY, offsetZ, u, v)` into the instance buffer [S1, unofficial].
2.  **Buffer Upload:** Upload the instance buffer to the GPU [S1, unofficial].
3.  **Vertex Shader Execution:**
    *   For each vertex of the base cube, retrieve the local position $P_{local}$ and the instance offset $O_{inst}$ using `gl_InstanceID` [S1, unofficial].
    *   Compute the combined position: $P_{combined} = P_{local} + O_{inst}$ [S1, unofficial].
    *   Transform to clip space: $gl\_Position = MVP \times vec4(P_{combined}, 1.0)$ [S1, unofficial].
    *   Pass the instance UVs ($u, v$) to the fragment shader [S1, unofficial].
4.  **Fragment Shader Execution:**
    *   Sample the base color from the palette texture: $C_{base} = texture(uPaletteTexture, vUV).rgb$ [S1, unofficial].
    *   Compute lighting using a Phong model [S1, unofficial]:
        *   Ambient: $C_{ambient} = C_{base} \times I_{ambient}$
        *   Diffuse: $C_{diffuse} = C_{base} \times \max(N \cdot L, 0.0)$
        *   Specular: $C_{specular} = (1.0) \times S_{intensity} \times R^{shininess}$ (where $R$ is the reflection vector) [S1, unofficial].
    *   Combine: $C_{final} = C_{ambient} + C_{diffuse} + C_{specular}$ [S1, unofficial].
5.  **Rasterization:** The GPU rasterizes the instanced cubes. Enable back-face culling to discard faces oriented away from the camera [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader Logic (GLSL-like)
uniform mat4 uMVPMatrix;
uniform sampler2D uPaletteTexture;
in vec3 aLocalPosition;
in vec3 aInstanceOffset;
in vec2 aInstanceUV;
out vec2 vUV;

void main() {
    // Combine local vertex position with instance offset
    vec3 localCombined = aLocalPosition + aInstanceOffset;
    
    // Transform to clip space
    gl_Position = uMVPMatrix * vec4(localCombined, 1.0);
    
    // Pass UVs to fragment shader
    vUV = aInstanceUV;
}

# Fragment Shader Logic (GLSL-like)
uniform vec3 uLightPosWorld;
uniform vec3 uCameraPosWorld;
uniform float uAmbientIntensity;
uniform float uSpecularIntensity;
uniform float uShininess;
in vec2 vUV;
out vec4 fragColor;

void main() {
    // Sample base color from palette texture
    vec3 baseColor = texture(uPaletteTexture, vUV).rgb;
    
    // Calculate lighting vectors (assuming normal is available via interpolation or geometry)
    // Note: For a unit cube, normals are constant per face. 
    // In a real implementation, normals must be passed from vertex shader or computed.
    // Here we assume a simplified Phong calculation as per source description.
    
    vec3 ambient = baseColor * uAmbientIntensity;
    
    // Placeholder for Normal (N) and Light (L) calculation
    // N = normalize(vNormal); 
    // L = normalize(uLightPosWorld - worldPosition);
    // diffuse = baseColor * max(dot(N, L), 0.0);
    
    // Placeholder for Specular
    // specular = vec3(1.0) * uSpecularIntensity; 
    
    // Final color (simplified based on source notes)
    fragColor = vec4(ambient, 1.0); 
}
```

**Parameters:**
*   `AMBIENT_INTENSITY`: 0.35 [S1, unofficial].
*   `SPECULAR_INTENSITY`: 0.5 [S1, unofficial].
*   `SHININESS`: 32 [S1, unofficial].
*   `LIGHT_POS_WORLD`: $(10, 15, 10)$ [S1, unofficial].
*   `MIN_ZOOM`: 0.3 [S1, unofficial].
*   `MAX_ZOOM`: 3.0 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Coordinate System Mismatch:** Source formats may use Z-up (e.g., `.vly`) while OpenGL uses Y-up. This must be handled by swapping Y and Z components during offset calculation [S1, unofficial].
*   **Palette Texture Artifacts:** Sampling at boundaries can cause bleeding. Mitigate by setting texture wrap modes to `GL_CLAMP_TO_EDGE` for both S and T axes [S1, unofficial].
*   **Precision:** Floating-point precision in instance offsets may cause z-fighting if voxels are extremely close or the scene scale is large [G].

## Subtopics

### Sphere Impostors for DEM Pebbles
Renders Discrete Element Method (DEM) pebble beds using instanced sphere impostors [S2, unofficial]. Each pebble is drawn as a quad with an exact orthographic ray-sphere hit test in the fragment shader, providing sphere shading and depth [S2, unofficial]. It differs from standard voxel cubes by using a 2D primitive with a 3D intersection test in the fragment shader, allowing for smooth spherical appearance without the geometry overhead of actual spheres [S2, unofficial]. It is chosen for visualizing large numbers of discrete particles (e.g., 27,000 pebbles) at interactive rates with low CPU overhead (1.2-1.3 ms per frame) [S2, unofficial].

### Splat quad vertex layout
Sets up a 4-vertex quad with specific coordinates $(-2,-2), (2,-2), (2,2), (-2,2)$ and indices $0,1,2,0,2,3$ for shader-based scaling of projected 2D Gaussian eigenvectors [S3, unofficial]. The vertex buffer uses `VET_FLOAT2` for position, and the instance buffer uses `VET_FLOAT1` for texture coordinates [S3, unofficial]. This layout is used in 3D Gaussian Splatting via instancing to efficiently project ellipses onto the screen [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Setup time is $O(V)$ to compute offsets and pack data, where $V$ is the number of voxels [S1, unofficial]. Draw time is $O(1)$ draw call with instancing, but fragment shader cost is proportional to the number of visible voxels [S1, unofficial].
*   **Memory Complexity:** $O(V)$ for the instance buffer (5 floats per voxel) and $O(P)$ for the palette texture [S1, unofficial].
*   **Performance:** Implemented in the VoxelRenderer Android application for rendering `.vly` models using OpenGL ES 3.0 [S1, unofficial].
*   **Strengths:** Low CPU overhead for draw calls; efficient memory usage for large voxel counts; flexible material assignment via palette textures [S1, unofficial].
*   **Limitations:** Requires a palette texture; back-face culling is necessary for performance; coordinate system conversions must be handled manually [S1, unofficial].
*   **When to use:** When rendering large, static voxel models with simple shading requirements and limited GPU memory bandwidth [S1, unofficial].
*   **When not to use:** When dynamic, per-voxel geometry changes are frequent (requires buffer updates) or when complex, non-uniform lighting per voxel is needed beyond simple Phong [S1, unofficial].

## Background
This method is implemented in the VoxelRenderer Android application for rendering `.vly` voxel models using OpenGL ES 3.0 [S1, unofficial]. It builds on the principles of [Voxels rasterized as geometry](27-voxels-rasterized-as-geometry.md) by using instancing to reduce draw call overhead [S1, unofficial].

## Sources
[S1] Il-castor/VoxelRenderer, https://github.com/Il-castor/VoxelRenderer, fetched 2026-10-26 (unofficial)
[S2] dhoby-ghaut: GPU rendering for the 3D and 2D views (sphere impostors for pebbles; GPU or accelerated CSG ray tracing), https://github.com/theodoreOnzGit/outram-park-backend/issues/587, fetched 2026-10-26 (unofficial)
[S3] splat-ros/RVizSplat, https://github.com/splat-ros/RVizSplat, fetched 2026-10-27 (unofficial)
