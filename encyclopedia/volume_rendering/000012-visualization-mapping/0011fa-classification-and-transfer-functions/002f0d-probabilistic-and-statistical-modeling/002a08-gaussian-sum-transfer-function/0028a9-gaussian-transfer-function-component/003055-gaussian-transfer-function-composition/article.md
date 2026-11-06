# Gaussian Transfer Function Composition

Also called: TF Blending

## Overview
Gaussian Transfer Function Composition is an interactive method for constructing multi-dimensional transfer functions by blending 2D Gaussian opacity functions with associated color definitions in a reduced-dimensional map space [S1]. It solves the problem of manually defining complex, multi-dimensional classification rules by allowing users to iteratively add Gaussian "blobs" to a transfer function canvas [S1]. What sets it apart from its siblings is its reliance on a pre-computed dimensional reduction map (such as a Kohonen map or spherical map) where the transfer function is defined as an image, rather than directly in scalar or gradient space [S1]. It builds on [Gaussian Transfer Function Component](4330-gaussian-transfer-function-component.md) by providing the mechanism to combine multiple components into a final RGBA texture.

## How it works
The method operates on a 2D map representation of the data (e.g., a Kohonen Self-Organizing Map or a spherical map) where each pixel corresponds to a cluster of voxel signatures [S1]. The transfer function is represented as an RGBA image over this map space [S1].

**Inputs and Outputs**
*   **Input:** A 2D map of data signatures (e.g., scalar, gradient magnitude, directional second derivative mapped to RGB channels) [S1].
*   **Input:** User interaction defining Gaussian parameters (center, radius $\sigma$, height $k$, and color) [S1].
*   **Output:** A final composed RGBA transfer function image [S1].

**Data Structures**
The transfer function is stored as an image with fields for Red, Green, Blue, and Alpha channels for each pixel in the map space [S1].
*   `alpha_current`: 2D array of opacity values, initialized to 0.
*   `color_current`: 2D array of RGB color vectors, initialized to (0, 0, 0).
*   `map_coordinates`: 2D array of coordinates corresponding to the map space [S1].

**Steps**
1.  **Initialize:** Set `alpha_current` to zero and `color_current` to black (RGB 0,0,0) for all pixels in the map space [S1].
2.  **Define Gaussian:** The user specifies a new Gaussian component by clicking or dragging on the map to set the center (peak) and radius ($\sigma$) [S1]. The user also specifies a color and a height factor $k$ [S1].
3.  **Calculate Gaussian Opacity:** For each pixel in the map space, calculate the distance $d$ from the pixel to the Gaussian center [S1]. Compute the Gaussian opacity $\alpha_{Gaussian}$ using Equation 7 [S1]:
    $$ \alpha_{Gaussian} = k \times \exp\left(-\frac{d^2}{2\sigma^2}\right) $$
    where $k$ is a constant between 0 and 1, linearly mapped to the circle color (blue=0, red=1) [S1].
4.  **Calculate Gaussian Color:** Determine the color $C_{Gaussian}$ for the new Gaussian component. Three types are supported [S1]:
    *   A constant color chosen from a color picker [S1].
    *   Map coordinates directly mapped to color channels [S1].
    *   Interpolated weight vector elements of map cells mapped to color channels [S1].
5.  **Blend Opacity:** Update the current opacity $\alpha_{new}$ by blending the existing opacity with the new Gaussian opacity using Equation 5 [S1]:
    $$ \alpha_{new} = 1 - (1 - \alpha_{current}) \times (1 - \alpha_{Gaussian}) $$
6.  **Blend Color:** Update the current color $C_{new}$ by blending the existing color with the new Gaussian color using Equation 6 [S1]:
    $$ C_{new} = \frac{\alpha_{current} \times C_{current} + \alpha_{Gaussian} \times C_{Gaussian}}{\alpha_{current} + \alpha_{Gaussian}} $$
    If $\alpha_{current} + \alpha_{Gaussian} = 0$, $C_{new}$ remains unchanged [G].
