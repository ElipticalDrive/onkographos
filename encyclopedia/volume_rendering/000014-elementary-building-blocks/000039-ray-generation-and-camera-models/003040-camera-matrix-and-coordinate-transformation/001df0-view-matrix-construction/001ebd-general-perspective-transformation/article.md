# General Perspective Transformation
Also called: Eye-to-screen transformation, Perspective Transformation with Eye and Projection Plane Off-Origin

## Overview
The General Perspective Transformation is a camera model where both the eye point and the projection plane are positioned away from the world origin [S1]. It solves the problem of numerical instability in screen-z values that occurs when standard perspective projections are applied to scenes where the visibility bounding box (Vbb) is far from the origin [S1]. By shifting the eye and projection plane, it ensures the Jacobian of the transformation is the identity at the center of the Vbb, minimizing floating-point errors during the compositing of closely spaced geometry [S1]. It differs from standard textbook perspective matrices by explicitly decoupling the eye position from the origin to optimize precision for specific scene bounds [S1].

## How it works
**Inputs:**
- $e_o$: The distance from the origin to the eye along the negative z-axis in the base coordinate system [S1].
- $z_t$: The user-specified translation in the z-direction [S1].
- Scene geometry defined within a Visibility Bounding Box (Vbb) [S1].

**Outputs:**
- Transformed vertex coordinates where the projection plane is located at $z = -z_t$ [S1].
- A coordinate mapping that transforms world coordinates to screen/clip space [S1].

**Data Structures:**
- Not given in the sources.

**Steps:**
1. **Define Base Geometry:** Establish the initial Vbb centered at $(0, 0, -e_o)$ in the base coordinate system [S1].
2. **Apply User Translation:** The user specifies a translation value $z_t$ [S1]. This translation effectively moves both the eye point and the projection plane in the z-direction [S1].
3. **Calculate Eye Position:** The eye point is positioned at $(0, 0, e_o - z_t)$ [S1].
4. **Calculate Projection Plane:** The projection plane is positioned at $z = -z_t$ [S1].
5. **Perform Projection:** Project 3D points onto the plane $z = -z_t$ using perspective division relative to the eye position [S1].

**Formulas:**
- Eye Position: $\mathbf{e} = (0, 0, e_o - z_t)$ [S1].
- Projection Plane: $z = -z_t$ [S1].
- Jacobian Property: The Jacobian of this transformation is the identity matrix at the center of the Vbb [S1].

**Parameters:**
- $e_o$: Base eye distance. Chosen such that the Vbb is centered at $(0, 0, -e_o)$ before translation [S1].
- $z_t$: User translation. Chosen to align the projection plane with the Vbb center to minimize error [S1].

**Edge Cases and Pitfalls:**
- If the Vbb center is not aligned with the projection plane after translation, the Jacobian will not be identity at the center, leading to potential ordering inconsistencies in compositing [S1].
- Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def general_perspective_transform(world_point, e_o, z_t):
    """
    Transforms a world point using the general perspective transformation.
    
    Args:
        world_point: Tuple (x, y, z) in world coordinates.
        e_o: Base eye distance parameter.
        z_t: User-specified z-translation.
        
    Returns:
        Tuple (x_screen, y_screen, z_clip) in clip space.
    """
    # 1. Define Eye Position
    eye_z = e_o - z_t
    
    # 2. Define Projection Plane Z
    proj_plane_z = -z_t
    
    # 3. Calculate Vector from Eye to Point
    dx = world_point[0]
    dy = world_point[1]
    dz = world_point[2] - eye_z
    
    # 4. Perspective Projection
    # Distance from eye to projection plane is e_o
    focal_length = e_o
    
    if dz == 0:
        raise ValueError("Point is at the eye position")
        
    # Project onto plane z = -z_t
    x_screen = dx * (focal_length / dz)
    y_screen = dy * (focal_length / dz)
    
    # Map depth to clip space [0, 1] or [-1, 1]
    # The exact clip space mapping is not given in the sources.
    z_clip = dz
    
    return (x_screen, y_screen, z_clip)
```

## Subtopics

### [Numerically Stable Perspective Transformation](2008-numerically-stable-perspective-transformation.md)
This method prevents floating-point errors in screen-z values during the compositing of closely spaced triangles. It differs from standard perspective transformations by positioning the eye away from the world origin to minimize relative errors without requiring double-precision arithmetic. It is chosen when rendering large scenes with high-precision depth requirements. See [Numerically Stable Perspective Transformation](2008-numerically-stable-perspective-transformation.md).

### [Custom Perspective Projection Matrix Construction](2175-custom-perspective-projection-matrix-construction.md)
This method constructs the projection and view matrices required to transform 3D scene coordinates into clip space for rendering. It differs from standard symmetric frustum constructors by allowing explicit control over non-square focal lengths ($f_x, f_y$) and supporting viewport shift vectors to align the camera with a visibility bounding box. It is chosen when rendering requires explicit control over focal lengths and viewport alignment. See [Custom Perspective Projection Matrix Construction](2175-custom-perspective-projection-matrix-construction.md).

### Film Shift
Film shift constrains the film position by the refraction of the ray connecting the lens to the pivot. It is chosen when simulating optical refraction effects in multi-view rendering pipelines. See [Film Shift](2008-numerically-stable-perspective-transformation.md).

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Eliminates the need for double-precision arithmetic in many volume rendering scenarios by keeping the Jacobian identity at the Vbb center [S1].
- **Limitations:** Requires knowledge of the Vbb center to set $e_o$ and $z_t$ appropriately [S1].
- **When to use:** High-precision volume rendering, especially with irregular grids or multiple grids where depth ordering is critical [S1].
- **When not to use:** Not given in the sources.

## Background
The method originates from work on hierarchical and parallelizable direct volume rendering for irregular and multiple grids [S1]. It was developed to address the limitations of standard perspective transformations where the eye is fixed at the origin, which causes numerical instability when the scene geometry is far from the origin [S1]. The technique is used in volume rendering applications that require high precision in depth compositing, such as scientific visualization of large datasets [S1].

## Sources
[S1] Hierarchical and parallelizable direct volume rendering for irregular and multiple grids, https://doi.org/10.1109/visual.1996.567606, fetched 2026-10-24
[S2] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S3] Volumetric Multi‐View Rendering, https://doi.org/10.1111/cgf.14481, fetched 2026-10-26
