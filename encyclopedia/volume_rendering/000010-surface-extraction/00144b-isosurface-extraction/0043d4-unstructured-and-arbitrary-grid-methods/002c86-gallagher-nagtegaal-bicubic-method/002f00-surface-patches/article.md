# Surface Patches

Also called: B-spline surfaces, Bi-cubic surfaces, Bicubic Fit Surface Rendering, Catmull subdivision, Non-linear polynomial surface representation

## Overview
Surface patches are a class of surface representation techniques that use non-linear polynomials or splines to map a given surface, allowing for the natural rendering of curved geometry [S1, unofficial]. They are conceptually simple and serve as a smooth alternative to polygonal mesh approximations in isosurface extraction [S1, unofficial]. Unlike polygonal methods that approximate curvature with flat facets, surface patches define the surface analytically, enabling high-quality shading and hidden-line removal without the visual artifacts of low tessellation [S1, unofficial].

## How it works
**Inputs:** A set of control points (nodes) defining the surface geometry. These control points are typically derived from the underlying volumetric data using tessellation methods or by fitting the polynomial to discrete data points [S1, unofficial][S2, unofficial].

**Outputs:** A rendered image of the smooth surface, potentially with shading and hidden-line removal.

**Data Structures:**
The core data structure is the control point grid. For bicubic surfaces, this is a $4 \times 4$ grid of control points defining a single patch. The points are stored as 3D coordinates $(x, y, z)$. No specific bit-packing is mandated; standard floating-point or fixed-point representations are used depending on precision requirements [S1, unofficial].

**Parameters:**
- **Polynomial Family:** The most widely used mappings are B-splines and bi-cubic families [S1, unofficial].
- **Subdivision Depth:** Determined by pixel coverage criteria.

**Steps:**
1.  **Define Control Points:** Establish the control points for the surface patch. If the input is discrete data (e.g., from a grid or radar cross-section data), apply a fitting or tessellation method to generate these nodes [S1, unofficial][S2, unofficial].
2.  **Evaluate Surface:** Define the surface using the chosen polynomial basis (e.g., bicubic B-spline). The surface is defined by a function $S(u, v)$ where $u, v \in [0, 1]$ are parametric coordinates.
3.  **Subdivide for Rendering:** To eliminate cracking between patches and ensure smooth rendering, apply recursive subdivision. The specific procedure is Catmull's subdivision algorithm [S1, unofficial].
    -   Recursively divide the parametric region into sub-patches.
    -   Continue subdivision until the transformed sub-region covers only one pixel on the screen [S1, unofficial].
