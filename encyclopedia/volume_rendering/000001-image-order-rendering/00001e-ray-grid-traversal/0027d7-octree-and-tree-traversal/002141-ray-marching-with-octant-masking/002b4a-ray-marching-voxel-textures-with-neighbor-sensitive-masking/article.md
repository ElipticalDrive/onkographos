# Ray Marching Voxel Textures with Neighbor-Sensitive Masking

Also called: Hybrid Geometry-Shader-spawned cubes with ray marched volume textures, Voxel Mask Ray Marching

## Overview
This method renders dense voxel data by combining geometry shader instancing for component-level bounding boxes with ray marching through dense volume textures for the interior [S1, S2, unofficial]. It utilizes a neighbor-sensitive bitmask to determine voxel visibility based on adjacent components, effectively hiding internal faces without storing dense geometry for every voxel [S1, S2, unofficial]. Unlike pure instancing or static vertex buffers, this hybrid approach allows for dynamic updates without recomputing complex meshes while avoiding the high vertex count of instanced cubes [S2, unofficial]. It builds on [Ray Marching with Octant Masking](2488-ray-marching-with-octant-masking.md) by replacing octant-based traversal with uniform grid ray marching through a 2D array texture [S1, S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- `gs_objectPosition`: `vec3`, the position of the component in world space [S1, S2, unofficial].
- `gs_neighbor_material_mipmap`: `ivec3`, flat input from geometry shader [S1, S2, unofficial].
  - `x`: Bits 0-5 are neighbor material flags; bits 24-31 are `materialIndex` [S1, S2, unofficial].
  - `y`: `codes.x` for neighbor material [S1, S2, unofficial].
  - `z`: LOD level index (mipmap level) [S1, S2, unofficial].
- `gs_texCoord`: `vec3`, texture coordinates within the component box [S1, S2, unofficial].
- `u_componentTex`: `isampler2DArray`, uniform volume texture storing voxel data [S1, S2, unofficial]. Each texel fetch returns a 2D code (`ivec2`) [S2, unofficial].
- `c_mWorldView`: `mat4x4`, world-view matrix [S1, S2, unofficial].
- `c_mInverseWorldView`: `mat4x4`, inverse world-view matrix [S1, S2, unofficial].
- `c_maxTexRes`: `int`, maximum texture resolution used to calculate LOD-dependent texel size [S1, S2, unofficial].

**Outputs:**
- `fragColor`: `vec4`, the final color of the fragment [S1, S2, unofficial].

### Data Structures
**Voxel Mask:**
An integer bitmask where each bit represents a side of the voxel (bits 0-5) and one bit (bit 6, `0x40`) represents non-side-dependent voxels [S2, unofficial]. The mask is computed as:
$$ \text{voxelMask} = 0x40 \mid ((\sim \text{gs\_neighbor\_material\_mipmap}.x) \ \& \ 0x3f) $$
Visibility is determined by combining this mask with the texture mask using a logical AND: `(codes.y & voxelMask) == 0` indicates the voxel is not visible [S1, S2, unofficial].

**Ray Marching State:**
- `rayPos`: `ivec3`, current voxel coordinates in the texture [S1, S2, unofficial].
- `d`: `vec3`, distance to the next plane crossing along each axis [S1, S2, unofficial].
- `projLength`: `vec3`, projected distance between planes ($1 / |\text{dir}|$) [S1, S2, unofficial].
- `dirSign`: `ivec3`, sign of the view direction components ($1$ if $\ge 0$, $-1$ if $< 0$) [S1, S2, unofficial].

### Algorithm Steps
1. **Compute Component-Space Position:**
   Calculate the position in chunk/component space:
   $$ \text{chunkSpacePos} = \text{gs\_objectPosition} + \text{gs\_texCoord} - 0.5 $$
   [S1, S2, unofficial]

2. **Transform View Direction:**
   Transform the view direction to chunk space:
   $$ \text{viewDir} = \text{normalize}((\text{vec4}(\text{chunkSpacePos}, 1) \times \text{c\_mWorldView}).\text{xyz}) $$
   $$ \text{chunkSpaceDir} = \text{viewDir} \times \text{mat3x3}(\text{c\_mInverseWorldView}) $$
   [S1, S2, unofficial]

3. **Calculate Texture Resolution:**
   Determine the current texture size based on LOD:
   $$ \text{texSize} = \text{c\_maxTexRes} / (1 \ll \text{gs\_neighbor\_material\_mipmap}.z) $$
   [S1, S2, unofficial]

4. **Initialize Ray Marching:**
   - Compute direction signs:
     $$ \text{dirSign}.x = (\text{chunkSpaceDir}.x < 0.0) ? -1 : 1 $$
     $$ \text{dirSign}.y = (\text{chunkSpaceDir}.y < 0.0) ? -1 : 1 $$
     $$ \text{dirSign}.z = (\text{chunkSpaceDir}.z < 0.0) ? -1 : 1 $$
     [S1, S2, unofficial]
   - Calculate initial voxel position and ray position:
     $$ \text{voxelPos} = \text{gs\_texCoord} \times (\text{texSize} - 0.00001) $$
     $$ \text{rayPos} = \text{ivec3}(\text{voxelPos}) $$
     [S1, S2, unofficial]
   - Compute projected lengths and initial distances:
     $$ \text{absDir} = |\text{chunkSpaceDir}| $$
     $$ \text{projLength} = 1.0 / (\text{absDir} + 0.0001) $$
     $$ d = \text{voxelPos} - \text{rayPos} $$
     Adjust $d$ based on direction signs:
     If $\text{dirSign}.x == 1$, $d.x = 1 - d.x$ (same for $y, z$).
     $$ d = d \times \text{projLength} $$
     [S1, S2, unofficial]

5. **Ray March Loop:**
   If `materialIndex != 0` and `texSize > 1`:
   - Fetch initial voxel code:
     $$ \text{codes} = \text{texelFetch}(\text{u\_componentTex}, \text{ivec3}(\text{rayPos}.x + \text{rayPos}.y \times \text{texSize}, \text{rayPos}.z, \text{materialIndex}), \text{gs\_voxel\_material\_mipmap}.z).\text{xy} $$
     [S1, S2, unofficial]
   - While `(codes.y & voxelMask) == 0`:
     - Determine step direction:
       - If $d.x < d.y$ or $d.z < d.y$:
         - If $d.x < d.z$: Step in X.
           $$ \text{rayPos}.x += \text{dirSign}.x $$
           $$ d.yz -= d.x $$
           $$ d.x = \text{projLength}.x $$
         - Else: Step in Z.
           $$ \text{rayPos}.z += \text{dirSign}.z $$
           $$ d.xy -= d.z $$
           $$ d.z = \text{projLength}.z $$
       - Else: Step in Y.
         $$ \text{rayPos}.y += \text{dirSign}.y $$
         $$ d.xz -= d.y $$
         $$ d.y = \text{projLength}.y $$
     - Boundary Check:
       If $\text{rayPos} \ge \text{ivec3}(\text{texSize}, \text{texSize}, \text{texSize})$ or $\text{rayPos} < \text{ivec3}(0, 0, 0)$, discard fragment [S1, S2, unofficial].
     - Fetch next voxel code:
       $$ \text{codes} = \text{texelFetch}(\text{u\_componentTex}, \text{ivec3}(\text{rayPos}.x + \text{rayPos}.y \times \text{texSize}, \text{rayPos}.z, \text{materialIndex}), \text{gs\_voxel\_material\_mipmap}.z).\text{xy} $$
       [S1, S2, unofficial]

6. **Estimate Normal:**
   Calculate distance to plane intersection:
   $$ d = |d - \text{projLength}| $$
   [S1, S2, unofficial]
   Determine normal based on which axis had the smallest distance (indicating the plane crossed):
   - If $d.x < 0.0001$:
     $$ \text{normal} = -\text{dirSign}.x \times \text{vec3}(\text{c\_mWorldView}[0][0], \text{c\_mWorldView}[1][0], \text{c\_mWorldView}[2][0]) $$
   - Else if $d.y < 0.0001$:
     $$ \text{normal} = -\text{dirSign}.y \times \text{vec3}(\text{c\_mWorldView}[0][1], \text{c\_mWorldView}[1][1], \text{c\_mWorldView}[2][1]) $$
   - Else if $d.z < 0.0001$:
     $$ \text{normal} = -\text{dirSign}.z \times \text{vec3}(\text{c\_mWorldView}[0][2], \text{c\_mWorldView}[1][2], \text{c\_mWorldView}[2][2]) $$
   [S1, S2, unofficial]
   Normalize the normal:
   $$ \text{normal} = \text{normalize}(\text{normal}) $$
   [S1, S2, unofficial]

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs: gs_objectPosition, gs_neighbor_material_mipmap, gs_texCoord, u_componentTex, c_mWorldView, c_mInverseWorldView, c_maxTexRes
# Outputs: fragColor

def main():
    # 1. Compute Component-Space Position
    chunkSpacePos = gs_objectPosition + gs_texCoord - 0.5
    
    # 2. Transform View Direction
    viewDir = normalize((vec4(chunkSpacePos, 1) * c_mWorldView).xyz)
    chunkSpaceDir = viewDir * mat3x3(c_mInverseWorldView)
    
    # 3. Calculate Texture Resolution
    texSize = c_maxTexRes / (1 << gs_neighbor_material_mipmap.z)
    
    # 4. Initialize Ray Marching
    dirSign = ivec3(1, 1, 1)
    if chunkSpaceDir.x < 0.0: dirSign.x = -1
    if chunkSpaceDir.y < 0.0: dirSign.y = -1
    if chunkSpaceDir.z < 0.0: dirSign.z = -1
    
    voxelPos = gs_texCoord * (texSize - 0.00001)
    rayPos = ivec3(voxelPos)
    
    absDir = abs(chunkSpaceDir)
    projLength = 1.0 / (absDir + 0.0001)
    
    d = voxelPos - rayPos
    if dirSign.x == 1: d.x = 1.0 - d.x
    if dirSign.y == 1: d.y = 1.0 - d.y
    if dirSign.z == 1: d.z = 1.0 - d.z
    d = d * projLength
    
    # 5. Ray March Loop
    materialIndex = gs_neighbor_material_mipmap.x >> 24
    if materialIndex != 0 and texSize > 1:
        # Compute voxel mask
        voxelMask = 0x40 | ((~gs_neighbor_material_mipmap.x) & 0x3f)
        
        # Fetch initial code
        texCoord = ivec3(rayPos.x + rayPos.y * texSize, rayPos.z, materialIndex)
        codes = texelFetch(u_componentTex, texCoord, gs_neighbor_material_mipmap.z).xy
        
        while (codes.y & voxelMask) == 0:
            # Determine step direction
            if d.x < d.y or d.z < d.y:
                if d.x < d.z:
                    rayPos.x += dirSign.x
                    d.yz -= d.x
                    d.x = projLength.x
                else:
                    rayPos.z += dirSign.z
                    d.xy -= d.z
                    d.z = projLength.z
            else:
                rayPos.y += dirSign.y
                d.xz -= d.y
                d.y = projLength.y
            
            # Boundary Check
            if any(rayPos >= ivec3(texSize, texSize, texSize)) or any(rayPos < ivec3(0, 0, 0)):
                discard()
            
            # Fetch next code
            texCoord = ivec3(rayPos.x + rayPos.y * texSize, rayPos.z, materialIndex)
            codes = texelFetch(u_componentTex, texCoord, gs_neighbor_material_mipmap.z).xy
    
    # 6. Estimate Normal
    d = abs(d - projLength)
    normal = vec3(0.0)
    if d.x < 0.0001:
        normal = -dirSign.x * vec3(c_mWorldView[0][0], c_mWorldView[1][0], c_mWorldView[2][0])
    elif d.y < 0.0001:
        normal = -dirSign.y * vec3(c_mWorldView[0][1], c_mWorldView[1][1], c_mWorldView[2][1])
    elif d.z < 0.0001:
        normal = -dirSign.z * vec3(c_mWorldView[0][2], c_mWorldView[1][2], c_mWorldView[2][2])
    
    normal = normalize(normal)
    
    # Output color (implementation specific)
    fragColor = vec4(1.0, 1.0, 1.0, 1.0) # Placeholder for shading
```

### Parameters
- `c_maxTexRes`: The maximum texture resolution. Typical value is not explicitly stated as a number, but used to calculate `texSize` [S1, S2, unofficial].

### Edge Cases and Pitfalls
- **Boundary Check:** If `rayPos` exceeds texture bounds (`>= texSize` or `< 0`), the fragment must be discarded to prevent out-of-bounds access [S1, S2, unofficial].
- **LOD Artifacts:** Computing LOD in the fragment shader causes artifacts because a LOD seam can run through a single component, creating holes. LOD is computed in the geometry shader instead [S1, S2, unofficial].

## Subtopics

### Ray Marching with Octant Masking
The parent method using octant-based traversal. This method replaces it with uniform grid ray marching through volume textures for better performance on dense data [S1, S2, unofficial].

### Gigavoxels
A related technique using octrees for efficient ray-casting of 3D textures. This method differs by using uniform grids and volume textures, avoiding the complexity of octree traversal for dynamic transformations [S2, unofficial].

## Operating characteristics
- **Time Complexity**: The method reduces frame time from over 20ms to 3-4ms on a notebook GTX850M for 62,663 components with 16x16x16 voxels each [S1, S2, unofficial].
- **Memory Complexity**: Uses volume textures for dense internal data, avoiding the high vertex count of instanced cubes [S2, unofficial].
- **Performance**: Renders approximately 256 million theoretical voxels in 4.45ms on a notebook GTX850M [S1, S2, unofficial].
- **Strengths**: Allows dynamic updates without recomputing complex meshes; avoids high vertex count; efficient for dense voxel data [S2, unofficial].
- **Limitations**: More sensitive to screen resolution; LOD artifacts if computed in fragment shader [S1, S2, unofficial].
- **When to Use**: When rendering dense voxel data with dynamic updates and a blocky aesthetic is desired [S1, S2, unofficial].
- **When Not to Use**: When continuous surfaces are required (e.g., marching cubes) or when memory bandwidth is severely constrained [S2, unofficial].

## Background
This method was developed by Johannes Jendersie for the project Monolith, addressing the unbearable 20ms frame time of previous approaches [S1, S2, unofficial]. It was introduced in May 2015 [S1, S2, unofficial]. The technique leverages the observation that components consist of the same voxel patterns repeatedly, allowing for a hybrid approach of geometry shader instancing and ray marching [S1, S2, unofficial]. It was used to render stone and water components with 16x16x16 voxels each [S1, S2, unofficial].

## Sources
[S1] May | 2015 | Developerblog – Johannes Jendersie, https://jojendersie.de/2015/05/, fetched 2026-10-24 (unofficial)
[S2] Graphic | Developerblog – Johannes Jendersie, https://jojendersie.de/category/graphic/, fetched 2026-10-24 (unofficial)
