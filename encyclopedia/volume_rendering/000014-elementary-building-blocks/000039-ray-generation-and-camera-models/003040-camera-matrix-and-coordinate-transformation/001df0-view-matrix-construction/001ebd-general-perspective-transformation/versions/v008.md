# General Perspective Transformation

Also called: Eye-to-screen transformation, Perspective Transformation with Eye and Projection Plane Off-Origin

## Overview
This method defines a perspective projection where both the camera eye and the projection plane are displaced from the world origin, ensuring the Jacobian of the transformation is the identity at the center of the visibility bounding box (Vbb) [S1]. It addresses cases where a user-specified translation separates the Vbb center from the projection plane, a limitation of standard origin-centered projections [S1]. Unlike parallel projections that preserve relative screen space, this perspective method creates an illusion of depth by causing background objects to occupy less screen space than foreground objects of the same size [S2, unofficial].

## How it works
**Inputs:**
- Base eye distance parameter $e_o$ [S1].
- Z-translation parameter $z_t$ [S1].
- Visibility bounding box (Vbb) center (implicitly defines the identity Jacobian point) [S1].

**Outputs:**
- Transformed eye position [S1].
- Transformed projection plane equation [S1].

**Data Structures:**
- Standard 3D coordinate vectors and scalar parameters [G].

**Steps:**
1. **Define Base Geometry:** Establish the initial geometric basis where the projection plane is at $z=0$ and the Vbb is centered at $(0, 0, -e_o)$ before translation [S1].
2. **Apply Z-Translation:** The user applies a translation in the z-direction, denoted by $z_t$ [S1].
3. **Calculate Eye Position:** The translation moves the eye point to $(0, 0, e_o - z_t)$ [S1].
4. **Calculate Projection Plane:** The translation moves the projection plane to $z = -z_t$ [S1].
5. **Jacobian Property:** The specific placement of the eye and plane relative to the Vbb center ensures the Jacobian of the transformation is the identity matrix at that center [S1].

**Parameters:**
- $e_o$: Base eye distance parameter [S1].
- $z_t$: User-specified translation in z [S1].

**Edge Cases and Pitfalls:**
- If the Vbb center is not properly aligned with the identity Jacobian point, the transformation may introduce distortion [S1].
- Standard perspective transformations where the projection plane passes through the origin are insufficient for scenes where the center of the screen-z values is near the center of the Vbb but the Vbb itself is far from the origin [S1].

## Subtopics

### [Numerically Stable Perspective Transformation](2008-numerically-stable-perspective-transformation.md)
This variant prevents floating-point errors in screen-z values that occur when translating vertices far from the origin in standard perspective projections, which otherwise cause ordering inconsistencies during the compositing of thousands of closely spaced triangles. It differs from standard perspective transformations by positioning the eye away from the world origin, thereby establishing a geometric basis where relative errors are minimized without requiring double-precision arithmetic.

### [Custom Perspective Projection Matrix Construction](2175-custom-perspective-projection-matrix-construction.md)
This method constructs the perspective projection matrix and view matrix required to transform 3D world coordinates into clip space for volume rendering and rasterization. It solves the problem of defining the camera's field of view, aspect ratio, and depth clipping planes in a way that is compatible with specific rendering pipelines. It differs from generic graphics pipelines by explicitly managing intrinsic scaling (focal lengths) for multi-resolution training and constructing a specific projection matrix tailored for the near/far plane constraints of the rendering engine. It builds on [General Perspective Transformation](1845-general-perspective-transformation.md).

### Film Shift
This variant constrains the film position by the refraction of the ray connecting the lens to the pivot [S3]. The shifted film interaction position $x'_{\text{film}}$ is evaluated using $x'_{\text{lens}}$ and $x'_{\text{pivot}}$ [S3]. The change of measure between the base film interaction and the shifted one is accounted for with the Jacobian:
$$ |S'_{i \rightarrow j}(x_{\text{film}})| = \frac{g(x_{\text{lens}}, x_{\text{pivot}})}{g(x_{\text{lens}}, x_{\text{film}})} \times \frac{g(x'_{\text{lens}}, x'_{\text{pivot}})}{g(x'_{\text{lens}}, x'_{\text{film}})} $$
where $g(a,b) = D(a,b) / |b-a|^2$ and $D(a,b) = |\tilde{\boldsymbol{u}}_{ab} \times \tilde{\boldsymbol{n}}_b|$ if $b$ is on a surface, else $D(a,b) = 1$ [S3].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Ensures the Jacobian is the identity at the Vbb center, maintaining accurate geometric correspondence for volume rendering [S1].
- **Limitations:** Requires careful calculation of eye and plane positions relative to the Vbb [S1].
- **When to use:** When rendering scenes where the center of interest (Vbb) is far from the world origin, and standard perspective projections separate the Vbb center from the projection plane [S1].

## Background
The method originates from work on hierarchical and parallelizable direct volume rendering for irregular and multiple grids [S1]. It was developed to address the limitations of standard perspective transformations where the projection plane passes through the origin, which was insufficient for maintaining precision in screen-z values for large visibility bounding boxes [S1]. The concept of perspective projection creating an illusion of depth is a fundamental principle in computer graphics, often implemented using functions like `glFrustum` in OpenGL-based tools like Smokeview [S2, unofficial].

## Sources
[S1] Hierarchical and parallelizable direct volume rendering for irregular and multiple grids, https://doi.org/10.1109/visual.1996.567606, fetched 2026-10-24
[S2] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S3] Volumetric Multi‐View Rendering, https://doi.org/10.1111/cgf.14481, fetched 2026-10-26
