# Trivariate Cubic Polynomial Approximation for Scalar Fields

Also called: C1 Smooth Cubic Spline Contours, Piecewise C1 Approximation, Trivariate Cubic Approximation

## Overview
This method models 3D scalar field data (e.g., CT/MRI) by approximating subcubes of a cuberille volume with trivariate cubic polynomial functions that achieve C1 continuity on shared faces [S1, unofficial]. It provides a compact analytic representation for isosurface generation, distinguishing itself from lower-order approximations by avoiding polygon shading artifacts like Mach ringing [S1, unofficial].

## How it works
**Inputs:** A set of 2D scalar slices $S_0, S_1, \dots$ with resolution $W \times H$ and spacing $\Delta x, \Delta y, \Delta z$.
**Outputs:** A set of polygons representing isosurfaces $F(x,y,z) = c$.

**Data Structures:**
*   **Grid:** A user-defined grid of vertices. The source specifies a maximum grid size of $4 \times 4$ pixels per slice to balance computation cost and detail [S1, unofficial].
*   **C3 Data:** For each grid vertex, a set of 8 derivative values (function value, 3 first derivatives, 3 second derivatives, 1 mixed third derivative) are computed [S1, unofficial].
*   **Subcube:** A volume element defined by 8 vertices (corners of a grid cell in 3D). Each vertex contributes 8 coefficients, resulting in 64 constraints [S1, unofficial].

**Formulas:**
The approximation function $F(x,y,z)$ is represented in Bernstein-Bezier form [S1, unofficial]:
$$ F(x,y,z) = \sum_{i=0}^{3} \sum_{j=0}^{3} \sum_{k=0}^{3} w_{i,j,k} B_i^3(x) B_j^3(y) B_k^3(z) $$
where $B_i^3(t) = \binom{3}{i} t^i (1 - t)^{3-i}$ [S1, unofficial].

The coefficients $w_{i,j,k}$ are determined by solving a system of 64 equations in the least-squares sense, matching the function value and derivatives at the 8 vertices of the subcube [S1, unofficial].

**Derivative Computation (C3 Data):**
Derivatives are computed using finite differences. Let $\Delta x, \Delta y, \Delta z$ be the grid spacings. For a vertex at grid index $(u,v,w)$, the derivatives are approximated as follows [S1, unofficial]:

*   **Second derivatives:**
    $$ \frac{\partial^2 f}{\partial x^2} \approx \frac{f(u+1,v,w) - f(u-1,v,w)}{2 \Delta x^2} $$
    $$ \frac{\partial^2 f}{\partial y^2} \approx \frac{f(u,v+1,w) - f(u,v-1,w)}{2 \Delta y^2} $$
*   **Mixed derivatives:**
    The source provides specific, potentially garbled, finite difference stencils for mixed derivatives involving $z$. For example, for vertex $(1,1,0)$:
    $$ \frac{\partial^2 f}{\partial x \partial y} \approx \frac{f(2,2,0) - f(0,0,1)}{2 \Delta x \cdot 2 \Delta y} $$
    $$ \frac{\partial^2 f}{\partial x \partial y \partial z} \approx \frac{f(2,2,1) - f(0,0,0)}{2 \Delta x \cdot 2 \Delta y \cdot \Delta z} $$
    Note: The source text contains inconsistencies in indices for mixed derivatives (e.g., $f(0,0,1)$ vs $f(0,0,0)$) [S1, unofficial].

