# Image-Order Volumetric Compositing

Also called: Image-Order Compositing, Levoy's Image-Order Compositing, Levoy's Image-Order Volume Rendering

## Overview
Image-Order Volumetric Compositing is a volume rendering framework proposed by Marc Levoy, originally implemented on the PIXEL-PLANES 5 hardware [S1, unofficial]. It solves the problem of real-time volume visualization by precomputing color and opacity arrays from the raw scalar data, allowing the rendering loop to operate directly in image space rather than object space [S2, unofficial]. It sets itself apart from object-order methods by decoupling the shading and classification steps from the ray-casting loop, enabling efficient use of hardware acceleration for compositing [S1, unofficial].

## How it works
The method consists of a preprocessing phase to generate compositing arrays and a rendering phase to composite these arrays into an image.

### Inputs and Outputs
**Inputs:**
*   $S$: A 3D array of scalar data samples.
*   Camera parameters (view direction, projection).
*   Iso-surface value $v$ and target opacity $v_{ought}$ (for classification).
*   Gradient magnitude threshold or transition width parameter (for smoothing).

**Outputs:**
*   $S_c$: A 3D array of color values (RGB) at each grid location.
*   $S_\alpha$: A 3D array of opacity values at each grid location.
*   Final 2D image.

### Data Structures
1.  **$S$**: Input scalar field. Layout: $S[x][y][z]$ where $x, y, z$ are grid indices.
2.  **$S_c$**: Precomputed color field. Layout: $S_c[x][y][z]$ storing RGB tuples.
3.  **$S_\alpha$**: Precomputed opacity field. Layout: $S_\alpha[x][y][z]$ storing scalar opacity.
4.  **Interpolation Functions**:
    *   $f(x,y,z)$: Trilinear interpolation function for sample values.
    *   $f_c(x,y,z)$: Trilinear interpolation function for color values.
    *   $f_\alpha(x,y,z)$: Trilinear interpolation function for opacity values.

### Steps

#### 1. Preprocessing: Generate $S_c$ and $S_\alpha$
This step transforms the raw scalar data $S$ into compositing-ready arrays.

1.  **Compute Gradients**: For each sample in $S$, compute the gradient vector $\nabla f$. The components are obtained by partially differentiating the interpolation function $f$ with respect to $x, y,$ and $z$ [S2, unofficial].
    *   *Note*: To avoid aliasing artifacts from discontinuous normal vectors, the interpolation function must be first derivative continuous. A central differencing method is recommended for smoother gradient vectors [S2, unofficial].
2.  **Generate $S_c$ (Color)**: Perform a shading operation at each data sample.
    *   Use the unit gradient vector $\hat{n} = \frac{\nabla f}{\|\nabla f\|}$ as the surface normal.
    *   Apply a lighting model (e.g., Phong illumination) using $\hat{n}$ to compute the color.
    *   Store the result in $S_c[x][y][z]$ [S2, unofficial].
3.  **Generate $S_\alpha$ (Opacity)**: Perform surface classification on $S$ to determine opacity.
    *   **Basic Classification**: For an iso-surface at constant value $v$ with desired opacity $v_{ought}$:
        $$ S_\alpha(x,y,z) = \begin{cases} v_{ought} & \text{if } f(x,y,z) = v \\ 0 & \text{otherwise} \end{cases} $$
    *   **Smoothed Classification**: To reduce aliasing, assign $S_\alpha(x,y,z)$ close to $v_{ought}$ if $f(x,y,z)$ is close to $v$.
    *   **Transition Width**: To maintain a constant thickness of the transition region, the opacity should fall off at a rate inversely proportional to the magnitude of the local gradient vector $\|\nabla f\|$ [S2, unofficial].
    *   **Multiple Surfaces**: For multiple iso-surfaces, apply separate classification mappings and combine the resulting opacities [S2, unofficial].

#### 2. Rendering: Image-Order Compositing
Cast rays from the image plane through the precomputed arrays.

1.  **Ray Casting**: For each pixel, cast a ray through the volume.
2.  **Sampling**: Sample the arrays $S_c$ and $S_\alpha$ at evenly spaced locations along the ray.
    *   Use trilinear interpolation functions $f_c$ and $f_\alpha$ to determine the color $C_{sample}$ and opacity $\alpha_{sample}$ at each location [S2, unofficial].
