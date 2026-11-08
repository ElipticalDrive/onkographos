# Vignette Shading

Also called: Cubic Vignette, Vignette, Vignette Effect

## Overview
Vignette Shading is a post-processing effect applied to display images before tonemapping to darken the corners, simulating the reduction in light intensity at the edges of an image due to projection onto a flat plane [S1, S2, unofficial]. It is distinct from sibling methods like [Lit Sphere Shading](785-lit-sphere-shading.md) or [Orientation-Based Color Modification](798-orientation-based-color-modification.md) because it operates on the final 2D image plane based on pixel orientation relative to the camera frame, rather than modifying volume density or surface normals in 3D space [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: The rendered display image (before tonemapping), ray directions for each pixel, and the pixel frame orientation matrices.
- Output: The image with darkened corners.

**Data Structures:**
- `ray_directions`: Tensor of shape $(H, W, 3)$ containing the normalized direction vector $\mathbf{r}$ for each pixel.
- `pixel_frames`: Tensor of shape $(H, W, 3, 3)$ or $(H, W, 9)$ representing the local coordinate frame for each pixel. The third column (index 2) corresponds to the z-axis of the pixel frame, denoted as $\mathbf{f}_z$ [S2, unofficial].

**Formula:**
The vignette factor $V$ is calculated based on the alignment between the ray direction and the pixel frame's z-axis [S2, unofficial]:

$$
V = (\mathbf{r} \cdot \mathbf{f}_z)^3
$$

Where:
- $\mathbf{r}$ is the ray direction vector.
- $\mathbf{f}_z$ is the unit normal vector of the pixel frame (the third component of `pixel_frames`) [S2, unofficial].
- The exponent 3 creates a "cubic" falloff, ensuring the effect is smooth and differentiable [S2, unofficial].

This is equivalent to the geometric derivation:
$$
\text{Factor} = \frac{f^2}{f^2+r_p^2}(\mathbf{v}_p \cdot \mathbf{n}_p) = (\cos \theta)^3 = (\mathbf{v}_p \cdot \mathbf{n}_p)^3
$$
Where $f$ is focal distance, $r_p$ is radial distance, $\mathbf{v}_p$ is the vector from pixel to focal point, and $\mathbf{n}_p$ is the pixel normal [S2, unofficial]. In the implementation, the dot product $\mathbf{r} \cdot \mathbf{f}_z$ directly computes $\cos \theta$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vignette_shading(image, ray_directions, pixel_frames):
    """
    Apply cubic vignette shading to the image.
    
    Args:
        image: Tensor of shape (H, W, 3) or (H, W, 4), pre-tonemapping.
        ray_directions: Tensor of shape (H, W, 3), normalized ray directions.
        pixel_frames: Tensor of shape (H, W, 3, 3), where pixel_frames[..., 2] is the z-axis normal.
        
    Returns:
        Tensor of shape (H, W, 3) or (H, W, 4) with vignette applied.
    """
    # Extract the z-axis normal from the pixel frame (f_z)
    # pixel_frames shape: (H, W, 3, 3)
    # We want the third column (index 2) for each pixel
    f_z = pixel_frames[..., 2]  # Shape: (H, W, 3)
    
    # Compute dot product between ray direction and f_z
    # ray_directions shape: (H, W, 3)
    dot_product = (ray_directions * f_z).sum(dim=-1, keepdim=True)  # Shape: (H, W, 1)
    
    # Apply cubic falloff
    vignette_factor = dot_product.pow(3)  # Shape: (H, W, 1)
    
    # Apply vignette to the image
    # Assuming image has shape (H, W, C)
    shaded_image = image * vignette_factor
    
    return shaded_image
```

**Parameters:**
- Exponent: 3 (fixed for cubic vignette) [S2, unofficial].
- Focal distance $f$: Implicitly handled by the ray direction normalization and frame alignment [S2, unofficial].

**Edge Cases and Pitfalls:**
- If $\mathbf{r} \cdot \mathbf{f}_z$ is negative (ray pointing away from the frame normal), the cubic power preserves the sign, potentially inverting colors. In practice, ray directions are typically oriented such that this dot product is positive for visible pixels [G].
- Precision issues may arise if ray directions are not normalized. Ensure `ray_directions` are unit vectors [G].

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(H \times W)$, where $H$ and $W$ are image dimensions. The operation is a simple element-wise multiplication after a dot product.
- **Memory Complexity:** $O(H \times W)$ for storing intermediate tensors.
- **Performance:** Extremely fast, as it is a post-processing step applied to the final image [S1, unofficial].
- **Strengths:** Simple to implement, adds a cinematic look, simulates optical effects [S1, S2, unofficial].
- **Limitations:** Does not account for 3D scene geometry, only 2D image plane orientation [S2, unofficial].
- **When to use:** When a post-processing vignette effect is desired for stylistic reasons [S1, unofficial].
- **When not to use:** When physically accurate illumination modeling is required, or when the effect needs to vary with 3D depth [S2, unofficial].

## Background
Vignette Shading is used in real-time rendering pipelines to enhance the visual appeal of images by darkening corners, simulating lens effects [S1, unofficial]. It is implemented in projects like `mypbr` as a post-processing step before tonemapping [S1, unofficial]. The cubic formulation is derived from the geometric relationship between ray direction and pixel frame orientation in ray marching contexts [S2, unofficial].

## Sources
[S1] Milestone 13: MagicaVoxel look (.vox, DDA voxel grid, sun/sky, bloom, style voxel), https://github.com/adriendjn/mypbr/pull/10, fetched 2026-10-25 (unofficial)
[S2] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
