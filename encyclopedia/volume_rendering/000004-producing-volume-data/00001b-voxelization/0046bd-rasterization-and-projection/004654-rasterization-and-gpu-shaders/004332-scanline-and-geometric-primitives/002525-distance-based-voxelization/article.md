# Distance-based Voxelization
Also called: Direct Voxelization of Geometric Objects, Distance-based Voxelization for Cubic Objects, Lagrangian Method Voxelization

## Overview
Distance-based Voxelization is a method that voxelize objects directly without polygon conversion by including voxels where the distance to the surface $S$ is smaller than a threshold $t$ [S1]. Unlike [3D Scan-conversion](1895-3d-scan-conversion.md) which relies on polygon mesh intersection, this approach bypasses polygon conversion, offering efficiency and accuracy for regular shapes and surfaces where distance metrics are computable [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A geometric surface $S$ defined analytically or parametrically; a voxel grid resolution $(N_x, N_y, N_z)$; a distance threshold $t$; and a bounding box defining the search space [S1].
*   **Output:** A 6-connected or 26-connected voxel representation of the surface $S$ [S1].

**Data Structures:**
*   **Voxel Grid:** A 3D array of size $N_x \times N_y \times N_z$ storing binary inclusion values [S1].
*   **Surface Representation:** For regular shapes (spheres, cylinders), implicit equations are used [S1]. For general surfaces, a parametric or implicit form allowing distance evaluation is required [S1].

**Steps:**
1.  **Define Search Space:** Determine the bounding box of the object $S$ and restrict the voxel scan to a subset of the voxel space in the vicinity of the surface to ensure efficiency [S1].
2.  **Iterate Voxels:** For each voxel center $(x, y, z)$ within the search space, calculate the minimum distance to the surface $S$ [S1].
3.  **Distance Calculation:** For regular shapes, compute the distance directly using geometric formulas [S1]. For general surfaces, use the Lagrangian Method to find the point $(x_0, y_0, z_0)$ on the surface that minimizes the distance to the voxel center $(x, y, z)$ [S1].
4.  **Threshold Check:** If the calculated distance is smaller than $t$, mark the voxel as part of the surface representation [S1].

**Formulas:**
The distance minimization problem is defined by minimizing the squared Euclidean distance function [S1]:
$$p(x, y, z) = (x - x_0)^2 + (y - y_0)^2 + (z - z_0)^2$$
where $(x_0, y_0, z_0)$ is a point on the surface $S$ and $(x, y, z)$ is the voxel center [S1]. The goal is to find $(x_0, y_0, z_0)$ such that $p(x, y, z)$ is minimized subject to the constraint that $(x_0, y_0, z_0)$ lies on $S$ [S1].

**Parameters:**
*   **Threshold $t$:** Determines the thickness of the voxelized surface [S1]. Typical values depend on grid resolution; e.g., for a $256 \times 256 \times 256$ grid, specific $t$ values are chosen to achieve 6-separability [S1].
*   **Object Dimensions:** Sphere radius: 61.4 [S1]. Cylinder radius: 49.7 and height: 158.2 [S1]. Function parameter $k$ for $x^2 + y^2 + kz = 0$: 10.0 [S1].

**Pitfalls:**
*   **Numerical Stability:** Solving the Lagrangian Method for general surfaces may require solving high-order polynomial equations that lack analytical solutions [S1]. Numerical methods must be employed with high accuracy to avoid incorrect distance estimates [S1].
*   **Efficiency:** Scanning the entire voxel space is inefficient [S1]. The search must be restricted to the vicinity of the surface [S1].
*   **Topological Correctness:** For non-planar objects, topological correctness needs further proof and careful handling of the threshold $t$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def distance_based_voxelization(surface_distance_func, grid_resolution, threshold):
    """
    Voxelize a surface S by including voxels within distance threshold.
    
    Args:
        surface_distance_func: Callable that takes a 3D point (x, y, z) 
                               and returns the minimum distance to the surface.
        grid_resolution: Tuple (Nx, Ny, Nz) defining the voxel grid dimensions.
        threshold: Float, the maximum distance for a voxel to be included.
        
    Returns:
        A 3D list of shape (Nx, Ny, Nz) with boolean values.
    """
    Nx, Ny, Nz = grid_resolution
    # Initialize 3D grid with False values
    voxel_grid = [[[False] * Nz for _ in range(Ny)] for _ in range(Nx)]
    
    # Sources do not specify the exact bounding box derivation algorithm [G]
    # Assume the grid coordinates map directly to the search space
    
    for x in range(Nx):
        for y in range(Ny):
            for z in range(Nz):
                # Voxel center coordinates
                voxel_center = (x + 0.5, y + 0.5, z + 0.5)
                
                # Calculate minimum distance to surface
                # For regular shapes, this is a direct formula.
                # For general surfaces, this involves the Lagrangian Method.
                # Sources state that numerical methods must be used for high accuracy [S1]
                dist = surface_distance_func(voxel_center)
                
                # Include voxel if distance is within threshold
                if dist < threshold:
                    voxel_grid[x][y][z] = True
                    
    return voxel_grid
```

## Subtopics

### [Volume-Sampled Antialiasing](11212-volume-sampled-antialiasing.md)
A 3D object-space antialiasing technique that performs antialiasing once on a 3D view-independent representation during the modeling stage to produce alias-free 3D models stored in a volume buffer. It extends the distance-based approach by replacing the binary separability requirement with a 'uniform thickness' and using voxel values as a function of distance to the surface.

## Operating characteristics
*   **Time Complexity:** $O(N_x N_y N_z \cdot D)$, where $D$ is the cost of computing the distance to the surface [G]. For regular shapes, $D$ is $O(1)$ [S1]. For general surfaces using the Lagrangian Method, $D$ can be significantly higher due to iterative numerical solving [S1].
*   **Memory Complexity:** $O(N_x N_y N_z)$ for storing the voxel grid [G].
*   **Performance:** The method is efficient for regular shapes like spheres and cylinders where distance measurement is straightforward [S1]. For general surfaces, performance depends on the efficiency of the numerical solver for the Lagrangian Method [S1].
*   **Strengths:** Allows direct voxelization without polygon conversion [S1].
*   **Limitations:** Requires efficient distance computation; numerical instability for high-order polynomials; topological correctness for non-planar objects is not fully proven [S1].
*   **When to Use:** When direct voxelization of analytic or regular geometric objects is desired, and polygon conversion is undesirable or inefficient [S1].

## Background
The method was introduced by Cohen and Kaufman as an extension of their accurate algorithm for voxelizing polygon meshes [S1]. It was presented in the paper "An Accurate Method for Voxelizing Polygon Meshes" [S1]. The approach builds on the observation that voxelization can be viewed as a distance-based inclusion problem, allowing for the direct voxelization of geometric objects without intermediate polygon conversion [S1]. The project was partially supported by Ford Motor Company [S1].

## Sources
[S1] An Accurate Method for Voxelizing Polygon Meshes, https://web.eecs.utk.edu/~huangj/papers/polygon.pdf, fetched 2026-10-29
