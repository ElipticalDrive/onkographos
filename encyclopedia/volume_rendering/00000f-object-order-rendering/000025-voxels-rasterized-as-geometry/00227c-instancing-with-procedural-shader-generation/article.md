# Instancing with Procedural/Shader Generation

Also called: Instanced Voxel Rendering with Palette Textures, Palette-based Voxel Rendering

## Overview
This method renders a voxel model by instancing a single unit cube mesh, where each instance represents a voxel positioned via per-instance 3D offsets and colored via per-instance UV coordinates sampled from a 2D palette texture [S1, unofficial]. It solves the problem of efficiently rendering large voxel datasets by reducing draw calls to a single invocation while offloading color and position data to GPU buffers [S1, unofficial]. It differs from [Voxel Instancing](9449-voxel-instancing.md) by using a flat, procedural data layout for simple voxel blocks rather than complex, distinct object classes, and from [Instanced Voxel Rendering with Palette Textures](2794-instancing-with-procedural-shader-generation.md) (its sibling variant) by relying on explicit per-voxel UV attributes rather than procedural noise modulation for surface variation.

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel grid defining occupied positions; a 2D palette texture containing base colors; a unit cube mesh (6 faces, 24 vertices for flat shading or 8 for shared, though the implementation uses explicit face definitions).
*   **Output:** A rasterized image of the voxel model with Phong lighting.

**Data Structures:**
1.  **Instance Buffer (VBO):** A contiguous array of floats bound to `GL_ARRAY_BUFFER` with `GL_STATIC_DRAW` usage hint [S1, unofficial].
    *   **Layout:** Exactly 5 floats per instance in the order: `offsetX` (float), `offsetY` (float), `offsetZ` (float), `u` (float), `v` (float) [S1, unofficial].
    *   **Size:** `instanceCount * 5 * 4` bytes [S1, unofficial].
2.  **Palette Texture:** A 2D texture (`GL_TEXTURE_2D`) storing the color palette [S1, unofficial].
3.  **Cube Geometry:** A static VBO/EBO defining the unit cube vertices and indices [S1, unofficial].

**Numbered Steps:**
1.  **Data Preparation:** Iterate through the voxel grid. For each occupied voxel at grid coordinates $(i, j, k)$, calculate the world offset. Handle coordinate system mismatches (e.g., Z-up model to Y-up OpenGL) by swapping axes: `localY = gridZ`, `localZ = gridY` [S1, unofficial].
2.  **Buffer Population:** Pack the offset $(x, y, z)$ and the palette UV coordinates $(u, v)$ into the contiguous instance array [S1, unofficial].
3.  **Upload:** Upload the instance array to the GPU via `glBufferData` [S1, unofficial].
4.  **Shader Setup:** Bind the cube VBO/EBO and the instance VBO. Set up vertex attributes:
    *   Attribute 0: Cube vertex positions (local space).
    *   Attribute 1: Instance offsets (3 floats).
    *   Attribute 2: Instance UVs (2 floats) [S1, unofficial].
5.  **Vertex Shader Execution:**
    *   Combine local vertex position with instance offset: $P_{combined} = P_{local} + P_{instance\_offset}$ [S1, unofficial].
    *   Transform to clip space: $gl\_Position = MVP \times vec4(P_{combined}, 1.0)$ [S1, unofficial].
    *   Pass UVs to fragment shader [S1, unofficial].
6.  **Fragment Shader Execution:**
    *   Sample base color: $C_{base} = texture(uPaletteTexture, vUV).rgb$ [S1, unofficial].
    *   Compute Phong lighting:
        *   Ambient: $C_{ambient} = C_{base} \times I_{ambient}$
        *   Diffuse: $C_{diffuse} = C_{base} \times \max(\mathbf{N} \cdot \mathbf{L}, 0.0)$
        *   Specular: $C_{specular} = \mathbf{1.0} \times \rho \times I_{specular}$
    *   Final color: $C_{final} = C_{ambient} + C_{diffuse} + C_{specular}$ [S1, unofficial].
