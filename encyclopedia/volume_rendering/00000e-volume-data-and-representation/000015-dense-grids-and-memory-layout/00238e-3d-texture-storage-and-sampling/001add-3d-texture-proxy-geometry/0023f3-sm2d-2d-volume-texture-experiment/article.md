# Sm2D (2D Volume Texture Experiment)
Also called: 2D Volume Texture Experiment

## Overview
Sm2D is a 2-dimensional test application implemented as a 2D ray tracer to investigate the applicability of a texture-based approach for smoke volumes [S1, unofficial]. It simplifies the 3D volume rendering problem into a 1D viewport function where tetrahedra are represented by triangles [S1, unofficial]. It sets itself apart from its siblings by serving as a diagnostic tool to demonstrate the precision limits and rotational artifacts inherent in texture-based approximations compared to explicit geometric calculations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of triangles $t_k$ representing the smoke volume, a camera-aligned 2D grid $G$ of size $N_r \times N_c$ (rows $\times$ columns), and a set of viewing rays $r_j$.
*   **Output:** An absorption coefficient $\alpha(r_j)$ for each ray $r_j$, visualized as a 1D function across the viewport edge [S1, unofficial].

**Data Structures:**
*   **Grid $G$:** A 2D array of cells $c_i$. In the reference implementation, $G$ is $16 \times 16$ cells [S1, unofficial]. Each cell $c_i$ has a defined area $\text{area}(c_i)$ [S1, unofficial].
*   **Triangles:** A list of $N_t$ triangles $t_k$. Each triangle $t_k$ has a defined area $\text{area}(t_k)$ [S1, unofficial].
*   **Rays:** A list of $N_r$ rays $r_j$, entering the grid from below at equally spaced points along the viewport edge [S1, unofficial].

**Algorithm Steps:**

1.  **Initialize Grid Density:** For each cell $c_i$ in grid $G$, initialize the accumulated density $\sigma(c_i)$ to 0 [S1, unofficial].
2.  **Map Triangles to Grid:** For each triangle $t_k$ in the volume:
    a. Calculate the base density of the triangle:
       $$ \sigma(t_k) = \frac{1}{\text{area}(t_k)} $$
       where $\text{area}(t)$ returns the area of triangle $t$ [S1, unofficial].
    b. For each cell $c_i$ in grid $G$:
       i. Calculate the overlapping area between the cell and the triangle: $A_{overlap} = \text{overlap}(c_i, t_k)$ [S1, unofficial].
       ii. Add the triangle's contribution to the cell's density:
           $$ \sigma(c_i) \leftarrow \sigma(c_i) + \frac{\text{overlap}(c_i, t_k)}{\text{area}(c_i)} \cdot \sigma(t_k) $$
           where $\text{overlap}(x_0, x_1)$ returns the common area (or length) of objects $x_0$ and $x_1$ [S1, unofficial].
3.  **Ray Tracing:** For each ray $r_j$:
    a. Initialize the absorption coefficient $\alpha(r_j)$ to 0 [S1, unofficial].
    b. For each cell $c_i$ intersected by ray $r_j$:
       i. Calculate the interval length (distance between entry and exit points of $r_j$ in $c_i$): $L_{overlap} = \text{overlap}(r_j, c_i)$ [S1, unofficial].
       ii. Accumulate the absorption:
           $$ \alpha(r_j) \leftarrow \alpha(r_j) + \sigma(c_i) \cdot \text{overlap}(r_j, c_i) $$
           where $\text{overlap}(r_j, c_i)$ represents the length of the ray segment inside the cell [S1, unofficial].
