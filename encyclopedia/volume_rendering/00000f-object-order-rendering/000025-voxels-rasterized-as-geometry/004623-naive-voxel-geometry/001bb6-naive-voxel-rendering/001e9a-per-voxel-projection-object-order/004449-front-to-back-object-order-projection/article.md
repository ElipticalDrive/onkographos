# Front-to-Back Object-Order Projection

Also called: Back-to-Front Algorithm, Back-to-Front Object-Order Projection, Front-to-Back (FTB) Algorithm, Object-Order Back-to-Front Projection

## Overview
Front-to-Back Object-Order Projection is a volume rendering technique that traverses voxels in increasing distance from the image plane, projecting each voxel onto the screen and compositing it with the current pixel value [S1, S2, unofficial]. It solves the problem of correct transparency and occlusion handling in object-order rendering by ensuring that nearer voxels are composited over farther ones, effectively implementing the painter's algorithm for volumetric data [S1, unofficial]. Unlike its sibling [Object-order Binary Volume Projection](6605-object-order-binary-volume-projection.md), which handles binary data, this method supports semi-transparent materials and allows for early ray termination or pixel locking, where once a pixel is sufficiently opaque, subsequent voxels projecting to that pixel are ignored [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D volume dataset $V$ with dimensions $D_x \times D_y \times D_z$, a viewing direction, and an image plane.
- Output: A 2D image where each pixel contains the accumulated color and opacity from the front-most voxels.

**Data Structures:**
- **Volume Data:** A 3D array of voxel values (opacity $\alpha$ and color $C$).
- **Image Buffer:** A 2D array storing the accumulated color $C_{acc}$ and opacity $\alpha_{acc}$ for each pixel. Initialized to $C_{acc} = (0,0,0,0)$ and $\alpha_{acc} = 0$.
- **Z-buffer (Optional):** For strict hidden voxel removal in non-orthographic projections, a depth buffer may be used, though standard FTB relies on traversal order [S1, unofficial].

**Steps:**
1. **Determine Traversal Order:** Identify the axis most parallel to the viewing direction. This axis should be the outermost loop to allow for meaningful partial image results during traversal [S2, unofficial].
2. **Initialize Image Buffer:** Set all pixel accumulators to zero.
3. **Traverse Voxels:** Iterate through the volume in increasing distance from the image plane (front-to-back) [S1, S2, unofficial].
   - For orthographic projections, a triple nested loop is sufficient: outer loop traverses planes, middle loop processes rows, inner loop visits voxels along the row [S3, book].
   - Note: This specific loop order does not yield a strict furthest-to-closest ordering for *all* voxels globally, but it ensures that voxels projecting to a *single pixel* are processed in the correct order for orthographic projections [S3, book].
4. **Project and Composite:** For each voxel $v$:
   - Project the voxel center (or footprint) onto the image plane to determine pixel coordinates $(u, v)$.
   - Retrieve the current accumulated color $C_{acc}$ and opacity $\alpha_{acc}$ at $(u, v)$.
   - Composite the voxel:
     $$ C_{new} = C_{acc} + C_v \cdot \alpha_v \cdot (1 - \alpha_{acc}) $$
     $$ \alpha_{new} = \alpha_{acc} + \alpha_v \cdot (1 - \alpha_{acc}) $$
   - Update the image buffer with $C_{new}$ and $\alpha_{new}$.
   - **Early Termination/Pixel Locking:** If $\alpha_{acc}$ reaches a threshold (e.g., 1.0), subsequent voxels projecting to this pixel can be ignored, as they are hidden by the front-most voxels [S2, unofficial].

**Parameters:**
- **Opacity Threshold:** A value close to 1.0 (e.g., 0.99) for early termination [G].
- **Kernel Size:** If using splatting (a variant of this approach), a Gaussian kernel with finite extent is used to distribute voxel energy across multiple pixels [S3, book].