4.  **Render:** Render the resulting sub-patches. Scan-line implementations of Catmull's algorithm, developed by Blinn and Lane-Carpenter, can be used for efficient rasterization [S1, unofficial].
5.  **Shading and Visibility:** Apply surface-normal based shading algorithms to the rendered patches. Hidden-line removal can be performed if required [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Cracking:** Without recursive subdivision, adjacent patches may not join seamlessly, leading to visible cracks [S2, unofficial].
-   **Precision:** High-order polynomials can suffer from numerical instability if control points are poorly conditioned.
-   **Boundary Conditions:** Ensuring $C^1$ or $C^2$ continuity across patch boundaries requires careful selection of control points at the edges [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_surface_patches(control_points, camera, screen_resolution):
    """
    Renders a surface defined by control points using Catmull subdivision.
    control_points: List of 3D vectors defining the patch.
    camera: Camera object with projection matrix.
    screen_resolution: Tuple (width, height).
    """
    # Step 1: Define the parametric surface function S(u, v)
    # For bicubic B-spline, this involves basis matrix multiplication.
    # S(u, v) = U * M * ControlPoints * M^T * V^T
    # where U = [u^3, u^2, u, 1], V = [v^3, v^2, v, 1]
    # M is the bicubic basis matrix.
    
    # Step 2: Recursive Subdivision (Catmull's Algorithm)
    def subdivide(patch, min_pixel_area):
        # Project the current patch's bounding box to screen space
        screen_bbox = project_to_screen(patch)
        
        # Check if the patch covers less than or equal to one pixel
        if screen_bbox.width <= 1.0 and screen_bbox.height <= 1.0:
            # Render the patch as a flat polygon or evaluate normal for shading
            render_patch(patch)
            return
        
        # Recursively divide the patch into 4 sub-patches
        sub_patches = split_patch(patch)
        for sub_patch in sub_patches:
            subdivide(sub_patch, min_pixel_area)

    # Initial call with the root patch
    root_patch = create_patch_from_control_points(control_points)
    subdivide(root_patch, 1.0)

def split_patch(patch):
    """
    Splits a bicubic patch into 4 smaller bicubic patches.
    Returns a list of 4 new control point grids.
    """
    # Catmull-Clark or similar subdivision rules apply here.
    # For bicubic B-splines, this involves computing new control points
    # at the midpoints and edges of the parametric domain.
    # This is a standard geometric subdivision operation.
    # Implementation details depend on the specific basis matrix used.
    # [G] Standard bicubic subdivision computes new control points by averaging
    # existing ones according to the basis function's refinement mask.
    return [patch1, patch2, patch3, patch4]

def render_patch(patch):
    """
    Renders a single sub-patch with shading.
    """
    # Compute surface normal at the center of the patch
    normal = compute_normal(patch)
    # Apply lighting model
    color = apply_shading(normal, light_direction)
    # Draw the polygon corresponding to the patch
    draw_polygon(patch.vertices, color)
```

## Subtopics

### Bicubic Fit
A specific instance of surface patches using polynomials cubic in both parameters. It is joined through iterative evaluation. Suitable for smooth visualization of discrete data like radar cross-sections [S2, unofficial].

### B-Spline Surfaces
A widely used mapping family for surface patches. Offers local control and smoothness. Preferred when local modification of the surface shape is required [S1, unofficial].

### Catmull Subdivision
The recursive subdivision procedure used to display non-linear patches. It ensures that the surface is rendered with sufficient detail to cover only one pixel per sub-region, eliminating cracks [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** The rendering time depends on the depth of subdivision required to achieve pixel-level coverage. In the worst case, this is proportional to the number of pixels, but adaptive subdivision reduces this for smooth regions.
-   **Memory Complexity:** Storage is proportional to the number of control points. Subdivision is typically done on-the-fly, so memory usage is low compared to storing a full tessellation.
-   **Strengths:** Natural representation of curved surfaces; conceptually simple; supports hidden-line removal and smooth shading [S1, unofficial].
-   **Limitations:** Requires recursive subdivision to avoid cracking; may be computationally expensive for very complex surfaces if not optimized; less flexible than polygonal meshes for arbitrary topology [S1, unofficial][S2, unofficial].
-   **When to Use:** When visualizing smooth, continuous surfaces from volumetric or discrete data where polygonal approximation artifacts are unacceptable [S2, unofficial].
-   **When Not to Use:** For highly complex or non-smooth geometries where polygonal meshes are more appropriate; when real-time performance is critical and subdivision overhead is too high.

## Background
Surface patches were introduced as a method to render curved surfaces naturally using non-linear polynomials or splines [S1, unofficial]. The technique was developed in the context of 3D reconstruction and display systems, such as THREED (Three Dimensional Reconstruction And Display) [S1, unofficial]. Catmull devised the subdivision procedure for displaying non-linear patches, and scan-line implementations were later developed by Blinn and Lane-Carpenter [S1, unofficial]. The method has been applied in medical data visualization and radar cross-section data viewing [S1, unofficial][S2, unofficial].

## Sources
[S1] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA258905: Methods for Viewing Radar Cross Section Data in Three Dimensions, https://archive.org/details/DTIC_ADA258905, fetched 2026-10-26 (unofficial)