**Algorithm Steps:**
1.  Read two slices $S_0$ and $S_1$ from disk [S1, unofficial].
2.  Compute C3 data (derivatives up to third order) for all pixels located at the corners of the user-defined grid from slices $S_0$ and $S_1$ [S1, unofficial].
3.  Read the next slice $S_2$ [S1, unofficial].
4.  Compute C3 data from slices $S_0, S_1, S_2$. When computing derivatives involving $\Delta z$, use the lower slice ($S_0$) instead of the current slice ($S_1$) if applicable, and replace all $\Delta z$ with $2 \Delta z$ [S1, unofficial].
5.  Fit a trivariate cubic $(3,3,3)$ function $F(x,y,z)$ to each subcube formed by the grid vertices of $S_0$ and $S_1$. This involves solving 64 equations for 64 parameters in the least-squares sense [S1, unofficial].
6.  For each subcube, compute the polygons of the isosurface corresponding to a constant $c$ such that $F(x,y,z) = c$ [S1, unofficial].
7.  Discard slice $S_0$, shift $S_1 \to S_0$, and $S_2 \to S_1$, then go to Step 3 to read a new slice [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trivariate_cubic_approximation(slices, grid_spacing_x, grid_spacing_y, grid_spacing_z, grid_size=4):
    """
    slices: List of 2D arrays representing CT/MRI slices.
    grid_spacing_x, grid_spacing_y, grid_spacing_z: Spacing between pixels/slices.
    grid_size: Size of the user-defined grid (max 4x4).
    """
    # Initialize buffer for previous two slices
    S_prev = None
    S_curr = None
    
    # Read first two slices
    if len(slices) < 2:
        return []
    
    S_prev = slices[0]
    S_curr = slices[1]
    
    isosurfaces = []
    
    while True:
        # Step 2: Compute C3 data for corners of the grid in S_prev and S_curr
        # Note: The source implies a specific grid structure. 
        # We assume grid vertices are a subset of pixels.
        vertices_prev = compute_grid_vertices(S_prev, grid_size)
        vertices_curr = compute_grid_vertices(S_curr, grid_size)
        
        # Compute C3 data (derivatives) for these vertices
        # This requires accessing neighboring pixels/slices
        c3_data_prev = compute_c3_derivatives(vertices_prev, S_prev, grid_spacing_x, grid_spacing_y, grid_spacing_z)
        c3_data_curr = compute_c3_derivatives(vertices_curr, S_curr, grid_spacing_x, grid_spacing_y, grid_spacing_z)
        
        # Step 5: Fit trivariate cubic to subcubes formed by S_prev and S_curr
        # Each subcube has 8 vertices. Each vertex has 8 C3 data points.
        # Solve 64 equations for 64 coefficients (least squares)
        subcubes = create_subcubes(vertices_prev, vertices_curr)
        
        for subcube in subcubes:
            # Collect 64 constraints (8 vertices * 8 derivatives each)
            constraints = []
            for vertex in subcube.vertices:
                constraints.extend(c3_data_prev[vertex.id] if vertex in vertices_prev else c3_data_curr[vertex.id])
            
            # Solve for Bernstein-Bezier coefficients w_{i,j,k}
            # This is a linear system A * w = b
            coefficients = solve_least_squares(constraints)
            
            # Step 6: Compute isosurface polygons for this subcube
            # F(x,y,z) = c
            # Use marching cubes or similar on the polynomial
            polygons = compute_isosurface_polygons(coefficients, subcube.bounds, constant_c)
            isosurfaces.extend(polygons)
            
        # Step 7: Slide window
        if len(slices) > 2:
            S_prev = S_curr
            S_curr = slices.pop(0) # Read next slice
        else:
            break
            
    return isosurfaces

def compute_c3_derivatives(vertices, slice_data, dx, dy, dz):
    """
    Computes derivatives for given vertices using finite differences.
    Handles the specific logic for Z-derivatives using lower slices if available.
    """
    derivatives = {}
    for v in vertices:
        # Compute partial derivatives using finite difference formulas
        # Example for d2f/dx2
        d2f_dx2 = (slice_data[v.x+1, v.y, v.z] - slice_data[v.x-1, v.y, v.z]) / (2 * dx**2)
        # ... compute other derivatives ...
        derivatives[v.id] = {
            'f': slice_data[v.x, v.y, v.z],
            'df_dx': ...,
            'df_dy': ...,
            'df_dz': ...,
            'd2f_dx2': d2f_dx2,
            # ... other derivatives
        }
    return derivatives
```

## Operating characteristics
*   **Time Complexity:** Computing C3 data for all pixels is very computation expensive [S1, unofficial]. The algorithm uses a user-defined grid (max $4 \times 4$) to reduce this cost, but this loses detailed image information since most pixels are not used as grid corners [S1, unofficial].
*   **Memory Complexity:** Requires storing C3 data for grid vertices and intermediate slices. The source mentions an implementation on iPSC/860 with 16 MB local memory per node [S1, unofficial].
*   **Strengths:** Yields a compact C1 smooth reconstruction and concise analytic representation, avoiding polygon shading artifacts like Mach ringing [S1, unofficial]. Suitable for post-volume modeling manipulations and deformations [S1, unofficial].
*   **Limitations:** High computational cost for derivative calculation. Large grids reduce computation but lose detail. The method is specific to cuberille volumes.
*   **When to use:** When C1 continuity and artifact-free isosurfaces are required, and computational resources allow for the expensive derivative calculations.
*   **When not to use:** When real-time performance is critical and the overhead of solving 64 equations per subcube is prohibitive, or when the input data does not fit the cuberille model.

## Background
This method was developed for parallel generation of polygons representing isosurfaces of volume images, specifically for medical imaging data (CT/MRI) [S1, unofficial]. It was implemented on the Intel Delta iPSC/860 system at Purdue University [S1, unofficial]. The approach differs from standard piecewise linear or trilinear approximations by using higher-order schemes to achieve C1 continuity [S1, unofficial].

## Sources
[S1] Scalar Field Modeling & Visualization on the Intel Delta, https://docs.lib.purdue.edu/cgi/viewcontent.cgi?article=2138&context=cstech, fetched 2026-10-26 (unofficial)
