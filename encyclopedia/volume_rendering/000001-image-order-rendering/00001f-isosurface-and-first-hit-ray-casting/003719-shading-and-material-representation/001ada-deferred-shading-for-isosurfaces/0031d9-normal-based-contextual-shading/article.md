# Normal-based Contextual Shading

Also called: Contextual Shading

## Overview
Normal-based Contextual Shading is an object-space shading technique applied to isosurfaces to compute surface normals more accurately than image-space methods like Depth-Gradient Shading [S1, unofficial]. It solves the problem of inaccurate normal estimation at voxel boundaries by fitting a local surface approximation to the set of intersection points belonging to the same iso-surface [S1, unofficial]. Unlike its parent, Deferred Shading for Isosurfaces, which computes gradients in volume space, this method operates on the discrete set of closest intersection points stored per pixel [S1, unofficial]. It is distinct from its sibling, Surface shading for thresholded volumes, by explicitly using a larger voxel neighborhood to estimate surface inclination [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of closest intersection points $(x_i, y_i, z_i)$ for a given pixel, where each point lies on the iso-surface.
- Output: A surface normal vector $\mathbf{n}$ for shading.

**Data Structures:**
- Points: A list of 3D coordinates $(x, y, z)$ representing the intersection of the viewing ray with the iso-surface. The number of points varies but typically includes the primary intersection and potentially neighbors if a neighborhood is defined.
- Voxel Neighborhood: A set of voxels surrounding the intersection points. The size of this neighborhood is larger than the immediate voxel to capture surface curvature [S1, unofficial].

**Steps:**
1. **Collect Intersection Points:** For each pixel, retrieve the closest intersection points stored during the ray-casting pass. These points belong to the same iso-surface [S1, unofficial].
2. **Define Local Neighborhood:** Identify the voxel neighborhood surrounding the intersection points. A larger neighborhood is used to better estimate surface inclination [S1, unofficial].
3. **Fit Surface Approximation:** Fit a linear or biquadratic function to the set of points in the neighborhood to estimate the surface inclination [S1, unofficial].
   - Let the fitted function be $f(x, y, z)$.
   - For a linear fit, the surface is approximated by a plane.
   - For a biquadratic fit, the surface is approximated by a quadratic surface [S1, unofficial].
4. **Compute Normal:** Calculate the gradient of the fitted function at the intersection point to obtain the normal vector.
   - If a linear fit $z = ax + by + c$ is used, the normal is $\mathbf{n} = (-a, -b, 1)$ normalized.
   - If a biquadratic fit is used, the normal is derived from the partial derivatives of the quadratic function [S1, unofficial].
5. **Alternative Face-Based Normal (Contextual):** If the intersection occurs at a voxel face, determine the normal by examining the orientation of that face and the orientation of the four edge-connected faces on the surface [S2, unofficial].
   - Let the face orientation be $\mathbf{f}$.
   - Let the orientations of the four edge-connected faces be $\mathbf{e}_1, \mathbf{e}_2, \mathbf{e}_3, \mathbf{e}_4$.
   - The normal is estimated based on the consensus of these orientations [S2, unofficial].

**Parameters:**
- Voxel Neighborhood Size: Not given in the sources. A larger neighborhood improves accuracy but increases computational cost [S1, unofficial].
- Fit Order: Linear or biquadratic. Biquadratic provides higher accuracy for curved surfaces [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Significant Error in Approximated Normal:** Since a voxel face has only six possible orientations, the error in the approximated normal can be significant if the face-based method is used without sufficient context [S2, unofficial].
- **Precision:** Floating-point precision in fitting the surface approximation may affect normal accuracy, especially for small neighborhoods [G].
- **Degenerate Input:** If the set of intersection points is insufficient (e.g., only one point), fitting a surface is impossible. A fallback to volume-space gradient estimation or a default normal may be required [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_normal_contextual(points, neighborhood_size, fit_order='linear'):
    """
    Compute normal using Normal-based Contextual Shading.
    
    Args:
        points: List of (x, y, z) tuples representing intersection points.
        neighborhood_size: Size of the voxel neighborhood to consider.
        fit_order: 'linear' or 'biquadratic'.
        
    Returns:
        Normal vector (nx, ny, nz).
    """
    if len(points) < 3:
        # Fallback: Use volume-space gradient if points are insufficient
        # This requires access to the volume data, which is not provided here.
        # Return a default normal or handle as error.
        return (0, 0, 1)  # Placeholder [G]

    # Step 1: Fit surface approximation
    if fit_order == 'linear':
        # Fit a plane z = ax + by + c to the points
        # Using least squares: [x y 1] * [a b c]^T = z
        A = []
        b = []
        for (x, y, z) in points:
            A.append([x, y, 1])
            b.append(z)
        # Solve for [a, b, c]
        # a, b, c = np.linalg.lstsq(A, b, rcond=None)[0]
        # Normal is (-a, -b, 1) normalized
        # Note: Actual linear algebra implementation omitted for brevity [G]
        a, b, c = 0, 0, 0  # Placeholder [G]
        normal = (-a, -b, 1)
    elif fit_order == 'biquadratic':
        # Fit a biquadratic surface
        # This involves solving a larger system of equations
        # Normal is derived from partial derivatives
        # Placeholder for complex fitting logic [G]
        normal = (0, 0, 1)
    else:
        raise ValueError("Unsupported fit order")

    # Normalize the normal vector
    length = sqrt(normal[0]**2 + normal[1]**2 + normal[2]**2)
    if length == 0:
        return (0, 0, 1)
    return (normal[0]/length, normal[1]/length, normal[2]/length)

def compute_normal_face_based(face_orientation, edge_connected_faces):
    """
    Compute normal based on face and edge-connected face orientations.
    
    Args:
        face_orientation: Orientation of the primary face (one of 6 directions).
        edge_connected_faces: List of orientations of four edge-connected faces.
        
    Returns:
        Normal vector.
    """
    # Examine the orientation of the face and the four edge-connected faces
    # The normal is estimated based on the consensus of these orientations
    # This is a simplified representation; actual logic depends on specific implementation [S2, unofficial]
    # Placeholder for face-based normal estimation [G]
    return face_orientation
```

## Subtopics

### [Depth-Gradient Shading](6618-depth-gradient-shading.md)
Depth-Gradient Shading is an image-based shading technique that approximates the surface gradient by calculating differences between the depths of a given pixel and its neighbors in the z-buffer. It serves as a computationally inexpensive alternative to volume-space gradient estimation for shading isosurfaces, avoiding the need to compute gradients in the original volume data. Unlike its parent, [Normal-based Contextual Shading](6617-normal-based-contextual-shading.md), which operates in object space to handle discontinuities robustly, Depth-Gradient Shading operates in image space and is prone to artifacts at object boundaries.

## Operating characteristics
- **Time Complexity:** Depends on the number of intersection points and the size of the voxel neighborhood. Fitting a linear function is $O(N)$ where $N$ is the number of points. Fitting a biquadratic function is more expensive, $O(N^2)$ or higher depending on the solver [S1, unofficial].
- **Memory Complexity:** Requires storing the intersection points and the voxel neighborhood. The memory usage is proportional to the number of pixels and the neighborhood size [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Strengths:** Provides more accurate normals than image-space methods by using object-space information. Can handle curved surfaces better with biquadratic fitting [S1, unofficial].
- **Limitations:** Computationally more expensive than image-space methods. The error in approximated normal can be significant if the face-based method is used due to limited face orientations [S2, unofficial].
- **When to Use:** When high-quality shading is required and computational resources are available. Suitable for medical data visualization where surface accuracy is critical [S1, unofficial].
- **When Not to Use:** When real-time performance is critical and lower-quality shading is acceptable. Image-space methods like Depth-Gradient Shading may be preferred [S1, unofficial].

## Background
Normal-based Contextual Shading was introduced as an object-space shading technique to improve upon image-space depth-gradient shading [S1, unofficial]. It is part of the broader field of volume visualization, which has been developed by researchers such as Arie Kaufman [S2, unofficial]. The technique is particularly relevant for medical data visualization, where accurate surface representation is crucial [S1, unofficial]. The method builds on the idea of fitting local surface approximations to intersection points, a concept that has been explored in various forms in volume rendering literature [S1, unofficial].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S2] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