4.  **Output:** Visualize $\alpha(r_j)$ for all $j$ as a graph across the viewport [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sm2d(triangles, grid_rows, grid_cols, num_rays):
    """
    Simulates the Sm2D experiment.
    
    Args:
        triangles: List of Triangle objects, each with method area() and overlap_with(cell).
        grid_rows: Integer, number of rows in the grid.
        grid_cols: Integer, number of columns in the grid.
        num_rays: Integer, number of rays to trace.
        
    Returns:
        List of absorption coefficients alpha for each ray.
    """
    # 1. Initialize Grid
    # Grid cells are indexed by (r, c). 
    # Assume each cell has a uniform area. Let cell_area be a constant or derived from grid bounds.
    # For simplicity, assume grid covers unit square [0,1]x[0,1], so cell_area = 1/(grid_rows*grid_cols)
    cell_area = 1.0 / (grid_rows * grid_cols)
    
    # sigma_grid[r][c] stores the accumulated density for cell (r,c)
    sigma_grid = [[0.0 for _ in range(grid_cols)] for _ in range(grid_rows)]
    
    # 2. Map Triangles to Grid
    for t in triangles:
        t_area = t.area()
        if t_area == 0:
            continue # Avoid division by zero for degenerate triangles
            
        t_density = 1.0 / t_area
        
        for r in range(grid_rows):
            for c in range(grid_cols):
                # Define cell geometry for overlap calculation
                # Assuming grid cells are axis-aligned squares in [0,1]x[0,1]
                x0, y0 = c / grid_cols, r / grid_cols
                x1, y1 = (c + 1) / grid_cols, (r + 1) / grid_cols
                
                # Calculate overlap area between triangle t and cell (r,c)
                # This requires a geometric intersection function.
                # overlap_area = geometric_overlap_area(t, x0, y0, x1, y1)
                # [G] Placeholder for specific geometric intersection logic.
                # In the source, overlap() is a generic function.
                overlap_area = geometric_overlap_area(t, x0, y0, x1, y1)
                
                if overlap_area > 0:
                    sigma_grid[r][c] += (overlap_area / cell_area) * t_density

    # 3. Ray Tracing
    # Rays enter from below (y=0) at equally spaced x positions.
    # Rays travel upwards (positive y direction).
    ray_xs = [i / num_rays for i in range(num_rays)]
    absorption_coeffs = []
    
    for x_ray in ray_xs:
        alpha = 0.0
        
        # Ray intersects cells in column corresponding to x_ray
        # Determine which column the ray falls into
        col_idx = int(x_ray * grid_cols)
        if col_idx >= grid_cols:
            col_idx = grid_cols - 1
            
        # Ray spans from y=0 to y=1 (assuming grid height is 1)
        # It intersects all rows in that column
        for r in range(grid_rows):
            # Cell boundaries
            y0 = r / grid_rows
            y1 = (r + 1) / grid_rows
            
            # Ray segment inside this cell is from y0 to y1
            # Length is 1/grid_rows
            ray_segment_length = 1.0 / grid_rows
            
            # Overlap of ray and cell is the length of the segment
            # Since the ray is vertical and cell is axis aligned, 
            # the overlap length is simply the height of the cell if the ray is inside the column.
            overlap_length = ray_segment_length
            
            alpha += sigma_grid[r][col_idx] * overlap_length
            
        absorption_coeffs.append(alpha)
        
    return absorption_coeffs

def geometric_overlap_area(triangle, x0, y0, x1, y1):
    """
    [G] Computes the area of intersection between a triangle and an axis-aligned rectangle.
    This is a standard computational geometry operation.
    """
    # Implementation details omitted for brevity, but required for a complete implementation.
    # Common algorithms include Sutherland-Hodgman polygon clipping or 
    # decomposition into trapezoids.
    pass
```

**Parameters:**
*   Grid Resolution: The example uses a $16 \times 16$ grid [S1, unofficial].
*   Number of Rays: The example uses 64 rays [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Degenerate Triangles:** Triangles with zero area cause division by zero in $\sigma(t_k) = 1/\text{area}(t_k)$ [S1, unofficial].
*   **Rotational Artifacts:** The texture-based approach produces unstable and faulty results during rotation, unlike explicit calculations which remain smooth [S1, unofficial].
*   **Spikes:** Unpleasant spikes across the viewport can occur, especially for degenerated triangles or unfavorable ray angles. These are unavoidable under certain conditions regardless of grid resolution [S1, unofficial].
*   **Precision:** A texture-based approach will never get close to the precision of an explicit calculation without considerably increasing the number of grid cells [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_t \cdot N_c)$ for mapping triangles to the grid, where $N_t$ is the number of triangles and $N_c$ is the number of grid cells. Ray tracing is $O(N_r \cdot N_r)$ if rays are processed per cell, or $O(N_r)$ if optimized per ray column.
*   **Memory Complexity:** $O(N_c)$ to store the grid density $\sigma(c_i)$ [S1, unofficial].
*   **Strengths:** Simple implementation; demonstrates the fundamental limitations of texture-based volume rendering [S1, unofficial].
*   **Limitations:** Low precision compared to explicit methods; suffers from rotational artifacts and spikes; requires high resolution for acceptable accuracy [S1, unofficial].
*   **When to use:** As a diagnostic tool or educational example to understand texture-based volume rendering artifacts [S1, unofficial].
*   **When not to use:** For production-quality smoke rendering where precision and smooth rotation are required [S1, unofficial].

## Background
Sm2D was introduced in a Master's thesis by Dirk Sowinski in 2012 as part of research on real-time smoke volume rendering [S1, unofficial]. It serves as a simplified 2D test case to evaluate the applicability of texture-based approaches for smoke volumes, highlighting the trade-offs between hardware-accelerated texture sampling and explicit geometric ray tracing [S1, unofficial].

## Sources
[S1] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
