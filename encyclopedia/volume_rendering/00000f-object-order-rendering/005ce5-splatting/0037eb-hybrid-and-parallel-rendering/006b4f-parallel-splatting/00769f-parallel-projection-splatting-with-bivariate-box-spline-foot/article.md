# Parallel Projection Splatting with Bivariate Box Spline Footprint

Also called: Efficient Splatting

## Overview
Parallel Projection Splatting with Bivariate Box Spline Footprint is a volume rendering technique for parallel projections where splats have identical shapes, differing only in color intensity and scale factor [S1]. It projects $n$-dimensional voxels (modeled as constant transparency cubes) onto 2D image space, using the luminosity distribution of a bivariate box spline [S1]. Unlike general splatting methods that may require complex per-voxel footprint calculations or depth sorting, this method leverages the geometric properties of parallel projection to reuse a single standard splat shape, scaled and colored per voxel, without ordering voxels for occlusion [S1].

## How it works
### Inputs and Outputs
**Inputs:**
*   A decomposition of space into elementary volume regions (voxels) [S1].
*   A scalar field sampled at the center of each voxel, assumed constant within the voxel [S1].
*   For each color component (e.g., Red, Green, Blue), a luminosity scalar factor $L$ and a normalization range $(\min, \max)$ [S1].

**Outputs:**
*   A 2D image where each pixel $p$ contains the integrated luminosity from all voxels projected onto it [S1].

### Data Structures
*   **Voxel Grid:** A regular grid where each entry stores the scalar field value at the voxel center [S1].
*   **Standard Splat:** A 2D texture map stored in memory representing the luminosity distribution of a single unit voxel projected under the current view [S1].
*   **Mipmaps:** A hierarchy of the standard splat texture at different resolution levels to support automatic scaling [S1].

### Parameters
*   $L_r$: Luminosity scalar factor controlling the intensity of the red color component [S1].
*   $(\min_r, \max_r)$: The range of scalar field values associated with the red color component, used to normalize the field value $F_r$ [S1].
*   Similar parameters $L_g, (\min_g, \max_g)$ and $L_b, (\min_b, \max_b)$ apply to green and blue components [S1].

### Steps
1.  **Compute Standard Splat:** Calculate the shape of the standard splat or footprint. This is the projection of an $n$-dimensional cube (the voxel) of constant transparency onto the 2D image space. The luminosity distribution of this splat is a bivariate box spline [S1].
    *   *Note:* An exact drawing of the initial splat requires a square of size equal to one pixel, but in practice, a fairly larger square is sufficient because subsequent splat uses shrink the footprint to the necessary size [S1].
2.  **Store Splat:** Store the computed standard splat image in texture map memory [S1].
3.  **Render Voxels:** For each voxel in the input data:
    a. Determine the color intensity and scale factor based on the voxel's scalar field value and the configured ranges [S1].
    b. Draw the voxel by copying the standard splat, scaled by the determined size and color intensity [S1].
    c. Do not order voxels for occlusion; overlapping voxels are drawn without depth sorting [S1].

### Formulas
The color component $C(p)$ (e.g., Red, Green, Blue) for pixel $p$ is calculated as [S1]:

$$C(p) = L_c \int_{p+K} F_c dk$$