3.  **Compositing**: Composite the samples in back-to-front order.
    *   Initialize accumulated color $C_{acc} = 0$ and accumulated opacity $\alpha_{acc} = 0$.
    *   Add a fully opaque background (e.g., $C_{bg} = (1,1,1)$ or scene background) to ensure the ray has a final color if it exits the volume transparently.
    *   For each sample $i$ from back to front:
        $$ C_{acc} = C_{acc} + C_{sample,i} \cdot \alpha_{sample,i} \cdot (1 - \alpha_{acc}) $$
        $$ \alpha_{acc} = \alpha_{acc} + \alpha_{sample,i} \cdot (1 - \alpha_{acc}) $$
    *   Place $C_{acc}$ in the pixel [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_volume(S, gradient_method='central_difference'):
    """
    Preprocesses scalar volume S into color array Sc and opacity array S_alpha.
    """
    # Initialize arrays
    Sc = zeros_like(S, dtype=float) # RGB values
    S_alpha = zeros_like(S, dtype=float)
    
    # 1. Compute Gradients and Colors
    for x, y, z in grid_indices(S):
        # Compute gradient vector components by differentiating interpolation function f
        grad_x, grad_y, grad_z = gradient_method(S, x, y, z)
        grad_norm = sqrt(grad_x**2 + grad_y**2 + grad_z**2)
        
        if grad_norm > 0:
            # Unit normal vector
            nx, ny, nz = grad_x/grad_norm, grad_y/grad_norm, grad_z/grad_norm
            
            # Shading operation (e.g., Phong illumination)
            # Note: Specific lighting parameters are not detailed in sources [G]
            color = phong_illumination(nx, ny, nz)
            Sc[x, y, z] = color
        else:
            Sc[x, y, z] = 0

    # 2. Classify Opacity
    # Parameters: iso_value v, target_opacity v_ought, transition_width_param
    v = 0.5 # Example iso-surface value
    v_ought = 1.0
    
    for x, y, z in grid_indices(S):
        val = S[x, y, z]
        grad_norm = sqrt(gradient_magnitude(S, x, y, z))
        
        # Basic classification
        if abs(val - v) < epsilon:
            S_alpha[x, y, z] = v_ought
        else:
            # Smoothed classification: opacity falls off inversely proportional to gradient magnitude
            # This approximates constant transition region thickness [S2, unofficial]
            # Note: Exact formula for "fall off rate" is not explicitly given in sources [G]
            # Implementation typically uses a sigmoid or linear ramp based on distance from v
            # scaled by 1/grad_norm
            dist = abs(val - v)
            # Placeholder for specific smoothing function dependent on grad_norm
            S_alpha[x, y, z] = v_ought * smooth_step(dist, grad_norm)
            
    return Sc, S_alpha

def image_order_compositing(Sc, S_alpha, ray_origin, ray_direction, image_width, image_height):
    """
    Renders the volume by casting rays and compositing precomputed arrays.
    """
    image = zeros((image_height, image_width, 3), dtype=float)
    
    # Fully opaque background
    background_color = (1.0, 1.0, 1.0) 
    
    for y in range(image_height):
        for x in range(image_width):
            # Cast ray from pixel
            ray = Ray(ray_origin, ray_direction)
            
            # Sample along ray at evenly spaced locations
            samples = []
            for t in ray_march_steps(ray, volume_bounds):
                # Trilinear interpolation for color and opacity
                c_sample = trilinear_interpolate(Sc, ray.point_at(t))
                a_sample = trilinear_interpolate(S_alpha, ray.point_at(t))
                samples.append((c_sample, a_sample))
            
            # Back-to-front compositing
            acc_color = background_color
            acc_alpha = 0.0
            
            # Iterate from back to front
            for c_sample, a_sample in reversed(samples):
                # Standard front-to-back formula applied in reverse order logic
                # C_acc = C_acc + C_sample * alpha_sample * (1 - alpha_acc)
                # alpha_acc = alpha_acc + alpha_sample * (1 - alpha_acc)
                
                # Note: The source specifies back-to-front order for compositing
                # which implies iterating from the furthest sample to the nearest.
                
                weight = a_sample * (1.0 - acc_alpha)
                acc_color = acc_color + c_sample * weight
                acc_alpha = acc_alpha + a_sample * (1.0 - acc_alpha)
                
                # Early termination if fully opaque
                if acc_alpha >= 1.0:
                    break
            
            image[y, x] = acc_color
            
    return image
```

### Parameters
*   **$v$**: Iso-surface value. Typical values depend on the dataset (e.g., 0.5 for normalized data).
*   **$v_{ought}$**: Target opacity for the iso-surface. Typically 1.0 or close to it.
*   **Interpolation**: Trilinear interpolation is specified for $f_c$ and $f_\alpha$ [S2, unofficial].

### Pitfalls
*   **Aliasing**: If the interpolation function is not first derivative continuous, aliasing artifacts occur due to discontinuous normal vectors during shading [S2, unofficial].
*   **Transition Region**: Failing to adjust opacity fall-off based on gradient magnitude can result in variable thickness of the rendered surface, leading to visual inconsistencies [S2, unofficial].

## Operating characteristics
*   **Performance**: The system achieved real-time or near real-time performance on the PIXEL-PLANES 5 hardware [S1, unofficial].
*   **Complexity**: The preprocessing step is $O(N)$ where $N$ is the number of voxels. The rendering step is $O(P \cdot R)$ where $P$ is the number of pixels and $R$ is the number of samples per ray.
*   **Strengths**: Decouples shading from rendering, allowing for efficient hardware compositing. Enables real-time performance on specialized hardware.
*   **Limitations**: Requires significant memory for precomputed $S_c$ and $S_\alpha$ arrays. Less flexible for dynamic transfer functions compared to object-order methods.

## Background
*   **Origin**: Proposed by Marc Levoy [S1, unofficial].
*   **Implementation**: Implemented on the PIXEL-PLANES 5 hardware, which consisted of 32 independently programmable 20-MFLOPS graphics processors, 16 independently programmable renderers with 1/4 million pixel processors, a 512x512 pixel color frame buffer, and a 640 Mb/sec ring network [S1, unofficial].
*   **History**: The method is described in Levoy's work on image-order volumetric compositing [S2, unofficial]. It was one of the early frameworks to achieve real-time volume rendering by leveraging parallel hardware and precomputed data structures.

## Sources
[S1] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S2] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
