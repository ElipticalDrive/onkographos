# Heightmap Terrain
Also called: Heightmap, Heightfield, 2.5D Terrain

## Overview
Heightmap terrain is a storage and rendering method where landscape geometry is defined by a single height value ($z$) for every $(x, y)$ coordinate, typically stored in a 2D texture. It solves the problem of efficiently representing large, continuous outdoor environments by decoupling the data storage from the mesh generation, which is performed dynamically via geometry shaders or hardware tessellation. Unlike voxel terrain, which stores volume data, heightmaps store only surface elevation, making them significantly faster to render and more memory-efficient for collision detection, though they cannot represent overhangs or caves [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 2D texture (or array) containing height values. A global origin position and a global resolution (world units per texel) for the entire terrain quad.
*   **Output:** A triangle mesh representing the terrain surface, generated per frame.

**Data Structures**
*   **Heightmap Texture:** A 2D texture where each texel stores a single floating-point or fixed-point value representing the height $z$. The texture coordinates $(u, v)$ map linearly to world $(x, y)$.
*   **Terrain Metadata:** A structure containing:
    *   `origin`: World-space $(x, y, z)$ of the bottom-left corner of the heightmap.
    *   `resolution`: World units per texel (or total world width/height and texture dimensions).
    *   `dimensions`: Width and height of the heightmap in texels.

**Algorithm Steps**
1.  **Initialize Metadata:** Set the global origin and resolution. Define the heightmap texture dimensions $W \times H$.
2.  **Rasterize/Generate Height Data:** Populate the 2D texture with height values. This can be done via:
    *   Procedural generation (e.g., fractal Perlin noise) [S1, unofficial].
    *   Loading from an image file.
    *   Editing in a heightmap editor.
3.  **Generate Geometry (Per Frame):**
    *   **Option A: Geometry Shader:** Pass a single quad (or a grid of quads) to the GPU. The geometry shader iterates over the heightmap texture, sampling heights at specific intervals to generate vertices.
    *   **Option B: Hardware Tessellation:** Use a tessellation shader to subdivide a coarse mesh. The displacement shader samples the heightmap texture to displace vertices along the $z$-axis.
4.  **Vertex Position Calculation:** For a texel at grid coordinates $(i, j)$ with height $h$:
    $$ x = x_{origin} + i \cdot \Delta x $$
    $$ y = y_{origin} + j \cdot \Delta y $$
    $$ z = h $$
    Where $\Delta x$ and $\Delta y$ are the world units per texel derived from the resolution.
5.  **Render:** Draw the generated triangles.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for CPU-side setup and GPU-side conceptual logic

# 1. Data Structure Definition
class HeightmapTerrain:
    def __init__(self, origin_x, origin_y, origin_z, world_width, world_height, texture_width, texture_height):
        self.origin = (origin_x, origin_y, origin_z)
        self.world_width = world_width
        self.world_height = world_height
        self.tex_width = texture_width
        self.tex_height = texture_height
        self.height_texture = None # 2D array or GPU texture

    def sample_height(self, world_x, world_y):
        """
        Sample height from the texture given world coordinates.
        Returns z value.
        """
        # Calculate texel coordinates
        u = (world_x - self.origin[0]) / self.world_width
        v = (world_y - self.origin[1]) / self.world_height
        
        # Clamp to [0, 1]
        u = max(0.0, min(1.0, u))
        v = max(0.0, min(1.0, v))
        
        # Map to texture coordinates
        tex_u = u * self.tex_width
        tex_v = v * self.tex_height
        
        # Sample height (implementation depends on texture API)
        return self.height_texture.sample(tex_u, tex_v)

# 2. Geometry Generation Concept (GPU-side logic)
def generate_terrain_mesh(terrain: HeightmapTerrain, lod_level: int):
    """
    Conceptual generation of mesh vertices.
    In practice, this is done in a Geometry Shader or Tessellation Shader.
    """
    vertices = []
    
    # Determine step size based on LOD
    # Higher LOD level -> larger step (fewer vertices)
    step = 1 << lod_level 
    
    for i in range(0, terrain.tex_width, step):
        for j in range(0, terrain.tex_height, step):
            # Calculate world position
            # Note: Texture coordinates usually map (0,0) to bottom-left or top-left
            # Adjust mapping based on specific engine conventions
            u = i / terrain.tex_width
            v = j / terrain.tex_height
            
            world_x = terrain.origin[0] + u * terrain.world_width
            world_y = terrain.origin[1] + v * terrain.world_height
            
            # Sample height
            height = terrain.sample_height(world_x, world_y)
            
            vertices.append((world_x, world_y, height))
            
    # Generate indices to form triangles from vertices
    # This part is typically handled by the GPU primitive assembly
    return vertices
```

**Parameters**
*   **Texture Resolution:** Typically powers of two (e.g., 512x512, 1024x1024) to support hardware tessellation and mipmapping [S1, unofficial].
*   **LOD Level:** Controls the granularity of the generated mesh. Dynamic LOD adjusts this based on camera distance [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Non-Planar Quads:** If the four vertices defining a sub-quad are not on the same plane, the diagonal split in the triangle fan or strip becomes visible as an artifact. This is common on steep cliffs with edges not aligned to cardinal directions [S1, unofficial].
*   **Overhangs/Holes:** The method assumes a function $z = f(x, y)$. It cannot represent overhangs, caves, or vertical walls where multiple $z$ values exist for a single $(x, y)$ [S1, unofficial].
*   **Dynamic LOD Shifting:** Changing mesh resolution based on distance can cause "popping" or shifting geometry if the resolution drops too drastically [S1, unofficial].

## Operating characteristics
*   **Memory:** Very low. Only one value (height) per vertex is stored. No indices are required if geometry is generated on the fly [S1, unofficial].
*   **Speed:** Relatively fast rendering. The geometry shader is small and efficient. However, it is slower than pre-baked geometry terrain [S1, unofficial].
*   **Collision Detection:** Highly efficient. A horizontal position $(x, y)$ maps directly to one or four memory locations in the heightmap texture, allowing for fast lookups [S1, unofficial].
*   **Hardware Requirements:** Requires a GPU with geometry shader support or hardware tessellation capabilities. On systems without triangle-based 3D acceleration, ray marching heightmaps is the fastest alternative [S1, unofficial].
*   **Limitations:** Cannot represent non-manifold geometry (overhangs, caves). Precision is limited by texture resolution and bit depth.

## Background
Heightmaps have been a staple in terrain rendering since the early days of 3D graphics. They are used in numerous games and engines, including *Just Cause 2* (square sectors), *The Sims 2* (neighborhood terrain), and Valve's Source engine (where they are implemented as "Displacements," which are tessellated quads with powers-of-two resolutions) [S1, unofficial]. The method is often combined with noise functions for procedural generation [S1, unofficial].

## Sources
[S1] Heightmap, Voxel, Polygon (geometry) terrains, https://gamedev.stackexchange.com/questions/15573/heightmap-voxel-polygon-geometry-terrains, fetched 2026-10-26 (unofficial)