Where:
*   $L_c$ is the luminosity of the color component $c$ [S1].
*   $K$ is the projection kernel (the projection ray through $p$ in 3D, or generally the projection of the voxel along the view direction) [S1].
*   $dk$ is the $(n-2)$ dimensional differential element [S1].
*   $F_c$ is the scalar field value normalized in the $(\min_c, \max_c)$ range associated with the color component $c$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_projection_splatting(voxel_grid, view_direction, image_width, image_height):
    """
    Renders a volume using parallel projection splatting with bivariate box spline footprint.
    
    Args:
        voxel_grid: 3D array of scalar field values.
        view_direction: Normalized vector representing the parallel projection direction.
        image_width: Width of the output image in pixels.
        image_height: Height of the output image in pixels.
        
    Returns:
        image: 2D array representing the rendered image.
    """
    
    # Initialize output image
    image = zeros((image_height, image_width, 3))
    
    # Step 1: Compute the standard splat footprint
    # The splat is the projection of a unit voxel onto the image plane.
    # Its luminosity distribution is a bivariate box spline.
    # In practice, a square larger than 1 pixel is used for the initial splat texture.
    standard_splat = compute_bivariate_box_spline_footprint(view_direction)
    
    # Step 2: Store splat in texture memory (conceptually)
    # For software implementation, we keep the spline function or sampled texture.
    splat_texture = standard_splat
    
    # Step 3: Render each voxel
    # Iterate over all voxels in the grid
    for voxel in voxel_grid:
        # Get scalar value at voxel center
        scalar_value = voxel.scalar_value
        
        # Normalize scalar value to [0, 1] based on configured ranges
        # Assuming ranges are provided or derived from data min/max
        normalized_value = normalize(scalar_value, min_val=0.0, max_val=1.0)
        
        # Determine color intensity and scale factor
        # Luminosity factors L_r, L_g, L_b are parameters
        # Scale factor depends on the voxel size and projection
        color_intensity = normalized_value
        scale_factor = 1.0 # Assuming uniform voxel size for simplicity
        
        # Draw the splat
        # Project voxel center to image plane
        projected_center = project_to_image_plane(voxel.center, view_direction, image_width, image_height)
        
        # Draw the scaled and colored splat at the projected center
        draw_splat(
            image=image,
            center=projected_center,
            texture=splat_texture,
            scale=scale_factor,
            color=color_intensity
        )
        
    return image

def compute_bivariate_box_spline_footprint(view_direction):
    """
    Computes the 2D footprint of a voxel projected along the view direction.
    The footprint is a bivariate box spline.
    """
    # This function calculates the exact or approximated luminosity distribution.
    # For implementation, this can be precomputed and stored as a texture.
    # The shape depends on the view direction relative to the voxel axes.
    pass

def project_to_image_plane(point_3d, view_direction, width, height):
    """
    Projects a 3D point onto the 2D image plane using parallel projection.
    """
    # Implementation depends on the specific camera setup.
    # Generally involves transforming the point into camera space and projecting.
    pass

def draw_splat(image, center, texture, scale, color):
    """
    Draws a scaled and colored splat onto the image.
    """
    # Iterate over the bounding box of the splat in image coordinates
    # For each pixel in the bounding box, sample the texture, scale by color, and accumulate
    pass

def normalize(value, min_val, max_val):
    """
    Normalizes a value to [0, 1] based on min and max.
    """
    if max_val == min_val:
        return 0.0
    return (value - min_val) / (max_val - min_val)
```

### Edge Cases and Pitfalls
*   **Initial Splat Size:** An exact drawing of the initial splat requires a square of size equal to one pixel, but in practice, a fairly larger one is sufficient because subsequent splat uses shrink the footprint to the necessary size [S1].
*   **No Occlusion:** The method does not order voxels for occlusion. Overlapping voxels are drawn without depth sorting, which may lead to visual inaccuracies in scenes with significant depth variation [S1].
*   **Dimensionality:** The method is designed for $n$-dimensional spaces, but the projection kernel $K$ and differential element $dk$ depend on $n$. For $n=3$, $dk$ is 1-dimensional [S1].

## Operating characteristics
*   **Time Complexity:** The rendering time is proportional to the number of voxels, as each voxel is processed once. The cost of drawing each splat depends on the size of the footprint and the resolution of the splat texture. Using mipmaps allows for efficient scaling [S1].
*   **Memory Complexity:** Memory usage is dominated by the storage of the voxel grid and the standard splat texture. The hierarchical representation overhead is bounded by a term that decreases exponentially with the dimension $n$, making it very small for $n > 3$ [S1].
*   **Performance:** This approach allows rendering a large number of splats quickly, achieving almost interactive rates for fairly complex datasets, especially when hardware acceleration is used [S1].
*   **Strengths:** Simplicity and efficiency due to the reuse of a single standard splat shape. No need for depth sorting. Hardware acceleration can be leveraged by storing splats in texture memory [S1].
*   **Limitations:** Lack of occlusion handling may reduce visual quality in certain scenes. The method assumes constant transparency within voxels [S1].

## Background
This method was introduced in the context of hypervolume visualization, addressing the challenge of simplicity in rendering high-dimensional data [S1]. It builds on spline theory, specifically the properties of bivariate box splines, to define the splat footprint [S1]. The technique is particularly relevant for parallel projections where the uniformity of splat shapes can be exploited for efficiency [S1].

## Sources
[S1] Hypervolume visualization: a challenge in simplicity, https://doi.org/10.1145/288126.288173, fetched 2026-10-26