7.  **Rendering:** Issue a single `glDrawElementsInstanced` call [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader Logic (GLSL-like)
# Inputs:
#   aLocalPosition: vec3 (from cube VBO)
#   aInstanceOffset: vec3 (from instance VBO, attribute 1)
#   aInstanceUV: vec2 (from instance VBO, attribute 2)
# Uniforms:
#   uMVPMatrix: mat4

void main() {
    // Combine local position with instance offset
    vec3 localCombined = aLocalPosition + aInstanceOffset;
    
    // Transform to clip space
    gl_Position = uMVPMatrix * vec4(localCombined, 1.0);
    
    // Pass UV to fragment shader
    vUV = aInstanceUV;
}

# Fragment Shader Logic (GLSL-like)
# Inputs:
#   vUV: vec2
# Uniforms:
#   uPaletteTexture: sampler2D
#   uAmbientIntensity: float (e.g., 0.35)
#   uSpecularIntensity: float (e.g., 0.5)
#   uLightPosWorld: vec3
#   uCameraPosWorld: vec3
#   uShininess: float (e.g., 32.0)

void main() {
    // Sample base color from palette
    vec3 baseColor = texture(uPaletteTexture, vUV).rgb;
    
    // Calculate lighting vectors
    vec3 normal = normalize(gl_FrontFacing ? vec3(0,0,1) : vec3(0,0,-1)); // Simplified for unit cube faces
    // Note: In a real implementation, normals are derived from geometry or passed explicitly.
    // The source uses a simple Phong model.
    
    vec3 lightDir = normalize(uLightPosWorld - gl_FragCoord.xyz); // Approximation
    vec3 viewDir = normalize(uCameraPosWorld - gl_FragCoord.xyz);
    vec3 reflectDir = reflect(-lightDir, normal);
    
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), uShininess);
    
    vec3 ambient = baseColor * uAmbientIntensity;
    vec3 diffuse = baseColor * max(dot(normal, lightDir), 0.0);
    vec3 specular = vec3(1.0) * spec * uSpecularIntensity;
    
    gl_FragColor = vec4(ambient + diffuse + specular, 1.0);
}
```

**Parameters:**
*   `AMBIENT_INTENSITY`: 0.35 [S1, unofficial]
*   `SPECULAR_INTENSITY`: 0.5 [S1, unofficial]
*   `SHININESS`: 32.0 [S1, unofficial]
*   `LIGHT_POS_WORLD`: (10.0, 15.0, 10.0) [S1, unofficial]
*   `MIN_ZOOM`: 0.3 [S1, unofficial]
*   `MAX_ZOOM`: 3.0 [S1, unofficial]

**Edge Cases and Pitfalls:**
*   **Coordinate System Mismatch:** Model formats may use Z-up while OpenGL uses Y-up. This is handled by swapping Y and Z during offset calculation [S1, unofficial].
*   **Palette Texture Artifacts:** Sampling at boundaries can cause bleeding. Mitigated by setting texture wrap modes to `GL_CLAMP_TO_EDGE` for both S and T axes [S1, unofficial].

## Subtopics

### Splat quad vertex layout
This variant adapts the instancing principle for 3D Gaussian Splatting. It sets up a 4-vertex quad with specific coordinates (-2,-2), (2,-2), (2,2), (-2,2) and indices 0,1,2,0,2,3 to allow shader-based scaling of projected 2D Gaussian eigenvectors [S2, unofficial]. The vertex buffer uses `VET_FLOAT2` for positions, and the index buffer uses `IT_16BIT` [S2, unofficial]. The instance buffer contains 1 float per instance (`VET_FLOAT1`) representing texture coordinates or scale parameters [S2, unofficial]. It differs from standard voxel cubes by using a single quad per splat rather than a cube mesh, avoiding the memory overhead of tile-based rasterization [S1, unofficial].

### Instanced Rendering with Texture Arrays
This strategy combines instancing with OpenGL texture arrays to render many cubes sharing the same geometry but different sprites from a single texture atlas [S3, unofficial]. It retains standard texture coordinates (0.0, 1.0, 1.0, 0.0) for all faces and specifies the array slice as a per-instance vertex attribute [S3, unofficial]. This avoids adjacent texture bleeding and filtering issues common with sprite sheets [S3, unofficial]. It differs from naive instancing by allowing distinct textures per instance without separate draw calls, whereas sending the array slice via `glVertexAttrib1f` would require a separate draw call per cube [S3, unofficial].

### Procedural Object Instancing
This variant generates instance positions based on procedural rules via a built-in generator, rather than manual painting [S4]. It is accessed via the `generator` property of `VoxelInstanceLibraryItem` [S4]. It differs from manual instancing by automating spawn locations based on terrain conditions [S4]. Pitfalls include the lack of manual editor editing support and the immaturity of the custom instance generator API, which may change and is not available to scripts [S4].

## Operating characteristics
*   **Time Complexity:** Setup time is $O(V)$ to compute offsets and pack data, where $V$ is the number of voxels [S1, unofficial]. Draw time is $O(1)$ draw call, with fragment shader cost proportional to visible voxels [S1, unofficial].
*   **Memory Complexity:** $O(V)$ for the instance buffer (5 floats per voxel) and $O(P)$ for the palette texture [S1, unofficial].
*   **Performance:** Implemented in the VoxelRenderer Android application for rendering .vly voxel models using OpenGL ES 3.0 [S1, unofficial].
*   **Strengths:** Single draw call reduces CPU overhead; palette textures reduce memory bandwidth for color data compared to per-vertex colors.
*   **Limitations:** Limited to simple voxel blocks; complex lighting or shading requires more complex shaders; coordinate system conversions must be handled manually.

## Background
This method is implemented in the VoxelRenderer Android application for rendering .vly voxel models using OpenGL ES 3.0 [S1, unofficial]. It leverages standard OpenGL instancing features to optimize the rendering of large numbers of identical geometric primitives (cubes) with varying positions and colors.

## Sources
[S1] Il-castor/VoxelRenderer, https://github.com/Il-castor/VoxelRenderer, fetched 2026-10-26 (unofficial)
[S2] splat-ros/RVizSplat, https://github.com/splat-ros/RVizSplat, fetched 2026-10-27 (unofficial)
[S3] Best strategy on VAO and texture coordinates for voxel rendering?, https://gamedev.stackexchange.com/questions/128440/best-strategy-on-vao-and-texture-coordinates-for-voxel-rendering, fetched 2026-10-26 (unofficial)
[S4] Instancing - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/instancing/, fetched 2026-10-27
