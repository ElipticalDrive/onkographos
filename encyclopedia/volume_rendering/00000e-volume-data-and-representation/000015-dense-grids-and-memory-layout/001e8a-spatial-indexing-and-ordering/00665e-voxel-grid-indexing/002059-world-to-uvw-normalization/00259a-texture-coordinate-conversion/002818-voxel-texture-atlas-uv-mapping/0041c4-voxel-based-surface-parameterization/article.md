# Voxel-based Surface Parameterization

Also called: Implicit Voxel-Based Texture Coordinate Mapping, Voxelization Mapping

## Overview
Voxel-based Surface Parameterization is a method to map a 2D noise texture to a 3D surface by implicitly voxelizing the object's bounding box and projecting surface points to local voxel face coordinates [S1, book][S2, unofficial]. It transforms the world coordinate system to a system defined by one voxel, ensuring every component of every point is in $[0,1]$ [S1, book]. Unlike explicit UV mapping, it determines texture coordinates based on the surface normal and the voxel side-plane whose normal is most similar to the surface normal [S1, book]. This method supports continuous parameterization with irrelevant distortions for noise mapping and allows dynamic scale changes for automatic texture refinement [S1, book][S2, unofficial].

## How it works
**Inputs:**
- Vertex position in object/world space $v_g$ (3D vector).
- Surface normal at $v_g$, denoted $n$ (3D vector).
- Voxel edge length $l$ (scalar, controls resolution/frequency) [S2, unofficial].
- Bounding box minimum coordinate $b_{min}$ (3D vector) [S2, unofficial].

**Outputs:**
- Texture coordinates $t$ (2D vector in $[0,1]^2$) [S1, book].

**Data Structures:**
- No explicit voxel grid is created; the grid is implicit [S1, book].
- The transformation uses a $4 \times 4$ matrix $M_{voxel}$ [S2, unofficial].

**Steps:**

1.  **Voxelization Transformation**: Transform the vertex $v_g$ from object space to voxel space $v_{voxel}$ using the matrix $M_{voxel}$ [S2, unofficial].
    $$v_{voxel} = v_g \cdot M_{voxel}$$
    Where $M_{voxel}$ is defined as:
    $$
    M_{voxel} = \begin{pmatrix}
    l & 0 & 0 & -b_{min_x} \\
    0 & l & 0 & -b_{min_y} \\
    0 & 0 & l & -b_{min_z} \\
    0 & 0 & 0 & 0
    \end{pmatrix}
    $$
    Note: The sources present this as a row-vector multiplication $v_g \cdot M_{voxel}$ [S2, unofficial]. If using column vectors, the matrix would be transposed. The result $v_{voxel}$ is a 3D vector (the homogeneous coordinate is discarded or ignored for the subsequent step) [S2, unofficial].

2.  **Local Face Coordinate Calculation**: Compute the local coordinates $v_{hit}$ on the voxel's faces by taking the fractional part of $v_{voxel}$ [S2, unofficial]:
    $$v_{hit} = v_{voxel} - \lfloor v_{voxel} \rfloor$$
    Here, $\lfloor \cdot \rfloor$ denotes the component-wise floor operation. This ensures every component of $v_{hit}$ is in the range $[0, 1)$ [S1, book][S2, unofficial].

3.  **Texture Coordinate Selection**: Select the two components of $v_{hit}$ that form the plane closest to the tangential plane of the surface. This is determined by the surface normal $n$ [S1, book][S2, unofficial]. Let $n = (n_x, n_y, n_z)$ and $v_{hit} = (v_x, v_y, v_z)$. Identify the index $k \in \{x, y, z\}$ such that $n_k = \max\{|n_x|, |n_y|, |n_z|\}$ [S1, book][S2, unofficial]. The texture coordinates $t$ are the two components of $v_{hit}$ corresponding to the indices $i, j$ where $\{i, j, k\} = \{x, y, z\}$ and $i \neq j \neq k$ [S1, book][S2, unofficial]:
    $$t = (v_{hit_i}, v_{hit_j})$$
    This selects the two axes perpendicular to the surface normal's dominant axis [S1, book][S2, unofficial].

**Parameters:**
- $l$: Constant edge length of the voxels. Controls the resolution/frequency of the mapped noise texture [S2, unofficial]. Can be changed during calculation to produce different frequencies and allow automatic texture refinement when zooming [S1, book][S2, unofficial].
- $b_{min}$: The minimum coordinate of the bounding box, defining the root of the voxel space [S2, unofficial].

