# Raycasting with Irregular Grid Interpolation

Also called: Adaptive Grid Raycasting, TSDF Zero-Level Set Extraction

## Overview
This method extends SDF ray marching to handle hierarchical voxel grids with varying resolutions [S1]. It solves the problem of evaluating the Signed Distance Function (SDF) at arbitrary points near boundaries between different resolution levels, where standard tri-linear interpolation is invalid [S1]. It sets itself apart from its parent by replacing the uniform interpolation kernel with a local polynomial fit derived from a linear system, ensuring continuity across resolution boundaries [S1].

## How it works
**Inputs:**
*   Camera pose $(R, t)$ and intrinsic calibration $K$ [S1].
*   Hierarchical TSDF grid composed of voxel blocks with potentially different resolutions [S1].
*   Pre-computed depth range map: for each pixel, the minimum and maximum depth values derived from projecting the bounding boxes of observed voxel blocks into the image [S1].

**Outputs:**
*   A map of 3D points and surface normals (and optionally surface colors) corresponding to the zero-level set $F(X) = 0$ [S1].

**Data Structures:**
*   **Voxel Block:** Contains grid data at a specific resolution. The structure of the grid values is not specified beyond being accessible via coordinates [S1].
*   **Depth Range Map:** A 2D array mapping pixel coordinates $(u, v)$ to a depth interval $[d_{min}, d_{max}]$ [S1].

**Algorithm:**

1.  **Pre-computation:** Project the bounding boxes of all observed voxel blocks into the image plane. For each pixel, determine the plausible depth range $[d_{min}, d_{max}]$ by filling these projections with appropriate minimum and maximum depth values [S1].

2.  **Ray Casting Loop:** For each pixel $(u, v)$:
    a.  Construct the ray origin and direction from the camera model $(R, t, K)$ [S1].
    b.  Initialize the ray step range using the pre-computed $[d_{min}, d_{max}]$ for this pixel [S1].
    c.  Perform ray marching steps along the line of sight to find the point $X$ where $F(X) = 0$ [S1].

3.  **SDF Evaluation $F(X)$:** At each ray marching step, evaluate the SDF at the current 3D point $X = (X, Y, Z)$:
    a.  Identify the 8 surrounding grid points (corners of the voxel cube containing $X$) [S1].
    b.  **Check Resolution Uniformity:** Determine if any two of these 8 surrounding grid points reside in different resolution levels (i.e., different voxel blocks) [S1].
    c.  **If Uniform (Regular Grid):** Use standard tri-linear interpolation. Accumulate values from the 8 surrounding grid points weighted by linear interpolation coefficients [S1].
    d.  **If Non-Uniform (Irregular Grid):** Use the irregular interpolation function defined below [S1].

4.  **Irregular Grid Interpolation:**
    When the 8 surrounding points span different resolutions, solve for the interpolating function coefficients. The function is defined as:
    $$ F(X, Y, Z) = a_1 XYZ + a_2 XY + a_3 YZ + a_4 XZ + a_5 X + a_6 Y + a_7 Z + a_8 $$
    Each of the 8 surrounding grid points $(X_i, Y_i, Z_i)$ with known SDF values $F(X_i, Y_i, Z_i)$ provides one equation [S1]. This results in the linear system:
    $$ \mathbf{A} \mathbf{a} = \mathbf{b} $$
    where $\mathbf{a} = [a_1, \dots, a_8]^T$ and $\mathbf{b}$ is the vector of SDF values at the 8 points [S1]. The matrix $\mathbf{A}$ is constructed such that row $i$ is:
    $$ [X_i Y_i Z_i, \ X_i Y_i, \ Y_i Z_i, \ X_i Z_i, \ X_i, \ Y_i, \ Z_i, \ 1] $$
    To solve this system efficiently:
    a.  Arbitrarily shift the coordinate system such that $X_1 = 0$ [S1].
    b.  This trivially solves for $a_8$ (the constant term) [S1].
    c.  Solve the remaining inhomogeneous $7 \times 7$ system for $a_1, \dots, a_7$ using Gaussian elimination [S1].
    d.  Evaluate $F(X, Y, Z)$ using the solved coefficients [S1].

5.  **Termination:** Stop when the zero-level set is found within the depth range or the range is exhausted [S1].

**Pseudocode:**