7.  **Update State:** Set `alpha_current` to $\alpha_{new}$ and `color_current` to $C_{new}$ for all pixels [S1].
8.  **Iterate:** Repeat steps 2-7 for additional Gaussian components until the desired transfer function is reached [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compose_gaussian_tf(map_width, map_height, map_data, gaussians):
    """
    Composes a transfer function by blending multiple Gaussian components.
    
    Args:
        map_width (int): Width of the map image.
        map_height (int): Height of the map image.
        map_data (array): 2D array of map coordinates or signatures.
        gaussians (list): List of Gaussian component definitions.
                          Each element is a dict with keys:
                          - 'center': (x, y) tuple
                          - 'sigma': float
                          - 'k': float (height factor)
                          - 'color_type': str ('constant', 'coords', 'weights')
                          - 'color_value': color vector or reference
    
    Returns:
        tuple: (alpha_map, color_map) representing the final TF.
    """
    # Initialize current TF with zero opacity and black color
    alpha_current = np.zeros((map_height, map_width))
    color_current = np.zeros((map_height, map_width, 3))
    
    for g in gaussians:
        center = g['center']
        sigma = g['sigma']
        k = g['k']
        color_type = g['color_type']
        color_value = g['color_value']
        
        # Calculate distance from each pixel to the Gaussian center
        y, x = np.ogrid[:map_height, :map_width]
        d_squared = (x - center[0])**2 + (y - center[1])**2
        
        # Calculate Gaussian opacity for all pixels
        # alpha_gaussian = k * exp(-d^2 / (2 * sigma^2))
        alpha_gaussian = k * np.exp(-d_squared / (2 * sigma**2))
        
        # Determine Gaussian color for all pixels
        if color_type == 'constant':
            color_gaussian = np.full((map_height, map_width, 3), color_value)
        elif color_type == 'coords':
            # Map coordinates directly mapped to color channels
            # Assuming map_data contains normalized coordinates [0,1]
            color_gaussian = map_data # Shape: (map_height, map_width, 3)
        elif color_type == 'weights':
            # Interpolated weight vectors of map cells
            # This requires access to the SOM weight vectors and interpolation logic
            # Not fully specified in sources, assumed to be a function of map_data
            color_gaussian = interpolate_weights(map_data, color_value)
        
        # Blend Opacity: alpha_new = 1 - (1 - alpha_current) * (1 - alpha_gaussian)
        alpha_new = 1.0 - (1.0 - alpha_current) * (1.0 - alpha_gaussian)
        
        # Blend Color: C_new = (alpha_current * C_current + alpha_gaussian * C_gaussian) / (alpha_current + alpha_gaussian)
        # Handle division by zero where both alphas are zero
        denominator = alpha_current + alpha_gaussian
        # Avoid division by zero
        safe_denominator = np.where(denominator > 0, denominator, 1.0)
        
        color_new = (alpha_current[:, :, np.newaxis] * color_current + 
                     alpha_gaussian[:, :, np.newaxis] * color_gaussian) / safe_denominator[:, :, np.newaxis]
        
        # Update current state
        alpha_current = alpha_new
        color_current = color_new
        
    return alpha_current, color_current
```

**Parameters**
*   $\sigma$ (sigma): Standard deviation of the Gaussian, controlling the radius of influence [S1].
*   $k$: Height factor, constant between 0 and 1, controlling the maximum opacity [S1].
*   $s$: Constant for visualization blending, empirically set to 200 [S1].

**Edge Cases and Pitfalls**
*   **Division by Zero:** In the color blending formula, if both $\alpha_{current}$ and $\alpha_{Gaussian}$ are zero, the denominator is zero. The color should remain unchanged in this case [G].
*   **Precision:** Floating-point precision may affect the blending of many small Gaussian components [G].
*   **Map Boundaries:** The distance calculation $d$ must handle the topology of the map (e.g., spherical wrap-around if using a spherical map) [S1].

## Subtopics

### TF Visualization Blending
A visualization technique to display Transfer Functions by blending them with a checkerboard pattern, using a logarithmic opacity scaling to enhance visibility of low-opacity regions [S1]. It is used to inspect the transparency of the TF itself, not for rendering the volume [S1]. Choose this when debugging or inspecting the opacity distribution of a TF [S1]. It uses the formula $C_{out} = \alpha_{blending} \times C_{TF} + (1 - \alpha_{blending}) \times C_{checker}$ where $\alpha_{blending} = \frac{\log(1 + s \times \alpha_{TF})}{\log(1 + s)}$ and $s$ is a constant greater than one (empirically 200) [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels in the map and the number of Gaussian components [S1]. Each Gaussian requires a distance calculation and blending operation for all pixels [S1].
*   **Memory Complexity:** Proportional to the size of the map image (width $\times$ height $\times$ 4 bytes for RGBA) [S1].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Interactive previewing of the effect of composition while the user explores the map [S1]. Allows for intuitive construction of complex TFs using simple geometric primitives [S1].
*   **Limitations:** Dependent on the quality of the dimensional reduction map [S1]. The blending is additive in opacity, which may lead to saturation if many high-opacity Gaussians are used [G].
*   **When to use:** When an interactive, visual approach to TF design is preferred over parametric editing [S1].
*   **When not to use:** When high precision or analytical control over the TF is required [G].

## Background
The method was introduced in the paper "Design of Multi-dimensional Transfer Functions Using Dimensional Reduction" by [S1] in 2007. It is used in conjunction with dimensional reduction techniques like Kohonen maps or spherical maps to visualize and edit transfer functions in a reduced space [S1]. The approach allows users to define TFs by manipulating Gaussian blobs on these maps, providing an intuitive interface for exploring the relationship between data signatures and their visual properties [S1].

## Sources
[S1] Design of Multi-dimensional Transfer Functions Using Dimensional Reduction, https://doi.org/10.2312/vissym/eurovis07/131-138, fetched 2026-10-26
