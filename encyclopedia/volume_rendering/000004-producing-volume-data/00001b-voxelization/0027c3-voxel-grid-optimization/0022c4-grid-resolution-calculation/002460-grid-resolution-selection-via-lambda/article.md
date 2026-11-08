# Grid Resolution Selection via Lambda

Also called: Cleary et al. Heuristic, Lambda Grid Sizing, Uniform Grid Resolution Formula

## Overview
This method determines the number of cells in each dimension of a uniform 3D grid based on the scene's bounding box volume, the number of primitives, and a scaling factor $\lambda$ [S1][S2, unofficial]. It solves the problem of selecting an appropriate grid resolution that balances build time and traversal efficiency by approximating cubically shaped cells, which minimize the expected ray tracing cost due to their optimal surface-area-to-volume ratio [S1].

## How it works
**Inputs:**
*   $N$: Number of primitives (triangles) in the scene.
*   $V$: Volume of the scene's bounding box.
*   $d_x, d_y, d_z$: Extents (diagonal lengths or bounding box dimensions) of the scene's bounding box in each dimension.
*   $\lambda$: User-defined scaling factor (grid density).

**Outputs:**
*   $N_x, N_y, N_z$: Number of cells in the grid along the x, y, and z axes, respectively.

**Data Structures:**
No complex data structures are required for the calculation itself [G]. The output integers define the dimensions of the uniform grid array to be allocated [G].

**Steps:**
1.  Compute the scaling term: $T = \frac{\lambda N}{V}$ [G].
2.  Calculate the number of cells for each dimension by scaling the cube root of $T$ with the respective bounding box extent:
    $$N_x = d_x \sqrt[3]{T}$$
    $$N_y = d_y \sqrt[3]{T}$$
    $$N_z = d_z \sqrt[3]{T}$$ [S1][S2, unofficial]
3.  Round $N_x, N_y, N_z$ to the nearest integers [G]. Standard rounding or ceiling is typically used to ensure sufficient coverage, though the sources do not specify the rounding mode explicitly [G].

**Parameters:**
*   $\lambda$: The scaling factor. The typical and recommended value is 5 [S1]. Experiments show that most scenes are insensitive to this parameter and achieve optimum performance around $\lambda \approx 5$ for both static and dynamic scenes [S1].

