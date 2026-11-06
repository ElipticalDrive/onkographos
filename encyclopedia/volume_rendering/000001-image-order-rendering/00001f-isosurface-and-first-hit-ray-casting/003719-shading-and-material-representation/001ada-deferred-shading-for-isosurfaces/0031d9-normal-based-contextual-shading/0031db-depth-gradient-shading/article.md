# Depth-Gradient Shading

Also called: Z-buffer gradient shading

## Overview
Depth-gradient shading is an image-based shading technique that approximates the surface gradient by calculating differences between the depths of a given pixel and its neighbors in the z-buffer [S1, unofficial]. It is used to provide shading cues for isosurfaces or front faces in volume rendering without performing expensive object-space gradient calculations for every voxel [S1, unofficial]. It differs from its parent, Normal-based Contextual Shading, by being purely image-space and context-insensitive, which makes it faster but prone to sharp contour artifacts at object boundaries [S1, unofficial].

## How it works
**Inputs:**
- A depth buffer (z-buffer) containing the depth values of the closest surface intersection for each pixel.
- The pixel spacing in world space, assumed isotropic in X and Y directions ($\Delta x = \Delta y = \text{pixelSpacing}$) [S2].

**Outputs:**
- An estimated surface gradient vector $\nabla_{surf}$ for each pixel.

**Data Structures:**
- Depth buffer: A 2D array of floating-point depth values $z_{i,j}$ for pixel coordinates $(i,j)$ [S1, unofficial].

**Steps:**
1. For each pixel $(i,j)$ with depth $z_{i,j}$, retrieve the depth values of its four immediate neighbors: $(i-1,j)$, $(i+1,j)$, $(i,j-1)$, and $(i,j+1)$ [S1, unofficial].
2. Calculate the absolute depth difference between the current pixel and each neighbor:
   $$ d_x = |z_{i,j} - z_{i-1,j}| $$
   $$ d_x' = |z_{i,j} - z_{i+1,j}| $$
   $$ d_y = |z_{i,j} - z_{i,j-1}| $$
   $$ d_y' = |z_{i,j} - z_{i,j+1}| $$
3. Select the minimum depth difference to estimate the local slope magnitude, avoiding large jumps that may indicate object boundaries or noise:
   $$ d_{min} = \min(d_x, d_x', d_y, d_y') $$
   [S1, unofficial] notes that this approach produces sharp contours where neighboring pixels show different objects or have drastic depth value jumps [S1, unofficial].
4. Compute the surface gradient magnitude. Assuming the direction of the steepest descent corresponds to the neighbor with the smallest depth difference, the gradient magnitude is approximated by:
   $$ |\nabla_{surf}| \approx \frac{d_{min}}{\text{pixelSpacing}} $$
   [S2]
5. The direction of the gradient is determined by the neighbor with the minimal difference. If the minimum is $d_x$, the gradient is primarily in the X direction; if $d_y$, in the Y direction. The vector can be constructed as:
   $$ \nabla_{surf} = \left( \frac{z_{i,j} - z_{neighbor\_x}}{\text{pixelSpacing}}, \frac{z_{i,j} - z_{neighbor\_y}}{\text{pixelSpacing}}, 1 \right) $$
   Normalized appropriately for shading calculations. [S1, unofficial] states the gradient is approximated from the z-buffer calculating differences between depths.

**Parameters:**
- `pixelSpacing`: Distance between neighboring pixels in world space. Assumed isotropic [S2].

