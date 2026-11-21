# Single-Pass GPU Voxelization

Also called: LEGOr Voxelization, Voxelization: Vertex Shader

## Overview
Single-Pass GPU Voxelization is a technique that maps 3D mesh vertices directly to a 3D grid (voxel space) in a single rasterization pass using a Frame Buffer Object (FBO), avoiding the multiple passes required by methods like Karabassi et al. [S1]. It sets itself apart from its sibling [Geometry Shader Voxelization](2566-geometry-shader-voxelization.md) by encoding voxel indices into vertex attributes and the FBO alpha channel during the vertex shader stage, rather than rasterizing triangles into a 3D texture via a geometry shader. It is distinct from [GPU Voxelization Using OpenGL Image Atomics](2544-gpu-voxelization-using-opengl-image-atomics.md) by relying on standard rasterization depth testing and FBO downloads rather than atomic operations on 3D textures.

## How it works
**Inputs:**
- `vp`: 3D vertex position $(v_{px}, v_{py}, v_{pz})$.
- `vc`: 3D vertex color $(v_{cx}, v_{cy}, v_{cz})$.
- `size`: Integer grid resolution (side length of the cubic voxel grid).

**Outputs:**
- A 2D Frame Buffer Object (FBO) where each pixel corresponds to a projected vertex.
- The FBO channels store: Red ($R$) = $v_{cx}$, Green ($G$) = $v_{cy}$, Blue ($B$) = $v_{cz}$, Alpha ($A$) = normalized voxel index.
- A Vertex Buffer Object (VBO) generated from the downloaded FBO data, used for subsequent polygonization.

**Data Structures:**
- **FBO Layout**: A 2D texture or render target with RGBA format. The spatial resolution of the FBO must be sufficient to avoid overlapping projections of distinct vertices.
- **Voxel Index**: A linear integer index $w$ representing the position in the 3D grid, calculated as $w = i_x \cdot size^2 + i_y \cdot size + i_z$, where $i_x, i_y, i_z$ are the integer voxel coordinates.

**Steps:**

1.  **Vertex Shader Execution**: For each vertex of the input mesh:
    a.  Compute the integer voxel coordinates $(i_x, i_y, i_z)$ from the vertex position $v_p$. The source text implies a direct mapping or projection, but explicitly defines the linear index calculation [S1].
    b.  Calculate the linear voxel index $w$:
        $$ w = \lfloor v_{px} \rfloor \cdot size^2 + \lfloor v_{py} \rfloor \cdot size + \lfloor v_{pz} \rfloor $$
        *Note: The source uses `int()` which typically truncates towards zero or floors depending on implementation; for voxelization, floor is standard for positive coordinates [G].*
    c.  Check bounds: If $w$ is outside the valid grid range $[0, size^3 - 1]$, set the output alpha to `NULL` (or 0) and discard the vertex for further processing [S1].
    d.  Normalize the voxel index for storage in the 8-bit (or floating-point) alpha channel:
        $$ \alpha = \frac{w}{size^3} $$
    e.  Output the vertex color $(v_{cx}, v_{cy}, v_{cz})$ to the RGB channels and $\alpha$ to the Alpha channel.

2.  **Rasterization**: The vertices are rasterized into the FBO. Standard depth testing (Z-buffer) is applied. Vertices closer to the camera overwrite those further away. Vertices mapping to the same pixel position compete for the buffer slot [S1].

3.  **FBO Download and VBO Creation**:
    a.  Read the FBO data back to the CPU.
    b.  Interpret each pixel in the FBO as a vertex.
    c.  Construct a VBO where each vertex contains the RGB color and the Alpha value (which encodes the voxel index).

