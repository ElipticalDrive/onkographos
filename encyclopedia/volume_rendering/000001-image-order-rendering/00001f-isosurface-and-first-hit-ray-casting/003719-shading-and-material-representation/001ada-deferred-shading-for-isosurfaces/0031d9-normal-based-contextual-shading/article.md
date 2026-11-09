# Normal-based Contextual Shading

Also called: Contextual Shading

## Overview
Normal-based Contextual Shading is an object-space shading technique applied to isosurfaces to compute surface normals for illumination [S1, unofficial]. It solves the problem of inaccurate or artifact-prone normals in volume rendering by fitting a local surface approximation to the 3D intersection points of rays with the isosurface [S1, unofficial]. It differs from its sibling, Surface shading for thresholded volumes, by explicitly estimating surface inclination via local regression rather than relying on thresholded binary data properties [S1, unofficial].

## How it works
**Inputs:**
- A set of closest intersection points $\{P_i\}$ for each pixel, where each $P_i = (x_i, y_i, z_i)$ is the 3D coordinate of the ray-isosurface intersection [S1, unofficial].
- A voxel neighborhood size defining the local region around the central intersection point [S1, unofficial].

**Outputs:**
- A surface normal vector $\mathbf{n}$ for each pixel, used in the final shading pass [S1, unofficial].

**Data Structures:**
- A list of 3D points $\{P_i\}$ within the defined voxel neighborhood [S1, unofficial]. Standard floating-point coordinates are used [G].

**Steps:**
1. **Collect Neighborhood Points:** For a given pixel with intersection point $P_0$, identify all other intersection points $P_i$ that lie within a larger voxel neighborhood around $P_0$ [S1, unofficial].
2. **Fit Surface Approximation:** Fit a function to the set of points $\{P_i\}$ to estimate the local surface geometry [S1, unofficial]. The function can be a linear function (plane) $z = ax + by + c$ [S1, unofficial], or a biquadratic function $z = ax^2 + by^2 + cxy + dx + ey + f$ [S1, unofficial].
3. **Compute Normal:** Calculate the normal vector from the coefficients of the fitted function [S1, unofficial]. For a linear fit, the normal is $\mathbf{n} = (-a, -b, 1)$ normalized [S1, unofficial]. For a biquadratic fit, the normal is derived from the gradient of the surface at $P_0$ [S1, unofficial]. Alternatively, the normal for a voxel face on the surface can be determined by examining the orientation of that face and the orientation of the four edge-connected faces on the surface [S2, unofficial].
4. **Assign Normal:** Store the computed normal for the pixel to be used in the deferred shading pass [S1, unofficial].

**Parameters:**
- **Voxel Neighborhood Size:** A larger voxel neighborhood is taken into account to estimate the surface inclination [S1, unofficial]. The exact size is not specified in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
- Since a voxel face has only six possible orientations, the error in the approximated normal can be significant [S2, unofficial].
- At object boundaries or thin structures, the neighborhood may not contain enough points for a stable fit [G].
- If the neighborhood contains collinear or coplanar points, the linear fit may be ill-conditioned [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_normal_contextual(points):
    """
    Compute normal for a pixel using normal-based contextual shading.
    
    Args:
        points: List of 3D points (x, y, z) in the voxel neighborhood.
        
    Returns:
        Normal vector (nx, ny, nz).
    """
    if len(points) < 3:
        return (0.0, 0.0, 1.0)
        
    # Compute sums for least squares plane fit z = ax + by + c
    sx, sy, sxx, sxy, syy, sxz, syz, sz, n = 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, len(points)
    for p in points:
        x, y, z = p
        sx += x
        sy += y
        sxx += x * x
        sxy += x * y
        syy += y * y
        sxz += x * z
        syz += y * z
        sz += z
        
    # Solve 2x2 system for a and b:
    # [sxx sxy] [a]   [sxz]
    # [sxy syy] [b] = [syz]
    det = sxx * syy - sxy * sxy
    if abs(det) < 1e-9:
        return (0.0, 0.0, 1.0)
        
    a = (sxz * syy - syz * sxy) / det
    b = (syz * sxx - sxz * sxy) / det
    
    # Normal is (-a, -b, 1) normalized
    length = (a * a + b * b + 1.0) ** 0.5
    return (-a / length, -b / length, 1.0 / length)
```

## Subtopics

### [Depth-Gradient Shading](6618-depth-gradient-shading.md)
Depth-gradient shading is an image-based shading technique that approximates the surface gradient by calculating differences between the depths of a given pixel and its neighbors in the z-buffer. It provides shading cues for isosurfaces or front faces in volume rendering without performing expensive object-space gradient calculations for every voxel. It differs from its parent, Normal-based Contextual Shading, by being purely image-space and context-insensitive, which makes it faster but prone to sharp contour artifacts at object boundaries.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Slower than image-space depth-gradient shading due to object-space calculations [S1, unofficial].
- **Strengths:** Better visual quality than depth-gradient shading [S1, unofficial].
- **Limitations:** The error in the approximated normal can be significant due to the discrete six possible voxel face orientations [S2, unofficial].
- **When to Use:** Not given in the sources.
- **When Not to Use:** Not given in the sources.

## Background
Normal-based Contextual Shading is an object-space shading technique applied after finding closest intersection points to achieve better results than image-space depth-gradient shading [S1, unofficial]. It is based on normal computation methods that try to fit a linear or a biquadratic function to the set of points that belong to the same iso-surface [S1, unofficial]. These techniques take a larger voxel neighborhood into account to estimate the surface inclination [S1, unofficial]. The method is discussed in the context of interactive volume-rendering techniques for medical data visualization [S1, unofficial].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S2] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