```python
def evaluate_sdf_irregular(points_8, values_8, query_point):
    """
    Interpolate SDF value at query_point using 8 surrounding grid points.
    points_8: list of 8 tuples (X, Y, Z)
    values_8: list of 8 floats F(X, Y, Z)
    query_point: tuple (X, Y, Z)
    """
    # Shift coordinates so that the first point is at origin
    # This simplifies solving for a8
    shifted_points = []
    for i in range(8):
        x, y, z = points_8[i]
        # Shift relative to the first point
        sx = x - points_8[0][0]
        sy = y - points_8[0][1]
        sz = z - points_8[0][2]
        shifted_points.append((sx, sy, sz))
    
    # The first point is now (0,0,0)
    # F(0,0,0) = a8
    a8 = values_8[0]
    
    # Build the 7x7 system for a1..a7
    # The basis functions for the remaining coefficients are:
    # XYZ, XY, YZ, XZ, X, Y, Z
    # Note: The original formula uses absolute coordinates, but shifting
    # changes the basis. However, the source states: "By arbitrarily shifting
    # the coordinate system... such that X1=0... solve the remaining...".
    # We must construct the matrix A using the shifted coordinates for the
    # inhomogeneous part, but we must be careful. The source implies we
    # solve for the coefficients of the polynomial in the shifted space
    # or adjust the basis. Let's follow the source's implication of solving
    # the linear system derived from the points.
    
    # Re-reading source: "Each of the surrounding 8 grid points gives one equation...
    # By arbitrarily shifting... such that X1=0... solves for a8. We solve the
    # remaining... using Gaussian elimination."
    # This implies we solve the system for the coefficients a1..a7 in the
    # context of the shifted coordinates if we assume the form holds in shifted coords,
    # OR we keep the form in original coords but use the fact that a8 is known.
    # Let's stick to the most robust interpretation: Solve for a1..a7 using
    # the original points, knowing a8 = F(X1).
    
    # Actually, the source says "shifting... such that X1=0". This usually means
    # we transform the query point and grid points to a local frame where X1 is origin.
    # Let's assume the polynomial form is evaluated in this local frame.
    
    # Local frame origin at points_8[0]
    # Query point in local frame
    qx = query_point[0] - points_8[0][0]
    qy = query_point[1] - points_8[0][1]
    qz = query_point[2] - points_8[0][2]
    
    # Matrix A for the 7 unknowns (a1..a7) corresponding to basis:
    # XYZ, XY, YZ, XZ, X, Y, Z
    # Using the 7 points excluding the first one (since a8 is known)
    # Wait, the source says "remaining 7x7 system". This implies using 7 equations.
    # The 8th equation (for point 1) is used to find a8.
    
    A = []
    b = []
    
    for i in range(1, 8):
        x, y, z = shifted_points[i]
        # Basis functions evaluated at shifted point i
        row = [
            x * y * z,  # a1
            x * y,      # a2
            y * z,      # a3
            x * z,      # a4
            x,          # a5
            y,          # a6
            z           # a7
        ]
        A.append(row)
        # The value at this point is F_i.
        # F_i = a1*xyz + ... + a7*z + a8
        # So, a1*xyz + ... + a7*z = F_i - a8
        b.append(values_8[i] - a8)
        
    # Solve A * [a1..a7]^T = b using Gaussian elimination
    coeffs = solve_linear_system(A, b)
    
    # Evaluate at query point in local frame
    # F(q) = a1*qx*qy*qz + a2*qx*qy + a3*qy*qz + a4*qx*qz + a5*qx + a6*qy + a7*qz + a8
    result = (coeffs[0] * qx * qy * qz +
              coeffs[1] * qx * qy +
              coeffs[2] * qy * qz +
              coeffs[3] * qx * qz +
              coeffs[4] * qx +
              coeffs[5] * qy +
              coeffs[6] * qz +
              a8)
              
    return result

def raycast_pixel(u, v, camera, grid, depth_map):
    ray_origin, ray_direction = camera.get_ray(u, v)
    d_min, d_max = depth_map[u, v]
    
    # Standard ray marching loop
    t = d_min
    while t < d_max:
        X = ray_origin + t * ray_direction
        
        # Get 8 surrounding grid points
        points_8, values_8 = grid.get_surrounding_points(X)
        
        # Check if irregular
        if grid.is_irregular(points_8):
            F = evaluate_sdf_irregular(points_8, values_8, X)
        else:
            F = tri_linear_interpolate(points_8, values_8, X)
            
        if abs(F) < epsilon:
            return X, compute_normal(grid, X)
            
        t += step_size(F)
        
    return None
```

**Parameters:**
*   **Epsilon:** Threshold for zero-level set detection. Not given in the sources.
*   **Step Size:** Determined by the SDF value $F$ (implied by "SDF Ray Marching with Grid Lookup"). Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Cost:** Irregular grid interpolation is computationally expensive due to the linear system solution [S1].
*   **Optimization:** A simple check must be performed to ensure irregular interpolation is only used when at least two of the 8 surrounding grid points reside in different resolution levels [S1].
*   **Numerical Stability:** Gaussian elimination on the $7 \times 7$ system may be sensitive to ill-conditioned matrices if the grid points are nearly coplanar or degenerate. Not given in the sources.

## Operating characteristics
*   **Time Complexity:** Regular interpolation is $O(1)$ with a small constant. Irregular interpolation involves solving a $7 \times 7$ linear system, which is $O(1)$ but with a significantly higher constant factor than tri-linear interpolation [S1].
*   **Memory Complexity:** Depends on the hierarchical voxel block storage. The method itself requires storing the depth range map and the grid data [S1].
*   **Performance:** The method is implemented on GPU using CUDA [S1]. The cost of irregular interpolation is mitigated by the check to ensure it is only performed at resolution boundaries [S1].
*   **Strengths:** Handles hierarchical grids with varying resolutions seamlessly, maintaining accuracy at boundaries [S1].
*   **Limitations:** Higher computational cost at resolution boundaries compared to uniform grids [S1].
*   **When to use:** When using hierarchical TSDF representations with multiple resolution levels [S1].
*   **When not to use:** For uniform grids where tri-linear interpolation suffices [S1].

## Background
The method originates from the work on Hierarchical Voxel Block Hashing for Efficient Integration of Depth Images [S1]. It was developed to support online ICP tracking and TSDF integration in a hierarchical representation, allowing for efficient storage and processing of large-scale 3D data [S1]. The source code is available at http://infinitam.org [S1].

## Sources
[S1] Hierarchical Voxel Block Hashing for Efficient Integration of Depth Images, https://doi.org/10.1109/lra.2015.2512958, fetched 2026-11-08
