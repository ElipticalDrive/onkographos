# Implicit Surface Voxelization
Also called: Iso-surface voxelization

## Overview
Implicit Surface Voxelization converts a mathematical implicit surface definition into a discrete voxel representation by rasterizing 2D rectangles within a Region of Interest (ROI) and evaluating the implicit function in a pixel shader [S1, unofficial]. It differs from triangle-based rasterization methods (e.g., [Rasterization-based Voxelization](908-rasterization-based-voxelization.md)) by operating directly on analytical functions rather than polygonal approximations, allowing it to handle complex, non-polygonal geometry [S1, unofficial].

## How it works
**Inputs:**
1. An implicit surface function $f(\mathbf{p})$ defining the geometry, where $\mathbf{p}$ is a 3D point [S1, unofficial].
2. A Region of Interest (ROI) bounding the surface [S1, unofficial].
3. A voxel grid resolution and coordinate system [S1, unofficial].
4. A voxelization mode: Binary (extracting the iso-surface) or Solid (finding intervals) [S1, unofficial].

**Outputs:**
A set of occupied voxels corresponding to points satisfying the implicit surface conditions [S1, unofficial].

**Data Structures:**
* **Render Target:** A 2D buffer where each pixel maps to a specific 3D coordinate in the voxel grid space [S1, unofficial].
* **Worksheet:** An intermediate transformation target for output pixels before final storage [S1, unofficial].

**Steps:**
1. **Generate Rectangles:** Create a sequence of 2D rectangles embedded within the computed ROI of the target surface [S1, unofficial].
2. **Rasterize:** Rasterize each rectangle to the render target from front to back [S1, unofficial].
3. **Evaluate:** For each pixel in the render target, retrieve its 2D coordinates and map them to a 3D point $\mathbf{p}$ in the voxel space [S1, unofficial]. Evaluate the implicit function $f(\mathbf{p})$ [S1, unofficial].
4. **Condition Check:** Determine if the evaluation result satisfies the voxelization condition [S1, unofficial].
    * **Binary Voxelization:** Extract the defined iso-surface, typically where $f(\mathbf{p}) = 0$ [S1, unofficial].
    * **Solid Voxelization:** Find an interval that satisfies a specific inequality (e.g., $f(\mathbf{p}) \leq 0$ for the interior) [S1, unofficial].
