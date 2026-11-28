# Joseph's Method
Also called: Joseph forward projection, Joseph interpolation
## Overview
Joseph's Method is a forward projection algorithm for computing line integrals through a discrete voxel volume, primarily used in computed tomography (CT) reconstruction [S1, unofficial]. It solves the problem of accurately sampling a 3D volume along arbitrary ray directions while avoiding the artifacts associated with nearest-neighbor sampling [S1, unofficial]. Unlike Siddon’s DDA, which degenerates to nearest-neighbor sampling when rays align with grid axes, Joseph's Method uses driving-axis-aligned sampling combined with bilinear interpolation in the remaining dimensions to model the integrating nature of detector pixels [S1, unofficial]. It performs similarly to Köhler's method in accuracy, despite using a skewed interpolation kernel compared to Köhler's tri-linear kernel [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A 3D voxel grid $V$ with dimensions $D_x \times D_y \times D_z$.
- Input: Ray segment defined by endpoints $\mathbf{p}_1 = (x_1, y_1, z_1)$ and $\mathbf{p}_2 = (x_2, y_2, z_2)$ in voxel coordinates.
- Output: The weighted sum of interpolated values along the ray segment.

**Data Structures:**
- The voxel grid $V$ is accessed via integer indices $(i, j, k)$ corresponding to $(x, y, z)$ coordinates. Values are assumed to be stored in a contiguous array or 3D array structure.

**Steps:**
1. **Determine Principal Direction:** Calculate the absolute differences along each axis: $\Delta x = |x_1 - x_2|$, $\Delta y = |y_1 - y_2|$, $\Delta z = |z_1 - z_2|$. Identify the axis with the maximum difference. Assume without loss of generality that $\Delta x$ is the largest, making $x$ the principal direction. The other cases ($\Delta y$ or $\Delta z$ largest) are handled symmetrically [S1, unofficial].
2. **Iterate Along Principal Axis:** Iterate through the voxel indices $i$ along the principal axis ($x$) that the ray passes through. For each $i$, determine the corresponding $y$ and $z$ coordinates on the ray at the boundaries of the voxel slice.
3. **Bilinear Interpolation:** For each slice $i$, perform bilinear interpolation on the four surrounding voxels in the $(y, z)$ plane. Let the ray intersect the plane at $(y_i, z_i)$. The fractional parts $f_y = y_i - \lfloor y_i \rfloor$ and $f_z = z_i - \lfloor z_i \rfloor$ are used to interpolate between the four corner voxels $V(i, j, k), V(i, j+1, k), V(i, j, k+1), V(i, j+1, k+1)$, where $j = \lfloor y_i \rfloor$ and $k = \lfloor z_i \rfloor$.
4. **Sum Interpolated Values:** Sum the results of the bilinear interpolations for all slices $i$ traversed by the ray.
5. **Apply Compensation Factor:** Multiply the sum by a compensation factor to account for the longer traversal path of rays deviating from the principal direction. The factor is:
   $$ C = \frac{\sqrt{(x_1 - x_2)^2 + (y_1 - y_2)^2 + (z_1 - z_2)^2}}{|x_1 - x_2|} $$
   The final result is $C \times \sum_{i} \text{BilinearInterp}(i)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def joseph_method(V, dims, p1, p2):
    """
    Joseph's forward projection method.
    V: 3D array of voxel values.
    dims: Tuple (Dx, Dy, Dz) of grid dimensions.
    p1: Tuple (x1, y1, z1) start point in voxel coordinates.
    p2: Tuple (x2, y2, z2) end point in voxel coordinates.
    """
    x1, y1, z1 = p1
    x2, y2, z2 = p2
    
    dx = abs(x2 - x1)
    dy = abs(y2 - y1)
    dz = abs(z2 - z1)
    
    # Determine principal axis
    if dx >= dy and dx >= dz:
        principal = 'x'
        denom = dx
    elif dy >= dx and dy >= dz:
        principal = 'y'
        denom = dy
    else:
        principal = 'z'
        denom = dz
        
    # Calculate compensation factor
    ray_length = sqrt((x2-x1)**2 + (y2-y1)**2 + (z2-z1)**2)
    compensation = ray_length / denom
    
    total_sum = 0.0
    
    if principal == 'x':
        # Iterate over x slices
        x_start = min(x1, x2)
        x_end = max(x1, x2)
        for i in range(int(x_start), int(x_end) + 1):
            # Calculate y and z at the center or boundaries of the slice?
            # The source implies intersection points of each (y,z)-plane.
            # We interpolate at the ray's position within the slice.
            # To be precise, we need the y and z coordinates corresponding to the slice.
            # Assuming linear parameterization along the ray.
            # Let t be the parameter such that x(t) = i.
            # However, the source describes "intersection points of each (y,z)-plane".
            # This usually means evaluating the ray at the voxel boundaries or centers.
            # Given the bilinear interpolation on (y,z), we need y and z at the slice.
            # Let's assume we evaluate at the slice index i.
            # If the ray is x(t) = x1 + t*(x2-x1), then t = (i - x1)/(x2-x1).
            # But since we iterate integer slices, we can just interpolate y and z linearly.
            
            # Normalize t for the current slice i relative to the ray segment
            # This is an approximation; precise implementation requires careful t calculation.
            # For simplicity, we assume the ray passes through the slice at a specific y,z.
            # A more robust way: calculate y and z at the midpoint of the slice or boundaries.
            # The source is vague on the exact t for y,z. We assume linear interpolation of y,z with x.
            
            # Let's use the ratio of distances.
            # If x1 != x2:
            if x2 != x1:
                t = (i - x1) / (x2 - x1)
            else:
                t = 0.0 # Degenerate case
            
            y_i = y1 + t * (y2 - y1)
            z_i = z1 + t * (z2 - z1)
            
            # Bilinear interpolation in y-z plane at slice i
            total_sum += bilinear_interpolate(V, dims, i, y_i, z_i)
            
    elif principal == 'y':
        y_start = min(y1, y2)
        y_end = max(y1, y2)
        for j in range(int(y_start), int(y_end) + 1):
            if y2 != y1:
                t = (j - y1) / (y2 - y1)
            else:
                t = 0.0
            x_j = x1 + t * (x2 - x1)
            z_j = z1 + t * (z2 - z1)
            total_sum += bilinear_interpolate(V, dims, x_j, j, z_j)
            
    else: # principal == 'z'
        z_start = min(z1, z2)
        z_end = max(z1, z2)
        for k in range(int(z_start), int(z_end) + 1):
            if z2 != z1:
                t = (k - z1) / (z2 - z1)
            else:
                t = 0.0
            x_k = x1 + t * (x2 - x1)
            y_k = y1 + t * (y2 - y1)
            total_sum += bilinear_interpolate(V, dims, x_k, y_k, k)
            
    return compensation * total_sum

def bilinear_interpolate(V, dims, x, y, z):
    """
    Performs bilinear interpolation on the plane perpendicular to the principal axis.
    x, y, z are continuous coordinates. One of them is the principal axis index (integer or close to it).
    For principal='x', we interpolate in y and z at fixed x.
    """
    # Clamp coordinates to valid range
    x = max(0, min(x, dims[0] - 1))
    y = max(0, min(y, dims[1] - 1))
    z = max(0, min(z, dims[2] - 1))
    
    # Determine integer and fractional parts for the non-principal axes
    # Assuming principal is x, we interpolate y and z
    iy = int(y)
    iz = int(z)
    fy = y - iy
    fz = z - iz
    
    # Clamp indices
    iy0, iy1 = max(0, min(iy, dims[1]-1)), max(0, min(iy+1, dims[1]-1))
    iz0, iz1 = max(0, min(iz, dims[2]-1)), max(0, min(iz+1, dims[2]-1))
    
    # Get values
    v00 = V[int(x), iy0, iz0]
    v10 = V[int(x), iy1, iz0]
    v01 = V[int(x), iy0, iz1]
    v11 = V[int(x), iy1, iz1]
    
    # Interpolate
    v0 = v00 * (1 - fy) + v10 * fy
    v1 = v01 * (1 - fy) + v11 * fy
    return v0 * (1 - fz) + v1 * fz
```

**Parameters:**
- None explicitly defined beyond the ray endpoints and grid dimensions.

**Edge Cases and Pitfalls:**
- **Degenerate Rays:** If the ray is parallel to a principal axis (e.g., $x_1 = x_2$), the denominator in the compensation factor becomes zero. This case requires special handling, likely defaulting to nearest-neighbor or a different interpolation strategy [G].
- **Precision:** Floating-point precision issues may arise when calculating intersection points, especially for long rays or high-resolution grids [G].
- **Boundary Conditions:** Indices must be clamped to the valid range of the voxel grid to prevent out-of-bounds access [G].

## Subtopics

### [Generalized Joseph Projector with Linear Interpolation](22875-generalized-joseph-projector-with-linear-interpolation.md)
The Generalized Joseph Projector with Linear Interpolation (GJPlin) is a branchless, ray-driven forward projection algorithm designed for efficient GPU execution in cone-beam CT reconstruction. It solves the problem of accurately sampling a 3D voxel volume along arbitrary ray directions while maintaining high memory bandwidth efficiency and avoiding the artifacts associated with nearest-neighbor sampling. Unlike Siddon’s DDA, which degenerates to nearest-neighbor sampling when rays align with grid axes, GJPlin uses driving-axis-aligned sampling to ensure an optimal, non-redundant number of samples, combined with 2D bilinear interpolation in the remaining dimensions to model the integrating nature of detector pixels.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels traversed by the ray.
- **Memory Complexity:** Constant, $O(1)$ auxiliary space, assuming the voxel grid is provided.
- **Measured Performance:** Joseph's method performs similarly to Köhler's method in accuracy [S1, unofficial]. It is superior to Siddon's method in terms of accuracy, especially for rays deviating from principal directions [S1, unofficial].
- **Strengths:** Avoids nearest-neighbor artifacts; simple to implement; efficient for GPU execution when generalized.
- **Limitations:** Skewed interpolation kernel compared to tri-linear methods; requires careful handling of degenerate rays.
- **When to Use:** When high accuracy is required for forward projections in CT reconstruction, especially for rays with arbitrary orientations.
- **When Not to Use:** When computational resources are extremely limited and nearest-neighbor accuracy is acceptable, or when tri-linear interpolation is strictly required for consistency with other pipeline stages.

## Background
Joseph's Method was described by Joseph in 1982 for the two-dimensional case and is easily extended to three dimensions [S1, unofficial]. It is a forward projection method used in computed tomography (CT) reconstruction. The method was noted to perform similarly to Köhler's method in accuracy, despite using a skewed interpolation kernel [S1, unofficial]. The root mean square error of Joseph's method is significantly lower than that of Siddon's method [S1, unofficial].

## Sources
[S1] 5 Forward Projection through Voxel Volumes, https://www.cvl.isy.liu.se/education/undergraduate/tsbb31/download/TurbellPhDCh5.pdf, fetched 2026-10-25 (unofficial)
