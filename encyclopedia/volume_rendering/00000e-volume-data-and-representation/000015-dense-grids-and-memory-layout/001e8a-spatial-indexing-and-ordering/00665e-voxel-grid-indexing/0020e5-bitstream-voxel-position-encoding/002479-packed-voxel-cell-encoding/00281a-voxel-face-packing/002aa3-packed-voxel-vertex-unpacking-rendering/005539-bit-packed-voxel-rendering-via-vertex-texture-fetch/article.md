# Bit-packed Voxel Rendering via Vertex Texture Fetch

Also called: Bit-packed Voxel Rendering

## Overview
Bit-packed Voxel Rendering via Vertex Texture Fetch is a technique for rendering large voxel grids (e.g., $128^3$) by encoding voxel existence data into 2D RGBA textures and using vertex shaders to sample this data for geometry culling [S1, unofficial]. It differs from standard instanced rendering by avoiding per-voxel vertex buffers; instead, it projects a fixed set of vertices and discards those corresponding to empty space via near-plane clipping, leveraging vertex texture fetch capabilities [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid of resolution $R \times R \times R$ (typically $128 \times 128 \times 128$) [S1, unofficial].
*   **Input:** 4 2D textures, each of size $R \times R$ pixels (e.g., $128 \times 128$), formatted as RGBA8 [S1, unofficial].
*   **Input:** A vertex buffer containing a fixed set of vertices representing a single voxel box (or a simplified proxy) [S1, unofficial].
*   **Output:** Rendered voxel geometry on the screen [S1, unofficial].

**Data Structures:**
The 3D grid is flattened into 4 separate 2D textures. Each texture stores a slice of the Z-axis data.
*   **Texture Dimensions:** $128 \times 128$ pixels.
*   **Pixel Format:** RGBA, 8 bits per channel.
*   **Bit Packing:** Each pixel contains 32 bits ($4 \times 8$). Each bit corresponds to one voxel along the Z-axis for a specific $(X, Y)$ coordinate [S1, unofficial].
*   **Mapping:**
    *   Texture $T_0$ stores Z indices $0$ to $31$.
    *   Texture $T_1$ stores Z indices $32$ to $63$.
    *   Texture $T_2$ stores Z indices $64$ to $95$.
    *   Texture $T_3$ stores Z indices $96$ to $127$.
    *   Total voxels: $128 \times 128 \times 32 \times 4 = 2,097,152$ [S1, unofficial].

**Vertex Attributes:**
Two approaches are described for vertex generation [S1, unofficial]:
1.  **Explicit Geometry:** Vertex attributes for position and normal, plus a parallel attribute containing UV coordinates and bit information (effectively a 3D UV coordinate to sample the correct bit). This is memory-heavy but fast [S1, unofficial].
2.  **Procedural Geometry:** A single 3D UV-coordinate attribute and a vertex index stored in the `.w` component of a `vec4` attribute. The vertex shader calculates position and normal on the fly [S1, unofficial].

**Algorithm Steps:**
1.  **Texture Upload:** Load the 4 RGBA textures into GPU memory. Each pixel's 32 bits represent 32 voxels along the Z-axis for the corresponding $(X, Y)$ grid cell [S1, unofficial].
2.  **Vertex Setup:** Create a vertex buffer. If using the procedural approach, the buffer contains vertices for a single voxel unit, with attributes providing the $(X, Y)$ grid location and the vertex index within the box [S1, unofficial].
3.  **Shader Execution:** For each vertex in the buffer:
    a.  Determine the $(X, Y)$ grid coordinates from the vertex attributes [S1, unofficial].
    b.  Determine the Z-coordinate and the specific bit index within the 32-bit pixel [S1, unofficial].
    c.  Select the appropriate texture ($T_0$ to $T_3$) based on the Z-range [S1, unofficial].
    d.  Sample the texture at the $(X, Y)$ coordinate [S1, unofficial].
    e.  Extract the specific bit corresponding to the voxel's Z-position [S1, unofficial].
    f.  **Culling Logic:**
        *   If the bit is **set** (voxel exists): Project the vertex using the standard model-view-projection matrix [S1, unofficial].
        *   If the bit is **not set** (voxel empty): Move the vertex behind the near clipping plane (e.g., $z < -1$ in clip space) so the WebGL pipeline culls it [S1, unofficial].
4.  **Rendering:** Draw the vertices. Only vertices with set bits are visible [S1, unofficial].

**Pseudocode (Vertex Shader Logic):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for vertex shader logic
# Inputs:
#   vec3 a_uv: 3D UV coordinate representing grid position and bit index
#   vec4 a_vertex_data: Contains vertex index in .w
#   sampler2D textures[4]: The 4 bit-packed textures

void main() {
    // 1. Extract grid coordinates and bit index
    // a_uv.x = X grid coordinate (0-127)
    // a_uv.y = Y grid coordinate (0-127)
    // a_uv.z = Z grid coordinate (0-127)
    
    float x = a_uv.x;
    float y = a_uv.y;
    float z = a_uv.z;
    
    // 2. Determine which texture slice to sample
    // Each texture covers 32 Z levels
    int texture_index = int(z / 32.0);
    
    // 3. Determine the bit index within the 32-bit pixel
    int bit_index = int(z % 32.0);
    
    // 4. Sample the texture
    // Note: WebGL does not support native bit operations or ints on all drivers
    // We use floating point sampling to extract the bit
    vec4 color = texture2D(textures[texture_index], vec2(x / 128.0, y / 128.0));
    
    // 5. Extract the specific bit
    // Since we cannot use bitwise AND, we use floating point math
    // Assuming the bit is stored in the least significant bit of one channel
    // or distributed across channels. The source implies a direct bit mapping.
    // A common technique is to multiply by 2^bit_index and take the fractional part
    // or use a pre-computed lookup. However, the source notes "avoid using ints".
    // A robust way to check if the N-th bit is set in a float representation:
    // float bit_value = mod(color.r * pow(2.0, bit_index), 2.0);
    // if (bit_value >= 1.0) bit_is_set = true;
    
    // Simplified logic based on source description:
    float bit = extract_bit(color, bit_index); // Implementation detail [G]
    
    // 6. Project or Cull
    if (bit > 0.5) {
        // Voxel exists: Project normally
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    } else {
        // Voxel empty: Push behind near clipping plane
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        gl_Position.z = -2.0; // Behind near plane (assuming -1 to 1 clip space)
    }
}
```

**Parameters:**
*   **Grid Resolution:** $128 \times 128 \times 128$ voxels [S1, unofficial].
*   **Texture Size:** $128 \times 128$ pixels per texture [S1, unofficial].
*   **Number of Textures:** 4 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **WebGL Bit Operations:** WebGL lacks native bit operations (shift, and, or) and integer support on some Mac drivers. Avoid using `int` types; use floating-point sampling and manipulation instead [S1, unofficial].
*   **Vertex Texture Fetch Support:** Vertex texture fetch is required. While most computers support it, it is not universally supported in WebGL implementations [S1, unofficial].
*   **Library Choice:** Three.js may not support `drawArrays` cleanly for this approach; GLOW library is suggested as a cleaner alternative [S1, unofficial].

## Subtopics

### Packed Voxel Vertex Unpacking & Rendering
The parent method that unpacks a single 32-bit integer per vertex into spatial coordinates, material ID, face ID, ambient occlusion ID, and flip ID. This method is more general but requires more complex vertex attributes [4805-packed-voxel-vertex-unpacking-rendering.md].

### Voxel Face Packing
A method that packs voxel face data into textures for rendering. It differs by focusing on face-level detail rather than voxel existence culling [4804-voxel-face-packing.md].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per vertex for texture sampling and culling. The total rendering time depends on the number of vertices in the base mesh (e.g., 24 vertices per voxel if not instanced, or fewer if instanced/procedural).
*   **Memory Complexity:** $O(R^2)$ for texture storage, where $R$ is the grid resolution. For $128^3$, this is 4 textures of $128 \times 128 \times 4$ bytes = 256 KB total [S1, unofficial].
*   **Measured Performance:** The source does not provide specific FPS measurements, but notes that the explicit geometry method is "probably quite fast" while being memory-heavy [S1, unofficial].
*   **Strengths:** Efficient culling of empty space; low memory footprint for the data structure; simple shader logic.
*   **Limitations:** Requires vertex texture fetch support; limited by WebGL integer/bit operation support; not suitable for dynamic voxel changes without texture updates; fixed grid resolution [S1, unofficial].
*   **When to Use:** When rendering large, static or semi-static voxel grids where most voxels are empty, and vertex texture fetch is available.
*   **When Not to Use:** When dynamic voxel updates are frequent (texture updates are costly); when running on hardware without vertex texture fetch support; when using libraries that do not support the required draw calls [S1, unofficial].

## Background
The technique was proposed in a 2012 Stack Overflow answer by Mikael Emtinger for rendering animated voxel scenes in WebGL using Three.js or GLOW [S1, unofficial]. It was suggested as an alternative to loading large static files and incrementally adding/deleting voxels, leveraging the GPU's ability to sample textures in the vertex shader for culling [S1, unofficial].

## Sources
[S1] Best way to render a voxel animation using WebGL?, https://stackoverflow.com/questions/12076932/best-way-to-render-a-voxel-animation-using-webgl, fetched 2026-10-28 (unofficial)