5. **Output:** If the condition is met, mark the corresponding voxel as occupied and transform the data to the worksheet [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def implicit_surface_voxelization(implicit_func, roi_rectangles, resolution, mode='binary', epsilon=1e-6):
    # Initialize render target buffer matching resolution
    render_target = [[None for _ in range(resolution[1])] for _ in range(resolution[0])]
    # Initialize output storage (worksheet)
    worksheet = []

    # Step 1 & 2: Generate and rasterize rectangles front-to-back
    # Sources state the host application computes the ROI and generates rectangles [S1, unofficial].
    # We assume roi_rectangles is sorted front-to-back.
    for rect in roi_rectangles:
        for y in range(resolution[1]):
            for x in range(resolution[0]):
                # Map pixel coordinates to 3D point in voxel space [G]
                p_x = (x + 0.5) * (roi_rectangles[0].width / resolution[0]) + roi_rectangles[0].origin_x
                p_y = (y + 0.5) * (roi_rectangles[0].height / resolution[1]) + roi_rectangles[0].origin_y
                p_z = rect.z_center
                p = (p_x, p_y, p_z)
                render_target[y][x] = p

    # Step 3 & 4: Evaluate implicit function and check conditions
    for y in range(resolution[1]):
        for x in range(resolution[0]):
            p = render_target[y][x]
            if p is None:
                continue
            value = implicit_func(p)
            is_occupied = False
            if mode == 'binary':
                # Binary voxelization extracts the defined iso-surface
                if abs(value) < epsilon:
                    is_occupied = True
            elif mode == 'solid':
                # Solid voxelization finds an interval satisfying an inequation
                if value <= 0:
                    is_occupied = True

            # Step 5: Output and transform to worksheet
            if is_occupied:
                worksheet.append(p)
    return worksheet
```

**Parameters:**
* **Epsilon:** A small tolerance for binary voxelization equality checks. Not given in the sources. [G]
* **Instruction Slots:** The method's capability to handle complex surfaces depends on the shader program's instruction slot limit [S1, unofficial].

**Edge Cases and Pitfalls:**
* **Shader Limits:** Complex implicit surfaces may exceed the instruction slot limits of the shader program, limiting the complexity of surfaces that can be handled [S1, unofficial].
* **ROI Computation:** The method relies on the host application computing the ROI correctly; incorrect ROI definition may lead to missed voxels or excessive computation [S1, unofficial].

## Subtopics

### Analytical Implicit Voxelization
An approach to rendering general-form implicit surfaces by sampling an analytical implicit function into a regular grid via voxelization, followed by direct volume rendering [S3, unofficial]. It is chosen when subvoxel surface detail is critical or when the implicit function is defined analytically rather than as a mesh [S3, unofficial]. It differs from standard implicit surface voxelization by explicitly coupling the voxelization step with direct volume rendering techniques [S3, unofficial].

### Smooth Visual Hull Reconstruction
Voxelization of a surface defined as the product of $n$ filtered silhouette images, where the surface is the isosurface $S(\mathbf{p}) = 0.5$ [S4]. It is used in image-based modeling to reconstruct 3D shapes from multiple 2D camera views [S4]. The surface is defined by:
$$S(\mathbf{p}) = \prod_i \mathcal{I}_i(\mathcal{C}_i(\mathbf{p}))$$
where $\mathcal{I}_i: \mathbb{R}^2 \rightarrow [0, 1]$ is the filtered silhouette image for camera $i$, and $\mathcal{C}_i: \mathbb{R}^3 \rightarrow \mathbb{R}^2$ is the projective transform from NVC (Normalized Device Coordinates) space to image space for camera $i$ [S4]. This method provides a smooth normal field $\nabla S(\mathbf{p})$ over the surface if the silhouette images are smooth [S4].

## Operating characteristics
* **Time Complexity:** Proportional to the number of rectangles generated and the resolution of the render target, with one implicit function evaluation per pixel [S1, unofficial].
* **Memory Complexity:** Requires memory for the render target buffer and the worksheet transformation target [S1, unofficial].
* **Performance:** Can handle very complex implicit surfaces if the shader supports sufficient instruction slots [S1, unofficial]. Performance is hardware-dependent, relying on GPU rasterization capabilities [S1, unofficial].
* **Strengths:** Handles non-polygonal, complex implicit surfaces directly [S1, unofficial].
* **Limitations:** Limited by shader instruction slot length; requires explicit ROI computation by the host application [S1, unofficial].
* **When to Use:** When the geometry is defined by an implicit function rather than a triangle mesh [S1, unofficial].
* **When Not to Use:** When the implicit function is too complex for the available shader instruction slots, or when the ROI is difficult to compute [S1, unofficial].

## Background
The method was proposed in the context of real-time voxelization for complex models, distributing workload between CPU and GPU [S1, unofficial]. It is part of a broader set of techniques for voxelizing various surface forms, including parametric surfaces [S1, unofficial]. The approach leverages GPU rasterization pipelines to evaluate implicit functions in parallel [S1, unofficial].

## Sources
[S1] Real-time Voxelization for Complex Models, https://wo1fsea.github.io/files/About_NavMesh/P2_Real-time_Voxelization_for_Complex_Models.pdf, fetched 2026-10-29 (unofficial)
[S2] Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
[S3] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
[S4] Real-time high-resolution sparse voxelization with application to image-based modeling, https://doi.org/10.1145/2492045.2492053, fetched 2026-11-01