**Edge Cases and Pitfalls:**
- The mapping is continuous but not $C^1$-continuous [S1, book][S2, unofficial]. Discontinuities in the first derivatives are acceptable because they automatically vanish in the advection step (e.g., in LIC or reaction-diffusion) [S1, book][S2, unofficial].
- The term "voxel" is for illustration; voxels are never created explicitly [S1, book].
- Precision issues may arise if $v_{voxel}$ components are very close to integers due to floating-point errors, potentially causing jumps in $v_{hit}$. However, the sources do not specify a guard for this [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_based_surface_parameterization(v_g, n, l, b_min):
    """
    Compute texture coordinates for a surface point using voxel-based parameterization.
    
    Args:
        v_g: 3D vector (x, y, z) in object space.
        n: 3D vector (nx, ny, nz) surface normal at v_g.
        l: Scalar, voxel edge length.
        b_min: 3D vector (bx, by, bz), minimum of bounding box.
        
    Returns:
        t: 2D vector (u, v) texture coordinates in [0, 1).
    """
    # Step 1: Voxelization Transformation
    # v_voxel = v_g * M_voxel
    # M_voxel = [[l, 0, 0, -bx],
    #            [0, l, 0, -by],
    #            [0, 0, l, -bz],
    #            [0, 0, 0, 0]]
    # Using row vector multiplication:
    vx = v_g.x * l + (-b_min.x)
    vy = v_g.y * l + (-b_min.y)
    vz = v_g.z * l + (-b_min.z)
    
    # Step 2: Local Face Coordinate Calculation
    # v_hit = v_voxel - floor(v_voxel)
    hit_x = vx - math.floor(vx)
    hit_y = vy - math.floor(vy)
    hit_z = vz - math.floor(vz)
    
    # Step 3: Texture Coordinate Selection
    # Find the axis with the maximum absolute normal component
    abs_nx = abs(n.x)
    abs_ny = abs(n.y)
    abs_nz = abs(n.z)
    
    # Determine the dominant axis k
    if abs_nx >= abs_ny and abs_nx >= abs_nz:
        # k = x, so we pick y and z
        t = (hit_y, hit_z)
    elif abs_ny >= abs_nx and abs_ny >= abs_nz:
        # k = y, so we pick x and z
        t = (hit_x, hit_z)
    else:
        # k = z, so we pick x and y
        t = (hit_x, hit_y)
        
    return t
```

## Operating characteristics
- **Time Complexity**: $O(1)$ per vertex. The method involves basic arithmetic operations and comparisons [S1, book][S2, unofficial].
- **Memory Complexity**: $O(1)$ auxiliary memory. No explicit voxel grid or texture atlas storage is required for the parameterization itself [S1, book].
- **Strengths**:
  - Continuous parameterization with irrelevant distortions for noise mapping [S1, book][S2, unofficial].
  - Dynamic scale change capability allows automatic texture refinement when zooming [S1, book][S2, unofficial].
  - Simple implementation without explicit voxel creation [S1, book].
- **Limitations**:
  - Not $C^1$-continuous; discontinuities in first derivatives exist [S1, book][S2, unofficial].
  - Distortions may be visible if the noise texture is not robust to such parameterization (though sources claim distortions are irrelevant for noise) [S1, book].
- **When to use**: Suitable for image-space visualization techniques like Line Integral Convolution (LIC) or reaction-diffusion textures where seamless, scalable noise mapping is required [S1, book][S2, unofficial].
- **When not to use**: When $C^1$ continuity is strictly required for geometric accuracy or when the surface normal is undefined or unstable [S1, book][S2, unofficial].

## Background
- **Origin**: The method was introduced in the context of image-space tensor field visualization using a LIC-like method [S2, unofficial]. It was also discussed in the context of fabric-like visualization of tensor field data on arbitrary surfaces [S1, book].
- **History**: The approach was developed to provide an implicit parameterization of the surface for mapping noise textures, avoiding the need for pre-computed UV coordinates or explicit 3D textures [S2, unofficial]. It allows for dynamic refinement of the input noise to achieve different levels of detail, a feature not available in earlier approaches like Turk's [S2, unofficial].
- **Usage**: Primarily used in image-space visualization for tensor fields, specifically for applying noise textures to surfaces for LIC or reaction-diffusion [S1, book][S2, unofficial].

## Sources
[S1] Fabric-Like Visualization of Tensor Field Data on Arbitrary Surfaces in Image Space, https://doi.org/10.1007/978-3-642-27343-8_4, fetched 2026-10-27 (book)
[S2] Image-Space Tensor Field Visualization Using a LIC-like Method, https://doi.org/10.1007/978-3-642-21608-4_11, fetched 2026-10-27 (unofficial)
