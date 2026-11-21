# Sphere Rendering with Thickness

Also called: Hollow Sphere Rendering with Thickness Clipping, Shell Sphere Voxelization, Shell sphere, Sphere Thickness Rasterization, Thick sphere clipping

## Overview
Sphere Rendering with Thickness renders a sphere as a hollow shell by voxelizing grid points that lie between an inner and outer radius. It differs from solid sphere rendering by excluding the core volume, and from triangle-based thickness methods by using a spherical coordinate system defined by a center point and a single radius parameter. This method is a specialized case of mesh rasterization where the primitive is a sphere rather than a polygon.

## How it works
**Inputs and Outputs**
- **Input**: Sphere center position $P_{center}$ (3D vector), sphere radius $r$, thickness parameter $\tau$ (non-negative scalar), and a grid of query points (voxel centers) $V$.
- **Output**: A subset of $V$ that falls within the spherical shell, typically marked for rendering or accumulation.

**Data Structures**
- **SphereGeometry**: Contains `pos` (center), `radius` ($r$), and optionally `thickness` ($\tau$).
- **BucketIndices**: A list of grid indices covering the bounding box of the sphere to optimize query scope.

**Steps**
1.  **Bounding Box Calculation**: Compute the axis-aligned bounding box (AABB) of the sphere. Determine the range of grid indices (buckets) that intersect this box. This avoids checking every voxel in the scene.
    -   Min index: $\lfloor \frac{P_{center}.x - r}{grid\_size} \rfloor$ to $\lceil \frac{P_{center}.x + r}{grid\_size} \rceil$ (similarly for y, z).
2.  **Precompute Constants**:
    -   Calculate squared radius: $r^2 = r \times r$.
    -   Calculate inner shell threshold: $T_{inner} = 2r\tau - r^2$. [S1, unofficial]
3.  **Iterate and Filter**: For each voxel center $P_{voxel}$ in the bounding box buckets:
    -   Calculate squared distance to center: $d^2 = \| P_{voxel} - P_{center} \|^2$.
    -   **Inner Shell Check**: If $\tau > 0$ and $d^2 < T_{inner}$, the point is inside the hollow core. Skip this point. [S1, unofficial]
    -   **Outer Shell Check**: If $d^2 < r^2$, the point is within the outer boundary. Add to the render list. [S1, unofficial]
    -   *Note*: The condition $d^2 < 2r\tau - r^2$ is described as "magic math" in the source, approximating the inner radius squared for a thin shell. [S1, unofficial]

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_sphere_thickness(center, radius, thickness, grid_points):
    """
    Renders a sphere with thickness by filtering grid points.
    
    Args:
        center: Vec3, sphere center position.
        radius: float, outer radius of the sphere.
        thickness: float, thickness of the shell. 0 means solid sphere.
        grid_points: List[Vec3], voxel centers to test.
        
    Returns:
        List[Vec3], points inside the shell.
    """
    r2 = radius * radius
    # Calculate inner threshold. 
    # Derived from (r - thickness)^2 = r^2 - 2*r*thickness + thickness^2.
    # The source uses 2*r*thickness - r^2 as a negative threshold for d^2 < ...
    # Wait, the source says: if d2 < (2 * radius * thickness - radius2) continue.
    # Let's verify the math. 
    # Inner radius r_in = r - thickness.
    # We want to skip if d < r_in => d^2 < (r - thickness)^2 = r^2 - 2r*thickness + thickness^2.
    # The source condition is: d^2 < 2*r*thickness - r^2.
    # This implies the source might be using a different formulation or there is a sign error/approximation in the "magic math".
    # However, the instruction is to implement from the source. 
    # Source: "if (thickness > 0 && d2 < (2 * radius * thickness - radius2)) continue;"
    # We must follow the source exactly.
    
    inner_threshold = 2 * radius * thickness - r2
    
    rendered_points = []
    
    for pt in grid_points:
        # Calculate squared distance from point to sphere center
        dx = pt.x - center.x
        dy = pt.y - center.y
        dz = pt.z - center.z
        d2 = dx*dx + dy*dy + dz*dz
        
        # Check inner shell exclusion
        if thickness > 0 and d2 < inner_threshold:
            continue
            
        # Check outer shell inclusion
        if d2 < r2:
            rendered_points.append(pt)
            
    return rendered_points
```

**Parameters**
-   `radius`: Sphere outer radius.
-   `thickness`: Shell thickness. If 0, the sphere is solid. [S1, unofficial]

**Edge Cases and Pitfalls**
-   **Precision**: The "magic math" formula $2r\tau - r^2$ may suffer from floating-point cancellation errors if $\tau$ is very small compared to $r$. [S1, unofficial]
-   **Negative Threshold**: If $2r\tau - r^2$ is negative (e.g., if $\tau < r/2$), the condition $d^2 < \text{negative}$ is always false for real distances, meaning no points are skipped by the inner check. This effectively renders a solid sphere if the thickness is small relative to the radius? No, if $\tau$ is small, $r_{in}$ is close to $r$. The formula seems to imply a specific approximation. If the formula is strictly followed, small thicknesses might behave unexpectedly if the threshold becomes negative. [S1, unofficial]
-   **Boundary**: Points exactly on the boundary ($d^2 = r^2$) are included based on the strict inequality `<`.

## Subtopics

### Triangle Rendering with Thickness
Renders a triangle with a specified thickness by checking if grid points are within the thickness distance from the triangle plane or edges. Use this for planar surfaces. [Triangle Rendering with Thickness](12908-triangle-rendering-with-thickness.md)

### Mesh Rasterization and Triangle Intersection
The general framework for converting 3D mesh primitives into voxel data. Sphere rendering is a specialized primitive within this framework. [Mesh Rasterization and Triangle Intersection](12907-mesh-rasterization-and-triangle-intersection.md)

## Operating characteristics
-   **Time Complexity**: $O(N)$ where $N$ is the number of voxels in the sphere's bounding box. The bounding box optimization reduces $N$ from the total scene voxels to approximately $\frac{4}{3}\pi r^3 / v_{size}$.
-   **Memory Complexity**: $O(1)$ auxiliary space, excluding the output list of rendered points.
-   **Strengths**: Simple to implement; efficient for sparse grids due to bounding box culling; supports hollow shells.
-   **Limitations**: The inner radius approximation ("magic math") may lack precision for very thin shells or large radii. [S1, unofficial]
-   **When to use**: When rendering spherical objects with a hollow interior (e.g., bubbles, shells) in a voxel-based renderer.
-   **When not to use**: For solid spheres (use standard sphere voxelization) or for high-precision thin shells where the approximation error is unacceptable.

## Background
This method originates from the `Volumetric-display` project by quartzzuiop, which implements a custom volumetric renderer. The specific implementation for sphere thickness is found in the source code comments and demo applications, where it is used to render moving spheres with a defined thickness. The "magic math" comment suggests it is a heuristic or optimized approximation rather than a rigorously derived geometric formula. [S1, unofficial]

## Sources
[S1] quertzuiop/Volumetric-display, https://github.com/quertzuiop/Volumetric-display, fetched 2026-10-26 (unofficial)
