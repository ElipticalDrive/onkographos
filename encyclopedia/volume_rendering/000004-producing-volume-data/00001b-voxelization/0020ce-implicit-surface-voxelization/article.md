# Implicit Surface Voxelization
Also called: Iso-surface voxelization

## Overview
Implicit Surface Voxelization converts a mathematical implicit surface definition into a discrete voxel grid by rasterizing 2D rectangles within a Region of Interest (ROI) and evaluating the implicit function in the pixel shader [S1, S2, unofficial]. It differs from triangle-based rasterization methods (e.g., [Rasterization-based Voxelization](908-rasterization-based-voxelization.md)) by operating directly on analytical functions rather than polygonal approximations, allowing for the handling of complex, non-polygonal geometry [S1, S2, unofficial].

## How it works
**Inputs:**
1.  An implicit surface function $f(\mathbf{p})$ defining the geometry, where $\mathbf{p}$ is a 3D point [S1, S2, unofficial].
2.  A Region of Interest (ROI) bounding the surface [S1, S2, unofficial].
3.  A voxel grid resolution and coordinate system [S1, S2, unofficial].
4.  A voxelization mode: Binary (extracting the iso-surface) or Solid (finding intervals) [S1, S2, unofficial].

**Outputs:**
A voxel grid where occupied voxels correspond to points satisfying the implicit surface conditions [S1, S2, unofficial].

**Data Structures:**
*   **Render Target:** A 2D texture or buffer where each pixel maps to a specific 3D coordinate in the voxel grid space [S1, S2, unofficial].
*   **Worksheet:** The final voxel data structure storing the output [S1, S2, unofficial].

**Steps:**
1.  **Generate Rectangles:** Create a sequence of 2D rectangles embedded within the computed ROI of the target surface [S1, S2, unofficial].
2.  **Rasterize:** Rasterize each rectangle to the render target from front to back [S1, S2, unofficial].
3.  **Evaluate:** For each pixel in the render target, retrieve its 2D coordinates and map them to a 3D point $\mathbf{p}$ in the voxel space [S1, S2, unofficial]. Evaluate the implicit function $f(\mathbf{p})$ [S1, S2, unofficial].
4.  **Condition Check:** Determine if the evaluation result satisfies the voxelization condition [S1, S2, unofficial].
    *   **Binary Voxelization:** Extract the defined iso-surface, typically where $f(\mathbf{p}) = 0$ [S1, S2, unofficial].
    *   **Solid Voxelization:** Find an interval that satisfies a specific inequality (e.g., $f(\mathbf{p}) \leq 0$ for the interior) [S1, S2, unofficial].
