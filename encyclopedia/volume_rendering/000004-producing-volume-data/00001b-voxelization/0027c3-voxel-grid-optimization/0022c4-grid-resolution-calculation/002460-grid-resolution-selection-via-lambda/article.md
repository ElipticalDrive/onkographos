# Grid Resolution Selection via Lambda

Also called: Cleary et al. Heuristic, Lambda Grid Sizing, Uniform Grid Resolution Formula

## Overview
This method determines the number of cells in each dimension of a uniform or hierarchical voxel grid based on the scene's bounding box volume and the number of geometric primitives (triangles or particles) [S1][S3, unofficial]. It solves the problem of balancing grid granularity against memory overhead and traversal cost by ensuring the cell count scales with scene complexity [S1]. It is distinguished from adaptive methods like [Adaptive Voxelization](link) by its fixed, uniform cell size, and from [Grid Resolution Calculation](2908-grid-resolution-calculation.md) by its explicit dependence on the primitive count $N$ and the scaling factor $\lambda$, rather than solely on aspect ratios and a target Y-axis resolution.

## How it works
**Inputs:**
- $N$: Number of primitives (triangles or particles) in the scene.
- $V$: Volume of the scene's bounding box.
- $d_x, d_y, d_z$: Extents (diagonal lengths or box dimensions) of the bounding box along the X, Y, and Z axes.
- $\lambda$: User-defined scaling factor (grid density).

**Outputs:**
- $N_x, N_y, N_z$ (or $R_x, R_y, R_z$): The number of grid cells in each dimension.

**Data Structures:**
The method computes scalar integers for grid dimensions. No complex data structures are required for the calculation itself, though the resulting dimensions are used to allocate a 3D array of lists or linked lists for object storage [S1].

**Steps:**
1. Calculate the volume $V$ of the scene bounding box: $V = d_x \cdot d_y \cdot d_z$.
2. Compute the term $T = \frac{\lambda N}{V}$.
3. Determine the grid resolution for each axis using the chosen formula variant (see Formulas below).
4. Round the resulting floating-point values to the nearest integer to obtain the final cell counts $N_x, N_y, N_z$.

**Formulas:**
There are two primary variants found in the literature for calculating the resolution. The sources do not explicitly reconcile the difference between the square root and cube root formulations, though the cube root is theoretically motivated by volume scaling [S1][S3, unofficial].

*Variant A (Cube Root, cited by Cleary et al. and Wald et al.):*
$$N_x = d_x \sqrt[3]{\frac{\lambda N}{V}}, \quad N_y = d_y \sqrt[3]{\frac{\lambda N}{V}}, \quad N_z = d_z \sqrt[3]{\frac{\lambda N}{V}}$$
Where $d_x, d_y, d_z$ are the diagonal lengths or extents of the grid in each dimension [S1][S3, unofficial].

*Variant B (Square Root, cited by Ize et al.):*
$$R_x = d_x \sqrt{\frac{\lambda N}{V}}, \quad R_y = d_y \sqrt{\frac{\lambda N}{V}}, \quad R_z = d_z \sqrt{\frac{\lambda N}{V}}$$
Where $d_x, d_y, d_z$ are the extents of the box in all dimensions [S2, unofficial].

**Parameters:**
- $\lambda$ (Lambda): A scaling factor relating the number of cells to the number of primitives.
    - For triangle-based ray tracing (CGT): The typical value is $\lambda \approx 5$. Most scenes are insensitive to this parameter and achieve best performance around $\lambda = 5$ [S1].
    - For particle-based simulation data: Performance varies widely with $\lambda$. The optimal value is found to be $\lambda = 1$ [S5].

**Pitfalls:**
- **Hardware Assumptions:** Optimal grid densities derived for CPU ray tracing (e.g., by Ize et al.) may be sub-optimal for GPU ray tracing. This is because CPU analyses often assume fast atomic operations and compute-limited traversal, which may not hold for specific GPU architectures [S2, unofficial].
- **Primitive Approximation:** Treating geometric primitives as points for the purpose of grid density calculation can severely limit quality for other acceleration structures like BVHs [S2, unofficial].
- **Precision:** The result of the root calculation must be rounded to an integer. Using floor or ceil arbitrarily can lead to sub-optimal grid densities.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_grid_resolution(num_primitives, bbox_extents, lambda_factor, use_square_root=False):
    """
    Calculate grid resolution using the Lambda heuristic.
    
    Args:
        num_primitives (int): N, the number of triangles or particles.
        bbox_extents (tuple): (dx, dy, dz), the extents of the bounding box.
        lambda_factor (float): Lambda, the scaling factor.
        use_square_root (bool): If True, use the square root formula (Variant B); 
                                otherwise use the cube root formula (Variant A).
    
    Returns:
        tuple: (Nx, Ny, Nz), the number of cells in each dimension.
    """
    dx, dy, dz = bbox_extents
    
    # Calculate volume
    volume = dx * dy * dz
    
    if volume == 0:
        # Edge case: Zero volume bounding box
        return (1, 1, 1)
        
    # Calculate the scaling term
    term = (lambda_factor * num_primitives) / volume
    
    if use_square_root:
        # Variant B: Square root formula
        scale_x = math.sqrt(term)
        scale_y = math.sqrt(term)
        scale_z = math.sqrt(term)
    else:
        # Variant A: Cube root formula (Standard)
        scale_x = math.pow(term, 1.0/3.0)
        scale_y = math.pow(term, 1.0/3.0)
        scale_z = math.pow(term, 1.0/3.0)
        
    # Calculate resolutions
    Nx = round(dx * scale_x)
    Ny = round(dy * scale_y)
    Nz = round(dz * scale_z)
    
    # Ensure minimum resolution of 1
    Nx = max(1, Nx)
    Ny = max(1, Ny)
    Nz = max(1, Nz)
    
    return (Nx, Ny, Nz)
