# General Perspective Transformation
Also called: Eye-to-screen transformation, Perspective Transformation with Eye and Projection Plane Off-Origin

## Overview
The General Perspective Transformation is a camera model where both the eye point and the projection plane are positioned away from the world origin [S1]. It solves the problem of numerical instability in screen-z values that occurs when standard perspective projections (with the eye at the origin) are applied to scenes where the visibility bounding box (Vbb) is far from the origin [S1]. By shifting the eye and projection plane, it ensures the Jacobian of the transformation is the identity at the center of the Vbb, minimizing floating-point errors during the compositing of closely spaced geometry [S1]. It differs from standard textbook perspective matrices by explicitly decoupling the eye position from the origin to optimize precision for specific scene bounds [S1].

## How it works
**Inputs:**
- $e_o$: The distance from the origin to the eye along the negative z-axis in the base coordinate system [S1].
- $z_t$: The user-specified translation in the z-direction [S1].
- Scene geometry defined within a Visibility Bounding Box (Vbb) [S1].

**Outputs:**
- A 4x4 transformation matrix (or equivalent coordinate mapping) that transforms world coordinates to screen/clip space [S1].
- Transformed vertex coordinates where the projection plane is located at $z = -z_t$ [S1].

**Data Structures:**
- Standard 4x4 homogeneous transformation matrix. The specific layout depends on the implementation language, but conceptually it combines a translation and a perspective projection.

**Steps:**
1. **Define Base Geometry:** Establish the initial visibility bounding box (Vbb) centered at $(0, 0, -e_o)$ in the base coordinate system [S1].
2. **Apply User Translation:** The user specifies a translation value $z_t$ [S1]. This translation effectively moves both the eye point and the projection plane in the z-direction [S1].
3. **Calculate Eye Position:** The eye point is positioned at $(0, 0, e_o - z_t)$ [S1].
4. **Calculate Projection Plane:** The projection plane is positioned at $z = -z_t$ [S1].
5. **Perform Projection:** Project 3D points onto the plane $z = -z_t$ using standard perspective division relative to the eye position [S1].

**Formulas:**
- Eye Position: $\mathbf{e} = (0, 0, e_o - z_t)$ [S1].
- Projection Plane: $z = -z_t$ [S1].
- Jacobian Property: The Jacobian of this transformation is the identity matrix at the center of the Vbb [S1].

**Parameters:**
- $e_o$: Base eye distance. Typical value depends on scene scale; chosen such that the Vbb is centered at $(0, 0, -e_o)$ before translation [S1].
- $z_t$: User translation. Chosen to align the projection plane with the Vbb center to minimize error [S1].