5.  **Output:** If the condition is met, mark the corresponding voxel as occupied and transform the data to the worksheet [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def implicit_surface_voxelization(implicit_func, roi, resolution, mode='binary'):
    """
    Voxelize an implicit surface by rasterizing rectangles and evaluating the function.
    
    Args:
        implicit_func: Function f(p) -> float, where p is a 3D vector.
        roi: Bounding box or list of rectangles defining the region of interest.
        resolution: Tuple (x, y, z) defining the voxel grid dimensions.
        mode: 'binary' to extract iso-surface, 'solid' to fill interior.
        
    Returns:
        A 3D boolean array or voxel grid representing the surface.
    """
    # Initialize empty voxel grid
    voxel_grid = create_empty_grid(resolution)
    
    # Step 1: Generate sequence of rectangles embedded in ROI
    # Note: The specific algorithm for generating these rectangles is not detailed in sources [S1, S2, unofficial].
    # It assumes the host application computes the ROI.
    rectangles = generate_rectangles_in_roi(roi)
    
    # Step 2: Rasterize each rectangle to a render target
    # The render target maps 2D pixels to 3D points in the voxel space.
    render_target = initialize_render_target(resolution)
    
    for rect in rectangles:
        # Rasterize rectangle from front to back
        # This step populates the render target with pixel coordinates
        # corresponding to 3D points covered by the rectangle.
        rasterize_rectangle_to_target(rect, render_target)
        
    # Step 3 & 4: Evaluate implicit function and check conditions
    for pixel in render_target:
        # Get 3D point p corresponding to the pixel
        p = pixel_to_3d_point(pixel, resolution)
        
        # Evaluate implicit function
        value = implicit_func(p)
        
        is_occupied = False
        if mode == 'binary':
            # Binary voxelization: extract defined iso-surface
            # Typically f(p) == 0, but sources say "satisfies given conditions"
            # [G] Assuming standard zero-crossing for iso-surface extraction
            if abs(value) < epsilon:
                is_occupied = True
        elif mode == 'solid':
            # Solid voxelization: find interval satisfying inequality
            # [G] Assuming f(p) <= 0 for interior of closed surface
            if value <= 0:
                is_occupied = True
                
        # Step 5: Output if condition satisfied
        if is_occupied:
            # Transform to worksheet (voxel grid)
            voxel_grid[pixel.grid_index] = True
            
    return voxel_grid
```

**Parameters:**
*   **Epsilon:** A small tolerance for binary voxelization equality checks. Not given in the sources.
*   **Instruction Slots:** The method's capability to handle complex surfaces depends on the shader program's instruction slot limit [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Shader Limits:** Complex implicit surfaces may exceed the instruction slot limits of the shader program, limiting the complexity of surfaces that can be handled [S1, S2, unofficial].
*   **ROI Computation:** The method relies on the host application computing the ROI correctly; incorrect ROI definition may lead to missed voxels or excessive computation [S1, S2, unofficial].

## Subtopics

### Analytical Implicit Voxelization
An approach to rendering general-form implicit surfaces by sampling an analytical implicit function into a regular grid via voxelization, followed by direct volume rendering [S3, unofficial]. It is chosen when subvoxel surface detail is critical or when the implicit function is defined analytically rather than as a mesh [S3, unofficial]. It differs from standard implicit surface voxelization by explicitly coupling the voxelization step with direct volume rendering techniques [S3, unofficial].

### Smooth Visual Hull Reconstruction
Voxelization of a surface defined as the product of $n$ filtered silhouette images, where the surface is the isosurface $S(\mathbf{p}) = 0.5$ [S4]. It is used in image-based modeling to reconstruct 3D shapes from multiple 2D camera views [S4]. The surface is defined by:
$$S(\mathbf{p}) = \prod_i \mathcal{I}_i(\mathcal{C}_i(\mathbf{p}))$$
where $\mathcal{I}_i: \mathbb{R}^2 \rightarrow [0, 1]$ is the filtered silhouette image for camera $i$, and $\mathcal{C}_i: \mathbb{R}^3 \rightarrow \mathbb{R}^2$ is the projective transform from NVC (Normalized Device Coordinates) space to image space for camera $i$ [S4]. This method provides a smooth normal field $\nabla S(\mathbf{p})$ over the surface if the silhouette images are smooth [S4].

## Operating characteristics
*   **Time Complexity:** Depends on the number of rectangles generated and the resolution of the render target. The evaluation of the implicit function is performed per pixel [S1, S2, unofficial].
*   **Memory Complexity:** Requires memory for the render target and the final voxel grid [S1, S2, unofficial].
*   **Performance:** Can handle very complex implicit surfaces if the shader supports sufficient instruction slots [S1, S2, unofficial]. Performance is hardware-dependent, relying on GPU rasterization capabilities [S1, S2, unofficial].
*   **Strengths:** Handles non-polygonal, complex implicit surfaces directly; suitable for real-time applications if shader limits are respected [S1, S2, unofficial].
*   **Limitations:** Limited by shader instruction slot length; requires explicit ROI computation by the host application [S1, S2, unofficial].
*   **When to Use:** When the geometry is defined by an implicit function rather than a triangle mesh, and real-time voxelization is required [S1, S2, unofficial].
*   **When Not to Use:** When the implicit function is too complex for the available shader instruction slots, or when the ROI is difficult to compute [S1, S2, unofficial].

## Background
The method was proposed in the context of real-time voxelization for complex models, distributing workload between CPU and GPU [S1, S2, unofficial]. It is part of a broader set of techniques for voxelizing various surface forms, including parametric surfaces [S1, S2, unofficial]. The approach leverages GPU rasterization pipelines to evaluate implicit functions in parallel [S1, S2, unofficial].

## Sources
[S1] Real-time Voxelization for Complex Models, https://wo1fsea.github.io/files/About_NavMesh/P2_Real-time_Voxelization_for_Complex_Models.pdf, fetched 2026-10-29 (unofficial)
[S2] Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
[S3] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
[S4] Real-time high-resolution sparse voxelization with application to image-based modeling, https://doi.org/10.1145/2492045.2492053, fetched 2026-11-01