**Edge Cases and Pitfalls:**
- **Perspective Holes:** In perspective projections, discrete selection of projected pixels can cause "holes" in the image as the camera moves closer, because neighboring voxels may project to increasingly distant pixels [S3, book]. Splatting addresses this by distributing energy across a footprint [S3, book].
- **Loop Order Limitation:** The standard triple-nested loop order (planes, rows, voxels) is only sufficient for orthographic projections. For perspective, a more complex sorting or z-buffering may be required to maintain strict front-to-back order per pixel [S3, book].
- **Clipping:** Clipping planes orthogonal to the major axes limit the traversal region. Clipping planes parallel to the image plane require ignoring samples whose distance to the image plane is less than the cut plane distance [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def front_to_back_projection(volume, image_width, image_height, view_direction):
    # Initialize image buffer
    # C_acc: [image_height, image_width, 4] (RGBA)
    # alpha_acc: [image_height, image_width]
    C_acc = zeros((image_height, image_width, 4))
    alpha_acc = zeros((image_height, image_width))

    # Determine traversal order based on view direction
    # Assume view_direction is a vector (dx, dy, dz)
    # Choose the axis with the smallest absolute component as the outermost loop
    # to ensure voxels are processed in increasing distance order
    abs_dx, abs_dy, abs_dz = abs(view_direction.x), abs(view_direction.y), abs(view_direction.z)
    
    if abs_dx <= abs_dy and abs_dx <= abs_dz:
        # X is most parallel to view, traverse X last (inner loop)
        # Traverse Z, then Y, then X
        order = ['z', 'y', 'x']
    elif abs_dy <= abs_dx and abs_dy <= abs_dz:
        # Y is most parallel to view, traverse Y last
        # Traverse Z, then X, then Y
        order = ['z', 'x', 'y']
    else:
        # Z is most parallel to view, traverse Z last
        # Traverse X, then Y, then Z
        order = ['x', 'y', 'z']

    # Get volume dimensions
    Dx, Dy, Dz = volume.shape

    # Helper to get voxel value and project to pixel
    # This is a simplified projection; real implementation requires affine transform
    def project_and_composite(voxel_pos, voxel_color, voxel_alpha):
        # Project voxel_pos to screen coordinates (u, v)
        u, v = project_to_screen(voxel_pos, view_direction)
        
        # Check bounds
        if not (0 <= u < image_width and 0 <= v < image_height):
            return

        # Get current accumulated values
        current_C = C_acc[v, u]
        current_alpha = alpha_acc[v, u]

        # Composite
        # C_new = C_acc + C_v * alpha_v * (1 - alpha_acc)
        # alpha_new = alpha_acc + alpha_v * (1 - alpha_acc)
        
        weight = voxel_alpha * (1 - current_alpha)
        new_C = current_C + voxel_color * weight
        new_alpha = current_alpha + weight

        # Update buffer
        C_acc[v, u] = new_C
        alpha_acc[v, u] = new_alpha

    # Traverse volume in front-to-back order
    # Note: The exact loop structure depends on the 'order' determined above
    # For simplicity, assuming a generic iteration that respects the order
    for z in range(Dz):
        for y in range(Dy):
            for x in range(Dx):
                voxel = volume[z, y, x]
                if voxel.alpha > 0:
                    voxel_pos = (x, y, z)
                    project_and_composite(voxel_pos, voxel.color, voxel.alpha)
                    
                    # Early termination check could be added here per pixel
                    # if alpha_acc[v, u] >= threshold: skip further voxels for this pixel
```

## Subtopics

### [Back-to-Front Rendering](12692-back-to-front-rendering.md)
A viewing algorithm that traverses and renders scene objects in order of decreasing distance from the observer to correctly handle transparency and occlusion. It is the inverse of FTB, where pixels may change values multiple times during generation.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of voxels, assuming constant-time projection and compositing. However, perspective projections with splatting may increase complexity due to footprint calculations.
- **Memory Complexity:** $O(W \times H)$ for the image buffer, where $W$ and $H$ are image dimensions.
- **Performance:** FTB allows for early termination, which can significantly speed up rendering for opaque regions [S2, unofficial]. It also allows for partial image results to be displayed during traversal if the outermost loop is chosen correctly, improving interactivity [S2, unofficial].
- **Strengths:** Correct handling of transparency; early ray termination; partial image results; easy implementation of clipping planes [S2, unofficial].
- **Limitations:** Discrete pixel selection causes holes in perspective projections; loop order limitations for orthographic vs. perspective; requires careful handling of viewing direction for optimal traversal [S2, unofficial][S3, book].
- **When to Use:** Orthographic projections; scenes with significant opacity where early termination is beneficial; interactive applications requiring partial results [S2, unofficial][S3, book].
- **When Not to Use:** Perspective projections requiring high quality without splatting; scenes with complex transparency where strict per-pixel ordering is critical and loop order is insufficient [S3, book].

## Background
The front-to-back algorithm is essentially the same as the back-to-front (BTF) algorithm but traverses voxels in increasing distance order [S1, unofficial]. It was developed as an alternative to BTF to allow for early ray termination and partial image results [S2, unofficial]. The concept is rooted in the painter's algorithm, where objects are drawn from back to front, but adapted for volume rendering by processing voxels from front to back [S1, unofficial]. Splatting, a related technique, was proposed by Westover to address the hole artifacts in perspective projections [S3, book].

## Sources
[S1] Volume Visualization and Volume Rendering Techniques, https://www.cs.unh.edu/~cs880/visualization/Meissner-VolRenderingEGTutorial-rdb.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
[S3] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
