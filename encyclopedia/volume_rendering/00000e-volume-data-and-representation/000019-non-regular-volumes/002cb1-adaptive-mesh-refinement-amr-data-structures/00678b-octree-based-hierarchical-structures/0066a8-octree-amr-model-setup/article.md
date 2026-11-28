# Octree AMR Model Setup

Also called: AMR Decision Function Setup, Adaptive Octree Model Setup

## Overview
Octree AMR Model Setup configures a volume rendering or radiative transfer model to use an adaptive octree grid structure instead of a regular Cartesian grid, allowing for higher resolution in regions of interest while coarsening elsewhere [S1]. It differs from regular grid models by introducing a stochastic decision function that determines cell refinement based on local physical properties, and from sibling construction methods by focusing on the initialization of the refinement criteria and the associated model function signatures required for grid building [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   `x`, `y`, `z`: NumPy ndarrays of cell center coordinates. These are passed separately to model functions to simplify calling them during grid building when refinement criteria depend on physical structure [S1].
    *   `dx`, `dy`, `dz`: Cell sizes (implied by the decision function signature) [S1].
    *   `model`: An object providing methods to query physical properties (e.g., `getGasDensity`) [S1].
    *   `ppar`: A dictionary containing model parameters, including `threshold` and `nsample` [S1].
    *   `grid`: An optional existing `radmc3dOctree` or `radmc3dGrid` instance to avoid re-reading the spatial grid from file [S1].
*   **Outputs:**
    *   A boolean array indicating which cells should be refined (`True`) and which should not (`False`) [S1].
    *   A configured model structure ready for data reading and visualization [S1].

**Data Structures**
*   **Decision Function Signature:** The decision function must adhere to a fixed signature:
    ```python
    def decisionFunction(x=None, y=None, z=None, dx=None, dy=None, dz=None, model=None, ppar=None, **kwargs):
    ```
    Any extra keyword arguments (like `nsample` and `threshold`) are passed via `**kwargs` [S1].
*   **Model Function Signature:** For octree AMR compatible models, model functions (e.g., `getGasDensity`) accept three additional arguments compared to regular grids:
    ```python
    def func(x=None, y=None, z=None, grid=None, ppar=None):
    ```
    The `x`, `y`, `z` arrays contain cell center coordinates [S1].

**Numbered Steps**
1.  **Define Refinement Criteria:** Implement a decision function that receives cell centers and sizes. The function samples physical properties (e.g., gas density $\rho$) at random points within each cell [S1].
2.  **Sample Density:** For each cell, generate random offsets within the cell boundaries. The offsets are calculated as:
    $$ \text{offset} = (\text{random\_sample} - 0.5) \times \text{size} \times 4.0 $$
    where `random_sample` is a uniform random number in $[0, 1)$ [S1].
3.  **Evaluate Refinement Condition:** Calculate the relative density variation for each cell using the samples $\rho_i$:
    $$ \frac{\max{\rho_{\rm i}} - \min{\rho_\rm{i}}}{\max{\rho_{\rm i}}} > \text{threshold} $$
    If the condition is met, the cell is marked for refinement [S1].
4.  **Limit Refinement Depth:** Apply a maximum refinement level limit (`levelMaxLimit`) to prevent excessive depth [S1].
5.  **Read Data:** Use `analyze.readData` with `octree=True`. To optimize performance, pass an existing grid instance via the `grid` argument to avoid slow file-based tree traversal [S1].
6.  **Visualize:** Use `plotSlice2D` with `interpolateOctree` (nearest neighbor) to display 2D slices. Use `showgrid=True` and `gridalpha=0.1` to display cell boundaries [S1].

**Formulas**
*   **Refinement Condition:**
    $$ \frac{\max{\rho_{\rm i}} - \min{\rho_\rm{i}}}{\max{\rho_{\rm i}}} > \text{threshold} $$
    where $\rho_{\rm i}$ are gas density samples at random points within the cell [S1].
*   **Random Offset:**
    $$ \text{offset} = (\text{random\_sample} - 0.5) \times \text{cell\_size} \times 4.0 $$
    This ensures sampling points are distributed within the cell volume [S1].

**Parameters**
*   `nsample`: Number of random locations within the cell used to estimate density structure. Default example value: 30 [S1].
*   `threshold`: Lower limit for the relative density variation above which a cell is refined. Default example value: 0.9 [S1].
*   `levelMaxLimit`: Parameter to limit the maximum depth of the grid (highest refinement level). Can be set in `problem_params.inp` or as a keyword argument [S1].
*   `nproc`: Number of parallel processes used for interpolation in `plotSlice2D`. Example value: 3 [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def decisionFunction(x=None, y=None, z=None, dx=None, dy=None, dz=None, model=None, ppar=None, **kwargs):
    """
    Determines which octree cells should be refined based on density variation.
    
    Parameters:
    x, y, z: Arrays of cell center coordinates.
    dx, dy, dz: Cell sizes.
    model: Object with getGasDensity method.
    ppar: Dictionary with 'threshold'.
    kwargs: Contains 'nsample'.
    
    Returns:
    Boolean array indicating refinement decision.
    """
    ncell = x.shape[0]
    nsample = kwargs.get('nsample', 30)
    threshold = ppar.get('threshold', 0.9)
    
    # Initialize array to store density samples
    rho = np.zeros([ncell, nsample], dtype=np.float64)
    
    # Sample density at random points within each cell
    for isample in range(nsample):
        # Generate random offsets scaled by cell size
        xoffset = (np.random.random_sample(ncell) - 0.5) * dx * 4.0
        yoffset = (np.random.random_sample(ncell) - 0.5) * dy * 4.0
        zoffset = (np.random.random_sample(ncell) - 0.5) * dz * 4.0
        
        # Query density at offset positions
        rho[:, isample] = model.getGasDensity(x + xoffset, y + yoffset, z + zoffset, ppar=ppar)
    
    # Calculate max and min density for each cell
    rho_max = rho.max(axis=1)
    rho_min = rho.min(axis=1)
    
    # Check refinement condition
    # Condition: (max - min) / max > threshold
    jj = ((rho_max - rho_min) / rho_max > threshold)
    
    # Initialize decision array
    decision = np.zeros(ncell, dtype=bool)
    
    # Mark cells for refinement
    if np.any(jj):
        decision[jj] = True
        
    return decision
```

**Edge Cases and Pitfalls**
*   **Stochasticity:** The refinement decision is stochastic because it uses random offsets. This causes the grid structure to change slightly between runs [S1].
*   **Reproducibility:** To ensure reproducible grid structures, use a fixed seed for the random number generator in the decision function and increase `nsample` [S1].
*   **Performance:** Reading the spatial grid from file is slow because it follows the tree node-by-node. Passing an existing grid instance to `readData` avoids this [S1].
*   **Visualization:** Simple 2D plotting cannot display irregularly spaced octree data directly. Use `plotSlice2D` with `interpolateOctree` for nearest-neighbor interpolation to a regular grid [S1].

## Subtopics

### Adaptive Octree Construction with 2:1 Balance and Surface Propagation
Focuses on the structural balancing and surface-aware refinement of the octree, ensuring smooth transitions between levels. Choose when geometric accuracy and balanced tree structure are prioritized over pure physical property-based refinement. [830-adaptive-octree-construction-with-2-1-balance-and-surface-pr.md]

### Octree-based AMR Data Splitting
Deals with the partitioning and distribution of octree data for parallel processing or storage efficiency. Choose when managing large-scale datasets requiring distributed memory or efficient I/O. [12260-octree-based-amr-data-splitting.md]

## Operating characteristics
*   **Time Complexity:** Grid building using array operations is significantly faster than reading the grid from file node-by-node [S1]. However, increasing `nsample` slows down the model setup as more density queries are performed [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Reading the spatial grid from file is slower than reading the density file or creating the grid in memory [S1]. Using `nproc=3` for interpolation in `plotSlice2D` provides parallel speedup [S1].
*   **Strengths:** Allows high resolution in regions with sharp density transitions (e.g., disk edges, gaps) while coarsening elsewhere [S1].
*   **Limitations:** Stochastic nature leads to non-reproducible grids unless seeded; higher `nsample` increases setup time [S1].
*   **When to Use:** When the physical structure (e.g., density gradients) dictates the need for adaptive resolution.
*   **When Not to Use:** When reproducible grid structures are critical without the overhead of fixed seeds and high `nsample`, or when the scene lacks significant density variations requiring refinement.

## Background
Octree AMR models were introduced in radmc3dPy to support adaptive mesh refinement for radiative transfer simulations. The method was documented in radmc3dPy v0.29 and v0.30.2, extending the model function signature to include cell center coordinates for easier integration with refinement criteria [S1].

## Sources
[S1] Octree AMR models — radmc3dPy 0.30.2 documentation, https://www.ita.uni-heidelberg.de/~dullemond/software/radmc-3d/manual_rmcpy/tutorial_octree_models.html, fetched 2026-10-30
