# Point Engine for Scalar Fields
Also called: Simplified Marching Cubes for points, rgridb point generation
## Overview
A method to estimate foot points on an isosurface of a scalar field using a simplified version of Marching Cubes that finds only vertices without computing triangles, avoiding the expense of iterative refinement. It sets itself apart from its parent [Marching Cubes Surface Point Extraction](6417-marching-cubes-surface-point-extraction.md) by strictly limiting the search to three specific edges per voxel and storing all resulting points in a single flat list, thereby eliminating the need for triangle connectivity, table lookups, or shared vertex bookkeeping.
## How it works
**Inputs and outputs:**
- Input: A scalar field defined on a voxel grid.
- Output: A single list of points, where each point consists of a 3D position and an interpolated gradient vector.

**Data structures:**
- **Point List:** A dynamic array or contiguous buffer storing `Point` structures.
- **Point Structure:** Contains:
  - `position`: `vec3` (float32 x, y, z).
  - `gradient`: `vec3` (float32 x, y, z).

**Steps:**
1. Iterate through every voxel in the volume grid. Let the current voxel's center position be $p$.
2. Define the positions of three neighboring voxels relative to $p$:
   - $q = p + (1, 0, 0)$
   - $r = p + (0, 1, 0)$
   - $s = p + (0, 0, 1)$
   [G] Ensure boundary checks are performed to verify that $q, r, s$ are within the valid volume bounds before accessing their scalar values.
