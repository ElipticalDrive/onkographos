# GPU-based 3D Texture Advection with Logical-to-Physical Mapping

Also called: 2D texture-based implementation, 3D GPU-based texture advection, Semi-Lagrangian 3D Texture Advection

## Overview
This method visualizes unsteady 3D flow by advecting a property field $\rho(x,t)$ along a vector field $v(x,t)$ using a semi-Lagrangian approach, implemented on GPUs by mapping logical 3D grid memory to physical 2D texture memory to avoid slow 3D texture updates [S1]. It sets itself apart from its parent [GPU-based 3D Texture Advection](15149-gpu-based-3d-texture-advection.md) by replacing expensive 3D texture read/write operations with efficient 2D texture operations, and from its sibling [Texture Advection](18803-texture-advection.md) by explicitly defining the logical-to-physical coordinate mapping required to simulate 3D advection on 2D hardware [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A uniform 3D grid representing the property field $\rho$ and the vector field $v$ at time $t$. The grid coordinates are logical coordinates $x = (x, y, z)$.
*   **Output:** The updated property field $\rho$ at time $t + \Delta t$, stored in physical 2D texture memory.

**Data Structures:**
*   **Logical Memory:** A uniform 3D grid with coordinates $x = (x, y, z)$.
*   **Physical Memory:** A 2D uniform grid represented by one or more 2D textures with coordinates $u = (u, v)$.
*   **Tile Mapping:** A slice of constant $z$ in logical memory corresponds to a tile in physical memory. Tiles are positioned in row-first order. If the 3D domain exceeds the maximum size of a single 2D texture, multiple 2D textures are used, labeled by an integer index $i_{tex}$.
*   **Borders:** A one-texel-wide border is added around each tile in physical memory to prevent erroneous influence from neighboring tiles during bilinear interpolation [S1].

**Mapping Function:**
The mapping from logical to physical memory is defined as $\Phi : (x, y, z) \mapsto ((u, v); i_{tex})$. This is decomposed into:
$$ \Phi : (x, y, z) \mapsto (u_0 + u_{local}; i_{tex}) $$
where:
*   $u_0 = \Phi^0_u(z)$ is the origin of the tile in physical memory.
*   $i_{tex} = \Phi^0_{i_{tex}}(z)$ is the texture index for the tile.
*   $u_{local} = (s_x x, s_y y)$ are the local coordinates within the tile.
*   $s_x, s_y$ are scaling factors accounting for the different relative sizes of texels in logical and physical memory [S1].
*   $\Phi^0$ maps the logical $z$ value to the tile origin and texture index. This function can be implemented via a lookup table or dependent texture lookup [S1].

**Algorithm Steps:**
1.  **Initialize:** Set up physical 2D textures. Ensure each tile has a one-texel-wide border [S1].
2.  **Iterate Slices:** For each slice $z$ in the logical 3D domain:
    a.  Determine the corresponding tile origin $u_0$, texture index $i_{tex}$, and local scaling $s_x, s_y$ using $\Phi^0$ [S1].
    b.  Render a quadrilateral representing the 2D subset of the full 3D domain (the current slice) into the physical 2D texture. Restrict the viewport to the corresponding subregion of the physical texture [S1].
    c.  In the fragment program, for each fragment (representing a grid point in the current slice):
        i.  Compute the backward position $x_{prev}$ using first-order Euler integration:
            $$ x_{prev} = x(t) - \Delta t v(x(t), t) $$
            where $x(t)$ is the current logical position, $\Delta t$ is the time step, and $v$ is the vector field at $x(t)$ and $t$ [S1].
        ii. Map $x_{prev}$ to physical coordinates using $\Phi$. Let $x_{prev} = (x_{prev}, y_{prev}, z_{prev})$.
        iii. Determine the two closest tiles along the $z$-axis in physical memory that contain the $z$-coordinate of $x_{prev}$. Let these be tile $T_1$ (at $z_1$) and tile $T_2$ (at $z_2$), where $z_1 \le z_{prev} \le z_2$ [S1].
        iv. Map the $x_{prev}, y_{prev}$ coordinates to local coordinates within $T_1$ and $T_2$:
            $$ u_{local,1} = (s_x x_{prev}, s_y y_{prev}) $$
            $$ u_{local,2} = (s_x x_{prev}, s_y y_{prev}) $$
            (Note: $s_x, s_y$ are constant for the slice, but may vary if the mapping $\Phi^0$ changes with $z$. The source implies $s_x, s_y$ are derived from the logical $x,y$ scaling, so they are consistent for a given $x,y$ regardless of $z$, but the tile origin $u_0$ changes with $z$ [S1]).
        v.  Perform bilinear interpolation in $T_1$ at $u_{local,1}$ to get value $\rho_1$.
        vi. Perform bilinear interpolation in $T_2$ at $u_{local,2}$ to get value $\rho_2$.
        vii. Perform linear interpolation between $\rho_1$ and $\rho_2$ based on the relative position of $z_{prev}$ between $z_1$ and $z_2$ to get $\rho(x_{prev}, t - \Delta t)$ [S1].
        viii. Write $\rho(x_{prev}, t - \Delta t)$ to the current fragment's position in the physical texture (write access is restricted to grid points, i.e., single texels, so no interpolation is needed for writing) [S1].
3.  **Boundary Handling:** Use different fragment programs for boundary tiles (near the boundary between two physical textures) and internal tiles. Boundary tiles may require sampling from multiple 2D textures, while internal tiles only sample from one [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume logical grid dimensions: Nx, Ny, Nz
# Assume physical tile dimensions: TileW, TileH
# Assume physical texture dimensions: TexW, TexH
# Assume number of physical textures: NumTex
# Assume max velocity: vmax
# Assume time step: dt
# Assume vector field v(x, y, z, t) returns (vx, vy, vz)
# Assume property field rho_old is stored in physical 2D textures

# Precompute mapping table Phi0 for z in [0, Nz-1]
# Phi0[z] = (u0_u, u0_v, i_tex)
# u0_u, u0_v: origin of tile in physical texture i_tex
# i_tex: index of physical texture

# For each z_slice in range(Nz):
#     u0_u, u0_v, i_tex = Phi0[z_slice]
#     # Determine if this tile is a boundary tile
#     is_boundary = (i_tex != Phi0[z_slice + 1].i_tex) if z_slice < Nz - 1 else False
#     # Note: Boundary tiles are those near the boundary between two physical textures.
#     # The exact condition depends on the layout of tiles in physical memory.
#     # For simplicity, we assume a fragment program is selected based on whether
#     # the backward lookup might cross into another texture.
#     # The source states: "multiple 2D textures are only required in tiles that are close to the boundary
#     # between two physical textures because the maximum difference vector for the backward lookup
#     # in Eq. (2) is bounded by dt * vmax."
#     # So, we check if the backward lookup position could be in a different texture.

#     # Render a quad for this slice into physical texture i_tex, at offset (u0_u, u0_v)
#     # Viewport is restricted to the tile region (plus border)
#     for each fragment (x, y) in the quad (logical x, y coordinates):
#         # Current logical position
#         x_curr = x
#         y_curr = y
#         z_curr = z_slice

#         # Get vector field at current position
#         vx, vy, vz = v(x_curr, y_curr, z_curr, t)

#         # Backward Euler integration
#         x_prev = x_curr - dt * vx
#         y_prev = y_curr - dt * vy
#         z_prev = z_curr - dt * vz

#         # Find the two closest tiles along z-axis
#         # z_prev is in logical space. We need to find which logical z-slices
#         # correspond to the physical tiles that bound z_prev.
#         # Since logical z is discrete, z_prev might be between two integer z values.
#         # Let z_floor = floor(z_prev), z_ceil = ceil(z_prev)
#         # But z_prev is a continuous value. The tiles are at integer z slices.
#         # So we need to find the physical tiles corresponding to z_floor and z_ceil.

#         z_floor = floor(z_prev)
#         z_ceil = ceil(z_prev)

#         # Clamp z_floor and z_ceil to valid range [0, Nz-1]
#         z_floor = max(0, min(Nz - 1, z_floor))
#         z_ceil = max(0, min(Nz - 1, z_ceil))

#         # Get physical tile info for z_floor and z_ceil
#         u0_u_floor, u0_v_floor, i_tex_floor = Phi0[z_floor]
#         u0_u_ceil, u0_v_ceil, i_tex_ceil = Phi0[z_ceil]

#         # Local coordinates in each tile
#         # s_x, s_y are scaling factors. The source says u_local = (s_x * x, s_y * y).
#         # It does not specify how s_x, s_y are computed, but they account for
#         # different relative sizes of texels in logical and physical memory.
#         # We assume s_x, s_y are constants or precomputed per slice.
#         # For simplicity, we assume they are constants here.
#         u_local_floor_x = s_x * x_prev
#         u_local_floor_y = s_y * y_prev
#         u_local_ceil_x = s_x * x_prev
#         u_local_ceil_y = s_y * y_prev

#         # Perform bilinear interpolation in tile at z_floor
#         # The physical coordinates in the texture are (u0_u_floor + u_local_floor_x, u0_v_floor + u_local_floor_y)
#         rho_floor = bilinear_interpolate(texture[i_tex_floor], u0_u_floor + u_local_floor_x, u0_v_floor + u_local_floor_y)

#         # Perform bilinear interpolation in tile at z_ceil
#         rho_ceil = bilinear_interpolate(texture[i_tex_ceil], u0_u_ceil + u_local_ceil_x, u0_v_ceil + u_local_ceil_y)

#         # Linear interpolation between rho_floor and rho_ceil
#         # The weight is based on the relative position of z_prev between z_floor and z_ceil
#         # z_prev is in logical space. The distance between z_floor and z_ceil is 1.
#         # So the weight is (z_prev - z_floor) / (z_ceil - z_floor) = z_prev - z_floor
#         weight = z_prev - z_floor
#         rho_prev = lerp(rho_floor, rho_ceil, weight)

#         # Write to current fragment position in physical texture i_tex
#         # The current fragment position in the texture is (u0_u + x, u0_v + y)
#         # But wait, the current fragment is at (x, y) in logical space.
#         # The physical position is (u0_u + s_x * x, u0_v + s_y * y)
#         # However, the source says "write access is restricted to grid points (i.e., single texels)".
#         # This implies that the fragment shader writes to a specific texel.
#         # The fragment shader is rendering a quad that covers the tile.
#         # The fragment's position in the texture is determined by the fragment's position in the quad.
#         # The quad represents the logical slice. The fragment's position in the quad corresponds to (x, y).
#         # So the physical position is (u0_u + s_x * x, u0_v + s_y * y).
#         # But the source says "write access is restricted to grid points".
#         # This means we only write to texels that correspond to logical grid points.
#         # The fragment shader is invoked for each fragment in the quad.
#         # The quad covers the entire tile. The fragments correspond to the logical grid points in the slice.
#         # So the fragment's position in the texture is exactly the physical position of the logical grid point.
#         # So we write to (u0_u + s_x * x, u0_v + s_y * y).
#         # But s_x, s_y might not be 1. If s_x, s_y are not 1, then the logical grid points
#         # might not align with the texel centers.
#         # The source says "u_local = (s_x * x, s_y * y)".
#         # This suggests that the logical x, y are scaled to fit the physical tile.
#         # So the physical position of the logical grid point (x, y) is (u0_u + s_x * x, u0_v + s_y * y).
#         # And we write to this position.
#         # But the source also says "write access is restricted to grid points (i.e., single texels)".
#         # This implies that the physical position (u0_u + s_x * x, u0_v + s_y * y) is exactly a texel center.
#         # So s_x, s_y must be chosen such that s_x * x and s_y * y are integers for all x, y in the grid.
#         # This is a constraint on the scaling factors.
#         # For simplicity, we assume s_x = 1, s_y = 1, and the logical grid size matches the tile size.
#         # Then the physical position is (u0_u + x, u0_v + y).
#         # And we write to this position.
#         # But the source says "u_local = (s_x * x, s_y * y)".
#         # So we use s_x, s_y.
#         physical_x = u0_u + s_x * x
#         physical_y = u0_v + s_y * y
#         texture[i_tex][physical_x][physical_y] = rho_prev
```

**Parameters:**
*   $\Delta t$: The physical time step, which corresponds directly to the step size in computational (texture) space [S1].
*   $s_x, s_y$: Scaling factors that take into account the different relative sizes of texels in logical and physical memory [S1].
*   $v_{max}$: The maximum velocity of the data set. The maximum difference vector for the backward lookup is bounded by $\Delta t v_{max}$ [S1].

**Edge Cases and Pitfalls:**
*   **Trilinear Interpolation Boundaries:** Trilinear interpolation requires handling tile boundaries. A one-texel-wide border is added around each tile to avoid erroneous influence by neighboring tiles during bilinear interpolation [S1].
*   **Multiple Textures:** If the 3D domain is large, multiple 2D textures are used. The mapping $\Phi$ must correctly identify which texture to sample from for the backward lookup. The source notes that multiple 2D textures are only required in tiles close to the boundary between two physical textures because the backward lookup distance is bounded by $\Delta t v_{max}$ [S1].
*   **Write Access:** Write access is restricted to grid points (single texels) and does not need interpolation. This simplifies the write operation but requires careful mapping of logical grid points to physical texel centers [S1].

## Subtopics

### [2D Texture-Based Advection with Volumetric Illumination](19825-2d-texture-based-advection-with-volumetric-illumination.md)
This method provides an interactive technique for the dense texture-based visualization of unsteady 3D flow by achieving efficient 3D texture advection through the mapping of logical 3D memory to physical 2D memory. It sets itself apart from its sibling [2D Texture-Based Advection with Tile Reordering](19839-2d-texture-based-advection-with-tile-reordering.md) by utilizing a generic volumetric importance function to modify the transfer function, thereby addressing clutter and occlusion by emphasizing important flow regions and reducing visual complexity in less interesting parts. Unlike the parent [GPU-based 3D Texture Advection with Logical-to-Physical Mapping](19774-gpu-based-3d-texture-advection-with-logical-to-physical-mapp.md) which focuses on the general mapping strategy, this specific implementation integrates volumetric illumination and particle injection schemes directly into the 2D texture pipeline to optimize for real-time performance on specific hardware architectures.

### [2D Texture-Based Advection with Tile Reordering](19839-2d-texture-based-advection-with-tile-reordering.md)
A method for advecting particle traces in 3D unsteady flow using a single 2D texture on the GPU, avoiding the need for multiple copies of volume data or 3D textures by reordering the stacking direction dynamically. It sets itself apart from its parent [GPU-based 3D Texture Advection with Logical-to-Physical Mapping](19774-gpu-based-3d-texture-advection-with-logical-to-physical-mapp.md) by dynamically changing the stacking axis during advection to maintain visual quality, whereas the parent uses a fixed logical-to-physical mapping. It differs from its sibling [2D Texture-Based Advection with Volumetric Illumination](19825-2d-texture-based-advection-with-volumetric-illumination.md) by focusing specifically on the advection mechanism and memory layout rather than the illumination integration.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Fast read and write access to physical memory, independent of GPU architecture, because it uses 2D textures instead of 3D textures. 2D textures support efficient bilinear resampling and extremely fast write access via render-to-texture functionality [S1].
*   **Limitations:** Requires careful management of the logical-to-physical mapping, especially when multiple 2D textures are needed. Trilinear interpolation is more complex to implement than bilinear interpolation [S1].
*   **When to Use:** When visualizing unsteady 3D flow on GPUs where 3D texture updates are slow or unsupported.
*   **When Not to Use:** When the GPU supports fast 3D texture updates, or when the flow field is steady and simpler advection methods can be used.

## Background
This method was presented by Daniel Weiskopf, Tobias Schafhitzel, and Thomas Ertl from the Institute of Visualization and Interactive Systems, University of Stuttgart, in the paper "Real-Time Advection and Volumetric Illumination for the Visualization of 3D Unsteady Flow" [S1]. The paper was published in 2005. The method addresses the computational efficiency of 3D texture advection on GPUs by mapping logical 3D grid structures to physical 2D textures [S1].

## Sources
[S1] Real-Time Advection and Volumetric Illumination for the Visualization of 3D Unsteady Flow, https://doi.org/10.2312/vissym/eurovis05/013-020, fetched 2026-10-30