4.  **Polygonization (LEGOr Brick Generation)**:
    a.  Use a Geometry Shader to process the VBO.
    b.  Convert the normalized alpha value $\alpha$ back to spatial coordinates $(i_x, i_y, i_z)$ to find the center of the cube.
    c.  Generate the geometry (triangles) for the LEGOr brick (voxel) at that location.
    d.  Discard vertices with illegal/null values [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader Logic
def vertex_shader(vp: vec3, vc: vec3, size: int) -> vec4:
    # 1. Compute integer voxel coordinates
    # Assuming vp is already in voxel space [0, size]
    ix = int(vp.x)
    iy = int(vp.y)
    iz = int(vp.z)
    
    # 2. Compute linear voxel index w
    w = ix * size * size + iy * size + iz
    
    # 3. Check bounds
    max_voxels = size * size * size
    if w < 0 or w >= max_voxels:
        # Return null/invalid marker. 
        # Using 0.0 for alpha might conflict with valid index 0, 
        # but source says "null value". We assume downstream logic handles 0 or 
        # a specific sentinel. For implementation safety, we might use -1.0 
        # if alpha is float, or 0.0 if 0 is reserved for invalid.
        # Source says: "receives a null value as voxel number"
        return vec4(vc.r, vc.g, vc.b, 0.0) # Assuming 0.0 alpha indicates null/invalid
        
    # 4. Normalize voxel index for alpha channel
    alpha = w / max_voxels
    
    # 5. Emit pixel with color in RGB and normalized index in Alpha
    return vec4(vc.r, vc.g, vc.b, alpha)

# Post-Processing: Reconstructing Voxel Index from VBO
def reconstruct_voxel_index(alpha: float, size: int) -> int:
    max_voxels = size * size * size
    # Reverse the normalization
    w = int(alpha * max_voxels)
    
    # Convert linear index back to 3D coordinates
    iz = w % size
    w = w // size
    iy = w % size
    ix = w // size
    
    return vec3(ix, iy, iz)
```

**Parameters:**
- `size`: The grid resolution (e.g., 128, 256). Determines the precision of the voxelization [S1].
- FBO Resolution: Must be large enough to prevent overlapping projections of distinct vertices. Typical values are 512x512 for resolutions up to $128^3$ voxels, and >1024x1024 for complex objects [S1].

**Edge Cases and Pitfalls:**
- **Holes in Back Surfaces**: Because the method relies on vertex projection and Z-buffering, back-facing surfaces may be occluded by front-facing vertices, resulting in holes. This is mitigated by increasing the FBO size to reduce vertex overlap and ensure distinct vertices map to distinct pixels [S1].
- **Vertex Overlap**: Complex objects may have many vertices projecting to the same pixel. The Z-buffer resolves this by keeping only the closest vertex, potentially losing interior geometry information if not handled by the specific application (e.g., LEGOr modeling only requires surface representation) [S1].
- **Precision**: The normalization of the voxel index into the alpha channel introduces quantization error. Using floating-point FBOs is recommended to preserve index integrity [G].

## Subtopics

### Single-Pass GPU Voxelization via 3D Texture Render Target
A variant that utilizes the capability to set a 3D texture as a direct render target. This allows the recording of existence, color, and normal information directly into the voxel without the encoding/decoding mechanism required by the FBO-alpha method [S2, unofficial]. It uses a geometry shader to transform complex surfaces into high-resolution voxels in one pass. It computes an absorption coefficient based on the number of surface drawings per voxel and stores surface normals, transmission coefficients, and refraction indices for use in a multiple refraction algorithm for transparent objects [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(V)$ for the vertex shader pass, where $V$ is the number of vertices. The subsequent geometry shader pass is $O(P)$, where $P$ is the number of projected pixels (voxels).
- **Memory Complexity**: $O(size^2)$ for the FBO storage (2D projection) and $O(size^3)$ for the final voxel grid if explicitly stored, though the method primarily outputs a VBO of projected vertices.
- **Performance**: On an AMD Athlon 64 3500+ with 2GB RAM and GeForce 8800GTX, a 512x512 FBO produces good results for $128^3$ voxel resolutions. Complex objects require FBO sizes >1024x1024 [S1].
- **Speed**: Up to 5 times faster than Passalis et al. [S1], which renders the model six times using multiple buffers. This method renders the model only once using a single frame buffer [S1].
- **Strengths**: Single-pass rasterization; real-time performance; no preprocessing required [S2, unofficial].
- **Limitations**: Dependent on model surface complexity; prone to holes in back surfaces if FBO is too small; misses concavities in concave objects if not carefully tuned (though less prone than Karabassi et al. [S1]).

## Background
The method was introduced in the "Legolizer" system, described in the paper *Legolizer: A Real-Time System for Modeling and Rendering LEGO Representations of Boundary Models* [S1]. It was developed to create LEGO-like representations of 3D models efficiently. The approach was compared against Passalis et al. [S1] and Karabassi et al. [S1]. A related concept using 3D texture render targets for voxelization was explored in *Real-time Realistic Voxel-based Rendering* [S2, unofficial], which extended the single-pass idea to include normal and optical property storage.

## Sources
[S1] Legolizer: A Real-Time System for Modeling and Rendering LEGO Representations of Boundary Models, https://doi.org/10.1109/sibgrapi.2009.18, fetched 2026-11-02
[S2] Real-time Realistic Voxel-based Rendering, https://doi.org/10.2312/pe.pg.pg2011short.001-006, fetched 2026-10-27 (unofficial)