**Edge Cases and Pitfalls:**
*   **Zero Volume:** If $V$ is zero (degenerate bounding box), the formula is undefined. A guard clause should handle this, likely by falling back to a default resolution or skipping grid construction [G].
*   **Primitive Count:** If $N$ is zero, the resolution becomes zero. This should be handled by ensuring a minimum grid size (e.g., $1 \times 1 \times 1$) if the scene is empty but needs to be traversed [G].
*   **Precision:** The cube root operation should be performed with sufficient floating-point precision to avoid significant quantization errors in the final integer cell counts, especially for very large or small scenes [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calculate_grid_resolution(N, V, d_x, d_y, d_z, lambda_factor=5.0):
    """
    Calculate the number of cells in a uniform 3D grid.
    
    Args:
        N (int): Number of primitives (triangles).
        V (float): Volume of the scene bounding box.
        d_x (float): Extent of the bounding box in x-dimension.
        d_y (float): Extent of the bounding box in y-dimension.
        d_z (float): Extent of the bounding box in z-dimension.
        lambda_factor (float): Scaling factor (default 5.0).
        
    Returns:
        tuple: (N_x, N_y, N_z) number of cells in each dimension.
    """
    # Guard for degenerate bounding box [G]
    if V == 0:
        return (1, 1, 1)
    
    # Guard for empty scene [G]
    if N == 0:
        return (1, 1, 1)
        
    # Calculate the term inside the cube root [G]
    term = (lambda_factor * N) / V
    
    # Calculate cube root
    cube_root_term = term ** (1.0 / 3.0)
    
    # Calculate number of cells for each dimension
    N_x = d_x * cube_root_term
    N_y = d_y * cube_root_term
    N_z = d_z * cube_root_term
    
    # Round to nearest integer [G]
    N_x = round(N_x)
    N_y = round(N_y)
    N_z = round(N_z)
    
    # Ensure minimum resolution of 1 [G]
    N_x = max(1, N_x)
    N_y = max(1, N_y)
    N_z = max(1, N_z)
    
    return (N_x, N_y, N_z)
```

## Subtopics

### [Uniform Grid Construction](4876-uniform-grid-construction.md)
Uniform Grid Construction creates a spatial acceleration structure by partitioning scene objects into a regular 3D grid, allowing objects to overlap multiple cells if necessary. It builds on [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md) to determine cell dimensions, then assigns objects to cells based on their bounding boxes. It is distinguished from hierarchical methods like [Hierarchical Grid Organization with Macrocells](5025-hierarchical-grid-organization-with-macrocells.md) by its flat structure and from sparse approaches like [Coarse Grid for Large Models](5481-coarse-grid-for-large-models.md) by its dense, regular allocation.

### [Hierarchical Grid Organization with Macrocells](5025-hierarchical-grid-organization-with-macrocells.md)
This method organizes particle data into a hierarchical grid structure to accelerate ray tracing and volume rendering by enabling efficient spatial culling. It differs from flat grid approaches by introducing a coarser macrocell level that stores parameter ranges, allowing for run-time parameter range culling rather than simple empty/non-empty checks. It builds on [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md) for determining grid dimensions.

### [Coarse Grid for Large Models](5481-coarse-grid-for-large-models.md)
This method selects a significantly coarser voxel grid resolution (lower $\lambda$) for very large triangle models to minimize grid rebuild time, accepting increased ray traversal overhead to achieve better aggregate frame rates. It differs from the standard [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md) approach, which targets moderate scenes with $\lambda \approx 5$, by prioritizing build speed over traversal efficiency for models exceeding several million triangles.

### Voxel Size Calculation (Optical/Diffraction-based)
This variant assumes the size of an emission dot (voxel) is equal to the size of the laser focal point, which is an oval with two diameters [S3]. The diameter perpendicular to the laser beam is the diffraction limit, determined by the original beam width $a$, focal length $r$, and wavelength $\lambda$ as $w_f = \frac{2\lambda r}{a}$ [S3]. The diameter along the laser beam is geometrically obtained from the relationship $a : w_f = r : w_d/2$, yielding $w_d = 4\lambda \left(\frac{r}{a}\right)^2$ [S3].

## Operating characteristics
*   **Performance:** Optimum performance is reached at $\lambda \approx 5$ for nearly all tested scenes, both static and dynamic [S1]. The method is robust and insensitive to small variations in $\lambda$ [S1].
*   **Strengths:** Simple to implement; adapts to scene complexity; minimizes expected ray tracing cost by promoting cubic cells [S1].

## Background
The method originates from the work of Cleary et al. (1983) [S1], who proposed choosing the number of cells to be a multiple, $\lambda$, of the number of triangles. The heuristic is based on the principle that cubically shaped cells have the smallest surface area in relation to volume, thereby minimizing the expected ray tracing cost [S1].

## Sources
[S1] Ray Tracing Animated Scenes using Coherent Grid Traversal, https://www.sci.utah.edu/~thiago/papers/grid.pdf, fetched 2026-10-25
[S2] Two-Level Grids for Ray Tracing on GPUs, https://www.readkong.com/page/two-level-grids-for-ray-tracing-on-gpus-8829848, fetched 2026-10-25 (unofficial)
[S3] Fairy Lights in Femtoseconds: Aerial and Volumetric Graphics Rendered by Focused Femtosecond Laser Combined with Computational Holographic Fields, https://arxiv.org/abs/1506.06668v1, fetched 2026-10-26
