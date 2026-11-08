# Mandelbulb Isosurface Extraction via Marching Cubes

Also called: Mandelbulb Mesh Generation

## Overview
This method generates a 3D triangular mesh of the Mandelbulb fractal by evaluating a scalar field function on a 3D voxel grid and extracting the isosurface using the marching cubes algorithm [S1, unofficial]. It differs from ray-traced Mandelbulb rendering by producing a static, manipulable polygonal mesh suitable for standard graphics pipelines, rather than a per-pixel ray-marched image [S1, unofficial].

## How it works
**Inputs:**
- Grid resolution $N \times N \times N$ (not specified in sources, but implied by grid generation).
- Grid domain: Cube $[-1.2, 1.2] \times [-1.2, 1.2] \times [-1.2, 1.2]$ [S1, unofficial].
- Isosurface level: $0.005$ [S1, unofficial].
- Iteration count: 24 [S1, unofficial].

**Outputs:**
- A triangular mesh representing the Mandelbulb surface.

**Data Structures:**
- A 3D array of scalar values (floats/doubles) of size $N \times N \times N$, where each cell stores the result of `mandelbulb(x, y, z)` for the corresponding grid point.
- Standard marching cubes lookup tables for vertex interpolation and triangle topology.

**Scalar Field Function:**
The scalar field $V(x_0, y_0, z_0)$ is computed by iterating a polynomial mapping in Cartesian coordinates. The function returns a positive value if the point escapes the bounded region, and 0.0 if it remains bounded within the iteration limit.

1. Initialize $x = x_0$, $y = y_0$, $z = z_0$, and derivative accumulator $dr = 1.0$ [S1, unofficial].
2. For $i$ from 0 to 23:
   a. Calculate squared radius: $r^2 = x^2 + y^2 + z^2$ [S1, unofficial].
   b. If $r^2 > 4$:
      i. Return escape time value: $V = 0.25 \cdot \sqrt{r^2} \cdot \log(r^2) / dr$ [S1, unofficial].
   c. Calculate $r^6 = (r^2)^3$ [S1, unofficial].
   d. Update derivative: $dr = 8.0 \cdot r^6 \cdot \sqrt{r^2} \cdot dr + 1.0$ [S1, unofficial].
   e. Calculate angles:
      - $\theta = 8.0 \cdot \text{atan2}(\sqrt{x^2 + y^2}, z)$ [S1, unofficial].
      - $\phi = 8.0 \cdot \text{atan2}(y, x)$ [S1, unofficial].
   f. Calculate $r^8 = r^6 \cdot r^2$ [S1, unofficial].
   g. Update position:
      - $x = r^8 \cos(\phi) \sin(\theta) + x_0$ [S1, unofficial].
      - $y = r^8 \sin(\phi) \sin(\theta) + y_0$ [S1, unofficial].
      - $z = r^8 \cos(\theta) + z_0$ [S1, unofficial].
3. If the loop completes without escaping, return $0.0$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def mandelbulb_scalar_field(x0: float, y0: float, z0: float) -> float:
    """
    Computes the scalar field value for the Mandelbulb fractal at point (x0, y0, z0).
    Returns a positive value if the point escapes, 0.0 if it remains bounded.
    """
    x = x0
    y = y0
    z = z0
    dr = 1.0
    
    # Fixed iteration count as per source
    for _ in range(24):
        r2 = x * x + y * y + z * z
        
        # Escape condition
        if r2 > 4.0:
            return 0.25 * math.sqrt(r2) * math.log(r2) / dr
        
        # Derivative and coordinate update
        r6 = r2 * r2 * r2
        dr = 8.0 * r6 * math.sqrt(r2) * dr + 1.0
        
        theta = 8.0 * math.atan2(math.sqrt(x * x + y * y), z)
        phi = 8.0 * math.atan2(y, x)
        
        r8 = r6 * r2
        
        x = r8 * math.cos(phi) * math.sin(theta) + x0
        y = r8 * math.sin(phi) * math.sin(theta) + y0
        z = r8 * math.cos(theta) + z0
        
    return 0.0

def generate_mandelbulb_mesh(grid_size: int, domain_min: float = -1.2, domain_max: float = 1.2, isolevel: float = 0.005):
    """
    Generates a voxel grid and extracts the isosurface using marching cubes.
    Note: The marching cubes implementation itself is assumed to be available from standard libraries or previous articles.
    """
    # Calculate step size
    step = (domain_max - domain_min) / grid_size
    
    # Initialize scalar field grid
    # Dimensions: (grid_size + 1) x (grid_size + 1) x (grid_size + 1) for vertices
    # Or grid_size x grid_size x grid_size for cell centers, depending on MC implementation.
    # Standard MC operates on grid vertices.
    width = grid_size + 1
    scalar_field = [[[0.0 for _ in range(width)] for _ in range(width)] for _ in range(width)]
    
    for i in range(width):
        for j in range(width):
            for k in range(width):
                x = domain_min + i * step
                y = domain_min + j * step
                z = domain_min + k * step
                scalar_field[i][j][k] = mandelbulb_scalar_field(x, y, z)
                
    # Extract isosurface
    # The specific marching cubes algorithm details are assumed to be defined in the parent or sibling articles.
    # Here we assume a function `marching_cubes(scalar_field, isolevel)` returns a list of triangles.
    triangles = marching_cubes(scalar_field, isolevel)
    
    return triangles
```

**Edge Cases and Pitfalls:**
- **Precision:** The derivative calculation involves high powers ($r^6, r^8$). For points close to the boundary or deep inside the fractal, floating-point precision may affect the smoothness of the isosurface. Double precision is recommended [S1, unofficial].
- **Isosurface Level:** The value $0.005$ is critical. A different level will change the thickness and detail of the generated mesh [S1, unofficial].
- **Grid Resolution:** The source does not specify the grid resolution $N$. Higher resolutions are required to capture the fine details of the Mandelbulb, but increase memory and computation linearly with $N^3$.

## Operating characteristics
- **Time Complexity:** $O(N^3)$ for scalar field evaluation, where $N$ is the grid resolution. Each voxel requires 24 iterations of the Mandelbulb function.
- **Memory Complexity:** $O(N^3)$ to store the scalar field grid.
- **Performance:** The method is CPU-bound due to the iterative nature of the scalar field function. It is significantly slower than ray-tracing for visualization but produces a static mesh.
- **Strengths:** Produces a standard triangular mesh compatible with any 3D engine or mesh processing tool.
- **Limitations:** High memory usage for high-resolution grids. The mesh may be noisy or incomplete if the grid resolution is too low to capture the fractal's fine structures. The isosurface level must be carefully chosen to capture the desired detail.

## Background
The Mandelbulb is a 3D fractal discovered by Daniel Shapero and popularized by Paul Nylander and others. The specific scalar field function used here, which includes a derivative calculation for smooth coloring/escape time, is a standard approach for generating Mandelbulb meshes. The method of extracting the mesh via marching cubes on a voxel grid is a common technique for implicit surfaces. The specific parameters (24 iterations, domain $[-1.2, 1.2]^3$, isolevel $0.005$) were provided by Stéphane Laurent in a Stack Overflow answer [S1, unofficial].

## Sources
[S1] How to make a beautiful mesh of the Mandelbulb?, https://stackoverflow.com/questions/75255787/how-to-make-a-beautiful-mesh-of-the-mandelbulb, fetched 2026-10-27 (unofficial)