**Edge Cases and Pitfalls:**
- If the Vbb center is not aligned with the projection plane after translation, the Jacobian will not be identity at the center, leading to potential ordering inconsistencies in compositing [S1].
- Standard perspective transformations with the eye at the origin cause significant floating-point errors in screen-z values when vertices are far from the origin [S1]. This method mitigates that by moving the eye.

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
        Tuple (x_screen, y_screen, z_screen) in normalized device coordinates 
        or clip space, depending on subsequent pipeline steps.
    """
    # 1. Define Eye Position
    eye_x = 0.0
    eye_y = 0.0
    eye_z = e_o - z_t
    
    # 2. Define Projection Plane Z
    proj_plane_z = -z_t
    
    # 3. Calculate Vector from Eye to Point
    dx = world_point[0] - eye_x
    dy = world_point[1] - eye_y
    dz = world_point[2] - eye_z
    
    # 4. Perspective Projection
    # Standard perspective projection divides by depth relative to eye.
    # However, the specific screen coordinates depend on the focal length 
    # and the distance to the projection plane.
    # In the context of [S1], the goal is to map to a plane at z = -z_t.
    # The depth for division is typically the distance from the eye to the point 
    # along the view axis (z-axis in eye space).
    
    # Note: [S1] emphasizes the *positioning* of eye and plane to fix Jacobian.
    # The actual projection formula is standard perspective:
    # x' = x * (f / z_depth)
    # y' = y * (f / z_depth)
    # where z_depth is the distance from eye to point along view vector.
    
    # Assuming view vector is -Z (standard OpenGL-like or [S1] implied basis):
    z_depth = -dz  # Distance from eye to point along view direction
    
    if z_depth == 0:
        raise ValueError("Point is at the eye position")
        
    # Focal length f is implicitly handled by the projection plane distance 
    # and the desired field of view. In [S1], the focus is on the translation 
    # to fix numerical precision.
    # For a generic implementation, we assume a standard perspective divide.
    # The screen coordinates are relative to the projection plane.
    
    # To implement strictly from [S1], we note that the transformation 
    # ensures the Jacobian is identity at Vbb center.
    # A common way to achieve the "general" aspect is to construct a matrix.
    # Here we compute the projected coordinates directly.
    
    # Assuming unit focal length or normalized coordinates for simplicity 
    # as [S1] does not specify f explicitly, but implies a geometric basis.
    # We return the projected x, y and the depth z for further clipping.
    
    x_screen = dx / z_depth
    y_screen = dy / z_depth
    
    # The z_screen value is critical for depth buffering/compositing.
    # It is mapped to the range [0, 1] or [-1, 1] depending on the API.
    # [S1] focuses on the *value* of screen-z to prevent ordering errors.
    # We return the raw depth for the caller to map to clip space.
    z_screen = z_depth
    
    return (x_screen, y_screen, z_screen)
```

## Subtopics

### [Numerically Stable Perspective Transformation](2008-numerically-stable-perspective-transformation.md)
This method prevents floating-point errors in screen-z values that occur when translating vertices far from the origin in standard perspective projections, which otherwise cause ordering inconsistencies during the compositing of thousands of closely spaced triangles. It differs from standard perspective transformations by positioning the eye away from the world origin, thereby establishing a geometric basis where relative errors are minimized without requiring double-precision arithmetic. It is chosen when rendering large scenes with high-precision depth requirements where standard eye-at-origin models fail. See [Numerically Stable Perspective Transformation](2008-numerically-stable-perspective-transformation.md).

### [Custom Perspective Projection Matrix Construction](2175-custom-perspective-projection-matrix-construction.md)
This method constructs the specific projection and view matrices required to transform 3D scene coordinates into clip space for rendering, specifically tailored for Gaussian Splatting pipelines. It differs from standard textbook perspective matrices by allowing explicit control over focal lengths ($f_x, f_y$) and handling viewport shifts to align the camera center with the visibility bounding box (Vbb). It builds on [General Perspective Transformation](1845-general-perspective-transformation.md) by providing the concrete matrix assembly steps for a custom camera model. See [Custom Perspective Projection Matrix Construction](2175-custom-perspective-projection-matrix-construction.md).

### Film Shift
Film shift constrains the film position by the refraction of the ray connecting the lens to the pivot. The shifted film interaction position $x'_{\text{film}}$ is evaluated using $x'_{\text{lens}}$ and $x'_{\text{pivot}}$ [S3]. The change of measure is accounted for with the Jacobian:
$$|S'_{i \rightarrow j}(x_{\text{film}})| = \frac{g(x_{\text{lens}}, x_{\text{pivot}})}{g(x_{\text{lens}}, x_{\text{film}})} \times \frac{g(x'_{\text{lens}}, x'_{\text{pivot}})}{g(x'_{\text{lens}}, x'_{\text{film}})}$$
where $g(a,b) = D(a,b) / |b-a|^2$ and $D(a,b) = |\tilde{\boldsymbol{u}}_{ab} \times \tilde{\boldsymbol{n}}_b|$ if $b$ is on a surface, else $D(a,b) = 1$ [S3].

## Operating characteristics
- **Time Complexity:** $O(1)$ per vertex for the transformation itself.
- **Memory Complexity:** $O(1)$ additional memory for the transformation matrix/parameters.
- **Performance:** Significantly reduces precision errors in depth sorting for large scenes compared to standard eye-at-origin models [S1].
- **Strengths:** Eliminates the need for double-precision arithmetic in many volume rendering scenarios by keeping the Jacobian identity at the Vbb center [S1].
- **Limitations:** Requires knowledge of the Vbb center to set $e_o$ and $z_t$ appropriately [S1]. Not suitable for dynamic scenes where the Vbb changes rapidly without recomputation of parameters.
- **When to use:** High-precision volume rendering, especially with irregular grids or multiple grids where depth ordering is critical [S1].
- **When not to use:** Simple scenes where the Vbb is near the origin, or when standard OpenGL/DirectX pipelines with fixed-function matrices are sufficient and precision is not a bottleneck.

## Background
The method originates from work on hierarchical and parallelizable direct volume rendering for irregular and multiple grids [S1]. It was developed to address the limitations of standard perspective transformations where the eye is fixed at the origin, which causes numerical instability when the scene geometry is far from the origin [S1]. The technique is used in volume rendering applications that require high precision in depth compositing, such as scientific visualization of large datasets [S1].

## Sources
[S1] Hierarchical and parallelizable direct volume rendering for irregular and multiple grids, https://doi.org/10.1109/visual.1996.567606, fetched 2026-10-24
[S2] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S3] Volumetric Multi‐View Rendering, https://doi.org/10.1111/cgf.14481, fetched 2026-10-26