```

## Subtopics

### [Uniform Grid Construction](4876-uniform-grid-construction.md)
Uniform Grid Construction creates a spatial acceleration structure by partitioning scene objects into a regular 3D grid, allowing objects to overlap multiple cells if necessary. It builds on [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md) to determine cell dimensions, then assigns objects to cells based on their bounding boxes. It is distinguished from hierarchical methods like [Hierarchical Grid Organization with Macrocells](5025-hierarchical-grid-organization-with-macrocells.md) by its flat structure and from sparse approaches like [Coarse Grid for Large Models](5481-coarse-grid-for-large-models.md) by its dense, regular allocation.

### [Hierarchical Grid Organization with Macrocells](5025-hierarchical-grid-organization-with-macrocells.md)
This method organizes volumetric primitives (such as particles or spheres) into a two-level hierarchical grid structure to accelerate ray traversal and culling. It builds on [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md) to determine the finest grid resolution, then imposes a coarser macrocell level on top. It differs from [Uniform Grid Construction](4876-uniform-grid-construction.md) by adding hierarchical culling and from [Coarse Grid for Large Models](5481-coarse-grid-for-large-models.md) by using a specific two-level macrocell structure with parameter-range culling rather than just a static coarse grid.

### [Coarse Grid for Large Models](5481-coarse-grid-for-large-models.md)
This method selects a significantly coarser voxel grid resolution (lower $\lambda$) for very large triangle models to minimize grid rebuild time, accepting increased ray traversal overhead to achieve better aggregate frame rates. It differs from the standard [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md) approach, which targets moderate scenes with $\lambda \approx 5$, by prioritizing build speed over traversal efficiency for models exceeding several million triangles.

### Voxel Size Calculation (Optical/Diffraction-based)
This variant applies the lambda concept to optical diffraction limits rather than geometric acceleration structures. It assumes the size of an emission dot (voxel) is equal to the size of the laser focal point, which is an oval with two diameters [S4].
- **Perpendicular Diameter ($w_f$):** Determined by the diffraction limit: $w_f = \frac{2\lambda r}{a}$, where $\lambda$ is the wavelength, $r$ is the focal length, and $a$ is the original beam width [S4].
- **Parallel Diameter ($w_d$):** Derived from the geometric relationship $a : w_f = r : w_d/2$: $w_d = 4\lambda \left(\frac{r}{a}\right)^2$ [S4].

### Grid Resolution Parameter Tuning
In modified coherent grid traversal algorithms for particle-based simulation data, the parameter $\lambda$ behaves differently than in the original CGT algorithm. While the original CGT was largely insensitive to $\lambda$, the modified algorithm's performance varies widely with different values of $\lambda$ due to the extremely large number of particles in test datasets. The optimal value for these specific datasets is $\lambda = 1$ [S5].

## Operating characteristics
- **Time Complexity:** The calculation itself is $O(1)$ arithmetic operations. However, the choice of $\lambda$ directly impacts the cost of grid construction ($O(N)$ to insert primitives) and ray traversal ($O(\text{cells visited})$).
- **Memory Complexity:** The grid memory usage is proportional to the product $N_x \cdot N_y \cdot N_z$. Since $N_x, N_y, N_z \propto \sqrt[3]{\lambda N}$, the total cell count scales linearly with $N$ and $\lambda$.
- **Performance:** For triangle-based ray tracing, optimum performance is reached at approximately $\lambda \approx 5$ for nearly all tested scenes, both static and dynamic [S1]. Cubically shaped cells are chosen because they have the smallest surface area in relation to volume, minimizing a grid's expected ray tracing cost [S1].
- **Strengths:** Simple to implement; robust for a wide range of scenes; provides a good baseline for uniform grids.
- **Limitations:** Insensitive tuning in some contexts ($\lambda \approx 5$) can be a double-edged sword if the scene distribution is highly non-uniform; GPU-specific optimizations may require different $\lambda$ values than CPU-based analyses suggest [S2, unofficial].

## Background
The heuristic was introduced by Cleary et al. [CWVB83] and has been widely adopted in ray tracing literature, including by Wald et al. [WIK∗ 06] for coherent grid traversal [S2, unofficial]. It remains a standard method for initializing uniform grid acceleration structures in both CPU and GPU ray tracers [S1][S2, unofficial].

## Sources
[S1] PDF Ray Tracing Animated Scenes using Coherent Grid Traversal, https://www.sci.utah.edu/~thiago/papers/grid.pdf, fetched 2026-10-25
[S2] PDF Two-Level Grids for Ray Tracing on GPUs - UVic.ca, https://webhome.cs.uvic.ca/~blob/downloads/tyler/paper1227_final.pdf, fetched 2026-10-25 (unofficial)
[S3] Two-Level Grids for Ray Tracing on GPUs - ReadkonG.com, https://www.readkong.com/page/two-level-grids-for-ray-tracing-on-gpus-8829848, fetched 2026-10-25 (unofficial)
[S4] Fairy Lights in Femtoseconds: Aerial and Volumetric Graphics Rendered by Focused Femtosecond Laser Combined with Computational Holographic Fields, https://arxiv.org/abs/1506.06668v1, fetched 2026-10-26
[S5] A coherent grid traversal approach to visualizing particle-based simulation data, https://ieeexplore.ieee.org/abstract/document/4293019/, fetched 2026-10-25
