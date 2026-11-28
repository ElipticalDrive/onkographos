# Grid Gradient Normal Estimation

Also called: Finite Difference Heightmap Normals, Finite Difference Normals, Gradient-Based Normals, Heightmap Vertex Normal Calculation

## Overview
Grid Gradient Normal Estimation computes vertex normals by estimating the gradient of the underlying scalar field (density or height function) at grid vertices using finite differences, then interpolating these normals to mesh vertices [S1, S2, unofficial]. It differs from face-normal averaging by sampling the density function directly, which the original author considered as an alternative to averaging face normals for smooth shading if the field is differentiable [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A regular grid of scalar values (density $\rho$ or height $h$). For 3D, a 3D array `Grid[cx, cy, cz]`. For 2D heightmaps, a 2D array `H[i, j]` [S1, S2, unofficial].
- Output: Normal vectors $\mathbf{n}$ at grid vertices, which are then interpolated to mesh vertices [S1, unofficial].

**Data Structures:**
- `Grid`: A 3D array where `Grid[cx, cy, cz]` stores the scalar value at integer coordinates $(cx, cy, cz)$ [S1, unofficial].
- `Normal`: A 3D vector associated with each grid vertex [S1, unofficial].

**Steps:**
1. Iterate over each grid vertex at coordinates $(cx, cy, cz)$ [S1, unofficial].
2. Calculate the X component of the gradient by taking the difference between the value at the previous grid point in X and the next grid point in X [S1, unofficial].
3. Calculate the Y component of the gradient by taking the difference between the value at the previous grid point in Y and the next grid point in Y [S1, unofficial].
4. Calculate the Z component of the gradient by taking the difference between the value at the previous grid point in Z and the next grid point in Z [S1, unofficial].
5. Normalize the resulting gradient vector to obtain the grid vertex normal [S1, unofficial].
6. Interpolate the grid vertex normals to the positions of the mesh vertices (which lie on the edges of the grid cells) to obtain the final vertex normals [S1, unofficial].

**Formulas:**
For a 3D grid vertex at $(cx, cy, cz)$, the unnormalized gradient $\mathbf{g}$ is calculated as [S1, unofficial]:
$$
g_x = \text{Grid}[cx-1, cy, cz].\text{Value} - \text{Grid}[cx+1, cy, cz].\text{Value}
$$
$$
g_y = \text{Grid}[cx, cy-1, cz].\text{Value} - \text{Grid}[cx, cy+1, cz].\text{Value}
$$
$$
g_z = \text{Grid}[cx, cy, cz-1].\text{Value} - \text{Grid}[cx, cy, cz+1].\text{Value}
$$
The normal $\mathbf{n}$ is:
$$
\mathbf{n} = \text{normalize}(\mathbf{g})
$$

For a 2D heightmap vertex at $(i, j)$ with height values $h$, the normal is calculated using neighboring heights [S2, unofficial]:
$$
\mathbf{n}_{i,j} = \text{normalize}(\mathbf{g})
$$
Where the gradient vector $\mathbf{g}$ is constructed from the height differences and grid spacing $\Delta z$ [G]:
$$
\mathbf{g} = (h_{i-1,j} - h_{i+1,j},\ h_{i,j-1} - h_{i,j+1},\ 2 \cdot \Delta z)
$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_grid_normals_3d(grid, width, height, depth):
    """
    Computes normals at grid vertices using central finite differences.
    grid: 3D array of scalar values indexed as grid[x][y][z].
    width, height, depth: dimensions of the grid.
    Returns: 3D array of normal vectors.
    """
    normals = [[[None for _ in range(depth)] for _ in range(height)] for _ in range(width)]
    
    for cx in range(width):
        for cy in range(height):
            for cz in range(depth):
                gx = 0.0
                gy = 0.0
                gz = 0.0
                
                # X component
                if cx > 0 and cx < width - 1:
                    gx = grid[cx-1][cy][cz] - grid[cx+1][cy][cz]
                
                # Y component
                if cy > 0 and cy < height - 1:
                    gy = grid[cx][cy-1][cz] - grid[cx][cy+1][cz]
                
                # Z component
                if cz > 0 and cz < depth - 1:
                    gz = grid[cx][cy][cz-1] - grid[cx][cy][cz+1]
                
                # Normalize the gradient vector
                length = (gx*gx + gy*gy + gz*gz)**0.5
                if length > 0:
                    normals[cx][cy][cz] = (gx/length, gy/length, gz/length)
                else:
                    normals[cx][cy][cz] = (0.0, 0.0, 1.0) # Default normal [G]
                    
    return normals

def interpolate_normals_to_mesh(grid_normals, mesh_vertices, grid_spacing):
    """
    Interpolates grid vertex normals to mesh vertices.
    mesh_vertices: List of vertex positions (x, y, z) in world space.
    grid_spacing: Distance between grid points.
    Returns: List of normal vectors for mesh vertices.
    """
    mesh_normals = []
    for v in mesh_vertices:
        # Convert world position to grid coordinates
        gx = v.x / grid_spacing
        gy = v.y / grid_spacing
        gz = v.z / grid_spacing
        
        # Get integer and fractional parts
        ix = int(gx)
        iy = int(gy)
        iz = int(gz)
        fx = gx - ix
        fy = gy - iy
        fz = gz - iz
        
        # Clamp indices to valid range [G]
        ix = max(0, min(ix, len(grid_normals) - 1))
        iy = max(0, min(iy, len(grid_normals[0]) - 1))
        iz = max(0, min(iz, len(grid_normals[0][0]) - 1))
        
        # Retrieve 8 corner normals
        n000 = grid_normals[ix][iy][iz]
        n100 = grid_normals[ix+1][iy][iz]
        n010 = grid_normals[ix][iy+1][iz]
        n110 = grid_normals[ix+1][iy+1][iz]
        n001 = grid_normals[ix][iy][iz+1]
        n101 = grid_normals[ix+1][iy][iz+1]
        n011 = grid_normals[ix][iy+1][iz+1]
        n111 = grid_normals[ix+1][iy+1][iz+1]
        
        # Trilinear interpolation of normals
        nx = (1-fx)*(1-fy)*(1-fz)*n000[0] + fx*(1-fy)*(1-fz)*n100[0] + \
             (1-fx)*fy*(1-fz)*n010[0] + fx*fy*(1-fz)*n110[0] + \
             (1-fx)*(1-fy)*fz*n001[0] + fx*(1-fy)*fz*n101[0] + \
             (1-fx)*fy*fz*n011[0] + fx*fy*fz*n111[0]
             
        ny = (1-fx)*(1-fy)*(1-fz)*n000[1] + fx*(1-fy)*(1-fz)*n100[1] + \
             (1-fx)*fy*(1-fz)*n010[1] + fx*fy*(1-fz)*n110[1] + \
             (1-fx)*(1-fy)*fz*n001[1] + fx*(1-fy)*fz*n101[1] + \
             (1-fx)*fy*fz*n011[1] + fx*fy*fz*n111[1]
             
        nz = (1-fx)*(1-fy)*(1-fz)*n000[2] + fx*(1-fy)*(1-fz)*n100[2] + \
             (1-fx)*fy*(1-fz)*n010[2] + fx*fy*(1-fz)*n110[2] + \
             (1-fx)*(1-fy)*fz*n001[2] + fx*(1-fy)*fz*n101[2] + \
             (1-fx)*fy*fz*n011[2] + fx*fy*fz*n111[2]
             
        length = (nx*nx + ny*ny + nz*nz)**0.5
        if length > 0:
            mesh_normals.append((nx/length, ny/length, nz/length))
        else:
            mesh_normals.append((0.0, 0.0, 1.0)) # Default normal [G]
            
    return mesh_normals
```

**Edge Cases and Pitfalls:**
- **Boundaries:** The central difference requires neighbors on both sides. At grid boundaries, values must be handled (e.g., clamping, mirroring, or assuming zero gradient) [G].
- **Degenerate Input:** If the gradient vector is zero (e.g., flat region), normalization fails. A default normal should be returned [G].

## Subtopics

### Finite Difference Heightmap Normals
A 2D-specific variant for terrain meshes where normals are derived from height values in a 2D array [S2, unofficial]. Use when working with heightmaps.

### Gradient-Based Normals
A general term for any normal estimation using field gradients [S1, unofficial]. Use when the scalar field is available and differentiable [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources [G].
- **Memory Complexity:** Not given in the sources [G].
- **Performance:** Not given in the sources [G].
- **Strengths:** Provides smooth shading; directly uses the underlying field data [S1, S2, unofficial].
- **Limitations:** Requires access to the scalar field; sensitive to noise; boundary handling is non-trivial [G].
- **When to use:** When the scalar field is available and smooth shading is desired [S1, S2, unofficial].
- **When not to use:** When only mesh topology is available (use face averaging); when the field is noisy without preprocessing [G].

## Background
This method is used in terrain synthesis to compute smooth vertex normals for heightmaps [S2, unofficial]. It was discussed in the context of Marching Cubes algorithms as an alternative to averaging face normals for isosurfaces [S1, unofficial].

## Sources
[S1] Marching Cubes Vertex Normals, https://gamedev.stackexchange.com/questions/97707/marching-cubes-vertex-normals, fetched 2026-10-26 (unofficial)
[S2] Terrain synthesis using noise, https://trepo.tuni.fi/handle/10024/101043, fetched 2026-10-26 (unofficial)