**Edge cases and pitfalls:**
- **Sharp contours:** The method produces sharp contour artifacts when neighboring pixels display different objects or when there is a drastic jump in depth values [S1, unofficial].
- **Object boundaries:** Standard depth-gradient shading does not account for object or slope discontinuities, leading to incorrect normals at boundaries [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def depth_gradient_shading(depth_buffer, pixel_spacing):
    """
    Estimates surface gradient for each pixel using depth-buffer differences.
    
    Args:
        depth_buffer: 2D array of float, shape (H, W), containing z-values.
        pixel_spacing: float, world distance between adjacent pixels.
        
    Returns:
        gradients: 2D array of vectors (dx, dy, dz) for each pixel.
    """
    H, W = depth_buffer.shape
    gradients = np.zeros((H, W, 3))
    
    # Pad depth buffer to handle boundaries easily
    padded_depth = np.pad(depth_buffer, pad_width=1, mode='edge')
    
    for i in range(1, H + 1):
        for j in range(1, W + 1):
            z_center = padded_depth[i, j]
            
            # Neighbors: left, right, top, bottom
            z_left = padded_depth[i, j - 1]
            z_right = padded_depth[i, j + 1]
            z_top = padded_depth[i - 1, j]
            z_bottom = padded_depth[i + 1, j]
            
            # Calculate absolute differences
            diff_left = abs(z_center - z_left)
            diff_right = abs(z_center - z_right)
            diff_top = abs(z_center - z_top)
            diff_bottom = abs(z_center - z_bottom)
            
            # Find minimum difference
            diffs = [diff_left, diff_right, diff_top, diff_bottom]
            min_diff = min(diffs)
            
            # Determine direction based on minimum difference
            # Gradient vector approximation: (dx, dy, 1) scaled by depth change
            # We approximate the normal direction. 
            # If min is left/right, dx is non-zero. If top/bottom, dy is non-zero.
            
            dx = 0.0
            dy = 0.0
            
            if min_diff == diff_left:
                dx = (z_center - z_left) / pixel_spacing
            elif min_diff == diff_right:
                dx = (z_center - z_right) / pixel_spacing
            elif min_diff == diff_top:
                dy = (z_center - z_top) / pixel_spacing
            elif min_diff == diff_bottom:
                dy = (z_center - z_bottom) / pixel_spacing
            
            # The z-component of the gradient in image space is typically 1 for normalized device coordinates
            # or derived from the depth derivative. For shading, we often normalize the vector.
            # A common approximation for the surface normal from depth is:
            # n = (-dz/dx, -dz/dy, 1) normalized. Here we have discrete differences.
            
            # Using the selected difference to form the gradient vector components
            # The magnitude is min_diff / pixel_spacing
            # The direction is given by the axis of the minimum difference.
            
            # Construct gradient vector (approximate surface normal direction)
            # Note: The exact vector construction depends on the coordinate system.
            # Assuming standard view direction along Z.
            grad_x = dx
            grad_y = dy
            grad_z = 1.0 # Assumption for image-space gradient
            
            gradients[i-1, j-1] = [grad_x, grad_y, grad_z]
            
    return gradients
```

## Subtopics

### Depth-Gradient Shading with Ray-Layer Buffer
A technique to estimate surface gradients for shading by analyzing depth coherency in a ray-layer buffer, solving the problem of noisy shading in regions with low volume gradient magnitudes [S2]. It is chosen when mask surfaces cut through volume regions with low gradient magnitudes, which would otherwise result in noisy or flat shading [S2]. It differs from standard depth-gradient shading by using a ray-layer buffer and material IDs to ensure depth differences are computed only between pixels belonging to the same surface layer [S2].

**Steps:**
1. Assume that depth values produced by the same surface have strong coherency, such that an entry in the ray-layer buffer has at least one direct neighbor produced by the same surface and encoded into the same layer [S2].
2. For the current layer being ray casted, retrieve the depth values from the four neighboring pixels in the ray-layer buffer for the same layer [S2].
3. Compute the difference between the current depth and each neighbor's depth, selecting only the difference with the smaller absolute value to avoid using depth values from different layer surfaces [S2].
4. Calculate the surface gradient using the selected depth difference and `pixelSpacing` (distance between neighboring pixels in world space, assuming isotropic X and Y directions) [S2].
5. Blend the surface gradient with the volume gradient using the formula:
   $$ \text{gradient} = \text{blend} \cdot \text{volumeGradient} + (1 - \text{blend}) \cdot \text{surfaceGradient} $$
   where
   $$ \text{blend} = \min\left(1, \frac{\text{gradientMagnitude}}{\text{threshold}}\right) $$
   [S2]

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel for standard depth-gradient shading, as it involves a constant number of neighbor lookups and arithmetic operations. The ray-layer buffer variant also operates in $O(1)$ per pixel but requires access to the ray-layer buffer structure.
- **Memory Complexity:** $O(N)$ for the depth buffer, where $N$ is the number of pixels. The ray-layer buffer variant requires additional memory for the multi-layer buffer.
- **Strengths:** Simple to implement; computationally inexpensive; provides shading cues without precomputing volume gradients [S1, unofficial].
- **Limitations:** Prone to sharp contour artifacts at object boundaries and depth discontinuities [S1, unofficial]. Does not account for object or slope discontinuities [S1, unofficial]. Noisy shading in regions with low volume gradient magnitudes if not handled [S2].
- **When to use:** When performance is critical and the scene has few overlapping objects or smooth depth transitions.
- **When not to use:** When high-quality shading is required in complex scenes with multiple overlapping objects or sharp depth changes, or when volume gradients are low and noisy [S1, unofficial][S2].

## Background
Depth-gradient shading was introduced as a simple image-based shading technique for volume rendering [S1, unofficial]. It is cited in works such as [6] and [55] referenced in [S1, unofficial]. The technique is often contrasted with context-sensitive normal estimation [66] which addresses its artifacts [S1, unofficial]. The variant using a ray-layer buffer was developed to improve shading quality in regions with low volume gradient magnitudes, particularly in multi-layer volume ray casting on GPU [S2].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S2] Multi-layer Volume Ray Casting on GPU, https://doi.org/10.2312/vg/vg10/005-012, fetched 2026-10-26