3. For each neighbor $n \in \{q, r, s\}$:
   a. Retrieve the scalar values $V_p$ at $p$ and $V_n$ at $n$.
   b. Check if the edge connecting $p$ and $n$ intersects the isosurface. This is true if $V_p$ and $V_n$ lie on different sides of the isovalue $I$ (i.e., $(V_p - I) \cdot (V_n - I) < 0$).
   c. If an intersection exists:
      i. Calculate the intersection point $x_{int}$ using linear interpolation along the edge $pn$.
      ii. Interpolate the gradient at $x_{int}$ using the gradients at $p$ and $n$ (or the scalar field's spatial derivatives if available).
      iii. Append the pair $(x_{int}, \nabla f(x_{int}))$ to the Point List.
4. Repeat for all voxels.

**Formulas:**
- Intersection point $x_{int}$ on edge between $p$ and $n$:
  $$x_{int} = p + \frac{I - V_p}{V_n - V_p} (n - p)$$
  Where $I$ is the isovalue, $V_p$ and $V_n$ are scalar values at $p$ and $n$.
- Gradient interpolation $\nabla f(x_{int})$:
  $$\nabla f(x_{int}) = (1 - t) \nabla f(p) + t \nabla f(n)$$
  Where $t = \frac{I - V_p}{V_n - V_p}$.

**Parameters:**
- Isovalue $I$: Depends on the specific scalar field thresholding requirement. Not given in the sources.
- Gradient source: The sources mention interpolating the gradient but do not specify if it is derived from finite differences of the scalar field or provided as vector data. [G] Typically, $\nabla f$ is computed via central differences or trilinear derivative evaluation of the scalar field.

**Edge cases and pitfalls:**
- **Boundary Conditions:** The algorithm assumes neighbors $q, r, s$ exist. If a voxel is on the boundary of the volume, the corresponding neighbor lookup must be skipped or handled with padding.
- **Degenerate Cases:** If $V_n = V_p$, the denominator in the interpolation formula becomes zero. This occurs if the scalar field is constant along the edge. In this case, no unique intersection point exists via linear interpolation, and the edge should be skipped.
- **No Table Lookup:** Unlike standard Marching Cubes, this method does not use the 256-case lookup table. It only considers three edges per cell, which simplifies logic but may miss intersections on the other three edges of the cube.
- **Shared Vertices:** No bookkeeping is needed for shared vertices between adjacent voxels. Each voxel independently generates points for its three edges, potentially resulting in duplicate points at shared edges if not deduplicated (though the source notes imply a single list is used without explicit deduplication mentioned, relying on the renderer to handle point primitives).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_engine_scalar_field(volume, isovalue, voxel_size=1.0):
    """
    Extracts foot points on an isosurface using a simplified Marching Cubes approach.
    
    Args:
        volume: 3D array of scalar values.
        isovalue: The threshold value for the isosurface.
        voxel_size: Physical size of a voxel (default 1.0 for normalized coordinates).
        
    Returns:
        points: List of tuples (position, gradient).
    """
    points = []
    dims = volume.shape
    
    # Iterate over all voxels
    for x in range(dims[0]):
        for y in range(dims[1]):
            for z in range(dims[2]):
                p = vec3(x * voxel_size, y * voxel_size, z * voxel_size)
                v_p = volume[x, y, z]
                
                # Define neighbors: q, r, s
                neighbors = [
                    (vec3(1, 0, 0), (x + 1, y, z)),
                    (vec3(0, 1, 0), (x, y + 1, z)),
                    (vec3(0, 0, 1), (x, y, z + 1))
                ]
                
                for offset, (nx, ny, nz) in neighbors:
                    # Check boundary
                    if nx >= dims[0] or ny >= dims[1] or nz >= dims[2]:
                        continue
                        
                    n = p + offset * voxel_size
                    v_n = volume[nx, ny, nz]
                    
                    # Check if edge intersects isosurface
                    if (v_p - isovalue) * (v_n - isovalue) < 0:
                        # Avoid division by zero
                        if abs(v_n - v_p) < 1e-9:
                            continue
                            
                        t = (isovalue - v_p) / (v_n - v_p)
                        x_int = p + t * (n - p)
                        
                        # Interpolate gradient
                        # [G] Assuming gradients are computed from scalar field via finite differences
                        grad_p = compute_gradient(volume, x, y, z)
                        grad_n = compute_gradient(volume, nx, ny, nz)
                        grad_int = (1 - t) * grad_p + t * grad_n
                        
                        points.append((x_int, grad_int))
                        
    return points

def compute_gradient(volume, x, y, z):
    """
    Computes gradient using central differences.
    """
    dims = volume.shape
    dx = 0.0
    dy = 0.0
    dz = 0.0
    
    # X derivative
    if x > 0 and x < dims[0] - 1:
        dx = (volume[x+1, y, z] - volume[x-1, y, z]) / 2.0
    elif x == 0:
        dx = volume[x+1, y, z] - volume[x, y, z]
    else:
        dx = volume[x, y, z] - volume[x-1, y, z]
        
    # Y derivative
    if y > 0 and y < dims[1] - 1:
        dy = (volume[x, y+1, z] - volume[x, y-1, z]) / 2.0
    elif y == 0:
        dy = volume[x, y+1, z] - volume[x, y, z]
    else:
        dy = volume[x, y, z] - volume[x, y-1, z]
        
    # Z derivative
    if z > 0 and z < dims[2] - 1:
        dz = (volume[x, y, z+1] - volume[x, y, z-1]) / 2.0
    elif z == 0:
        dz = volume[x, y, z+1] - volume[x, y, z]
    else:
        dz = volume[x, y, z] - volume[x, y, z-1]
        
    return vec3(dx, dy, dz)
```
## Subtopics

### Dynamic Particle Distribution
A method to distribute particles on an isosurface by projecting them onto the implicit function and moving them based on inter-particle forces scaled by the sizing field $h$. It sets itself apart by using a physics-based relaxation approach to achieve uniform sampling density rather than simple geometric intersection. It is chosen when uniform point distribution is required for subsequent meshing or rendering.
- **Definition:** Uses a particle system framework to control particle distribution via a sizing field $h$.
- **Steps:**
  1. Initialize particles using positions from a marching cubes triangulation to ensure disconnected parts of the isosurface are seeded.
  2. Project particles onto the implicit function $F$ using Newton-Raphson gradient descent.
  3. Associate each particle with a potential function inducing inter-particle forces. Scale the distance between particles $p_i$ and $p_j$ by $\alpha_{ij}$ to determine force magnitude.
  4. Iteratively move particles to lower local energy states until the system reaches equilibrium.
- **Formulas:**
  - Scaled distance: $d_{ij} = \alpha_{ij} |\mathbf{x}_i - \mathbf{x}_j|$
  - Scaling factor: $\alpha_{ij} = \frac{\beta}{\min(h_i, h_j)}$
  - Where $h_i, h_j$ are values of the sizing field at particle positions, evaluated using trilinear reconstruction kernels.
- **Parameters:**
  - Constant $\beta = 0.5 / \cos(\pi/6)$.
- **Source:** [S2]

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels, as each voxel is visited once and a constant number of operations (3 edges) are performed.
- **Memory Complexity:** $O(P)$ where $P$ is the number of intersected edges. In the worst case (checkerboard pattern), $P \approx 3N$.
- **Measured Performance:** The method is significantly faster than full Marching Cubes because it avoids table lookups, triangle generation, and sorting. It is also faster than iterative refinement methods because it computes foot points in a single step per edge.
- **Strengths:** Simplicity, no connectivity data required, easy parallelization (each voxel is independent), low memory overhead for bookkeeping.
- **Limitations:** May produce duplicate points at shared edges if not deduplicated. Does not provide topological information (triangles). Accuracy depends on linear interpolation; highly curved surfaces may require denser sampling.
- **When to use:** When only point cloud representation is needed, or as a pre-processing step for point-based rendering (e.g., QSplat, rgridb).
- **When not to use:** When mesh connectivity is required for standard rasterization or physics simulation.
## Background
Originated in the context of hardware-accelerated point rendering of isosurfaces, specifically for the rgridb renderer. The method was proposed as an alternative to iterative foot-point refinement, which can be as expensive as Marching Cubes. It forms the core part of the point engine utilized by rgridb, leveraging OpenGL point primitives for rendering.
## Sources
[S1] Hardware Accelerated Point Rendering of Isosurfaces, http://wscg.zcu.cz/wscg2003/Papers_2003/J61.pdf, fetched 2026-10-26 (unofficial)
[S2] Topology, Accuracy, and Quality of Isosurface Meshes Using Dynamic Particles, https://doi.org/10.1109/tvcg.2007.70604, fetched 2026-10-30
