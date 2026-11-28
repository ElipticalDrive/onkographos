# Voxel Instance Rendering with Structured Buffer

Also called: Instanced Rendering with Scale Culling, Instancing with zero-scale hiding, StructuredBuffer Voxel Instancing

## Overview
Voxel Instance Rendering with Structured Buffer is a technique for rendering dynamic voxel grids by treating individual voxels as instanced geometry, where visibility is controlled by setting the instance scale to zero rather than rebuilding geometry buffers [S1, unofficial]. It differs from static voxel rendering methods by allowing arbitrary voxel removal or modification at runtime with a single draw call, trading off vertex shader complexity for reduced draw call overhead compared to per-voxel rendering [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A structured buffer (SSBO/StructuredBuffer) containing per-voxel instance data (position, color, scale) and a base geometry representing a single unit cube.
- **Output:** A rasterized image of the visible voxels with lighting and shadows applied.

**Data Structures:**
The instance buffer stores per-voxel records. Based on the source implementation, the layout is:
- `Position`: `float3` (world position offset of the voxel) [S2, unofficial].
- `Color`: `float3` (albedo or instance color) [S2, unofficial].
- `Scale`: Implicitly handled via a global `_VoxelScale` multiplier and the zero-scale trick [S1, S2, unofficial]. Note: The source [S1, unofficial] suggests adding a scale value to the instance buffer, while [S2, unofficial] applies a global `_VoxelScale` in the shader. The zero-scale hiding technique [S1, unofficial] implies that if a voxel is hidden, its effective scale becomes zero.

**Numbered Steps:**
1. **Create Base Geometry:** Define a single unit cube mesh with vertex positions, texture coordinates, and normals [S1, unofficial].
2. **Populate Instance Buffer:** Create a structured buffer containing per-voxel data. For each voxel in the grid, store its world position and color [S1, S2, unofficial].
3. **Vertex Shader Execution:**
   - Load instance data using the built-in `gl_InstanceID` (or `SV_InstanceID`) to index into the structured buffer [S2, unofficial].
   - Calculate the world position by adding the local vertex position to the instance position [S2, unofficial].
   - Transform the world position to clip space, applying a global voxel scale factor [S2, unofficial].
   - If a voxel is to be hidden, its scale is set to zero, effectively collapsing the geometry to a point or removing it from view [S1, unofficial].
4. **Lighting Calculation:**
   - Compute ambient lighting using Spherical Harmonics (SH9) based on the world normal [S2, unofficial].
   - Compute diffuse lighting using the dot product of the normal and light direction [S2, unofficial].
   - Apply shadow attenuation [S2, unofficial].
5. **Fragment Shader Execution:**
   - Sample the albedo texture using the interpolated UV coordinates [S2, unofficial].
   - Combine albedo, instance color, diffuse lighting, shadow, and ambient lighting to produce the final color [S2, unofficial].
6. **Draw Call:** Issue a single instanced draw call (e.g., `glDrawArraysInstanced` or `glDrawElementsInstanced`) with the instance count equal to the total number of voxels in the buffer [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
struct VoxelInstance {
    float3 Position;
    float3 Color;
};

StructuredBuffer<VoxelInstance> voxelBuffer;
float _VoxelScale;

void main() {
    // Load instance data
    VoxelInstance data = voxelBuffer[gl_InstanceID];
    
    // Get local vertex position from base cube geometry
    float3 localPosition = vertexPosition;
    
    // Calculate world position
    float3 worldPosition = data.Position + localPosition;
    
    // Transform to clip space with global voxel scale
    // Note: Zero-scale hiding is achieved by setting data.Position or a scale factor to zero
    // If scale is zero, the vertex is collapsed
    float4 clipPosition = UnityObjectToClipPos(worldPosition * _VoxelScale);
    
    // Compute lighting
    float3 worldNormal = normalize(normalMatrix * vertexNormal);
    float3 ambient = ShadeSH9(float4(worldNormal, 1.0f));
    float ndotl = max(dot(worldNormal, lightDirection), 0.0f);
    float3 diffuse = ndotl * _LightColor0.rgb;
    
    // Pass to fragment shader
    output.uv = vertexUV;
    output.color = data.Color;
    output.diffuse = diffuse;
    output.ambient = ambient;
    output.pos = clipPosition;
}

# Fragment Shader
void main() {
    // Sample albedo
    fixed4 albedo = tex2D(_MainTex, input.uv);
    
    // Combine lighting
    float3 lighting = input.diffuse * shadowAttenuation + input.ambient * 0.5;
    
    // Final color
    fixed4 output = fixed4(albedo.rgb * input.color * lighting, 1.0);
    return output;
}
```

**Parameters:**
- `_VoxelScale`: Global scale factor for all voxels [S2, unofficial].
- `shadowAttenuation`: Shadow factor applied to diffuse lighting [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Dynamic vs. Static:** This method is designed for dynamic voxels (moving, rotating, changing visibility). For static scenes, other methods are more efficient [S1, unofficial].
- **Zero-Scale Hiding:** Setting scale to zero hides the voxel but may still incur vertex shader overhead. For large numbers of hidden voxels, consider culling strategies [S1, unofficial].
- **Buffer Size:** The instance buffer size must accommodate the maximum number of voxels in the grid, even if many are hidden [S1, unofficial].

## Operating characteristics
- **Time Complexity:** O(N) for vertex processing, where N is the number of voxels. The draw call is O(1) regardless of N.
- **Memory Complexity:** O(N) for the instance buffer, storing position and color per voxel [S1, S2, unofficial].
- **Performance:** Reduces draw calls from N (per-voxel) to 1, significantly improving performance for dynamic scenes [S1, unofficial].
- **Strengths:** Simple implementation, supports dynamic visibility changes without geometry rebuilds [S1, unofficial].
- **Limitations:** Vertex shader overhead for hidden voxels, potential cache inefficiency due to dynamic SSBO access [G].
- **When to Use:** Dynamic voxel grids with frequent visibility changes [S1, unofficial].
- **When Not to Use:** Static scenes or scenes with very high voxel counts where culling is more efficient [S1, unofficial].

## Background
This technique is commonly used in voxel-based games and simulations where dynamic modification of the voxel grid is required. It leverages hardware instancing capabilities to efficiently render large numbers of similar objects [S1, S2, unofficial].

## Sources
[S1] How do I create a cube made out of smaller cubes (voxels)?, https://stackoverflow.com/questions/53039647/how-do-i-create-a-cube-made-out-of-smaller-cubes-voxels, fetched 2026-10-26 (unofficial)
[S2] Voltonik/VolumetricSmoke, https://github.com/Voltonik/VolumetricSmoke, fetched 2026-10-28 (unofficial)
