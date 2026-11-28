# Soft Point Splatting

Also called: Soft rasterization (for point clouds)

## Overview
Soft point splatting is a point cloud rendering technique that reconstructs a continuous signal from irregularly discrete samples using local reconstruction kernels, then resamples the signal in image space to produce a 2D feature map [S1, unofficial]. It differs from conventional point renderers (e.g., OpenGL/DirectX) by providing gradients with respect to point positions via the reconstruction kernel, whereas conventional methods project points to single pixels and do not provide such gradients [S1, unofficial].

## How it works
**Inputs:** A set of discrete samples $P = \{p_i\}$ where each point $p_i$ has an associated value $f_i$ (e.g., color, normal, or learned neural descriptor) [S1, unofficial].
**Outputs:** A 2D feature map (or final RGB image) [S1, unofficial].

**Data Structures:**
*   Point Cloud: List of points $p_i$ with attributes $f_i$.
*   Reconstruction Kernel: A function $r(\cdot)$, typically spatially-invariant and isotropic for performance [S1, unofficial].
*   Feature Map: A 2D grid storing accumulated feature values.

**Steps:**
1.  **Construct Continuous Signal:** Reconstruct the continuous signal $f$ from the discrete samples using local reconstruction kernels $r(\cdot)$ [S1, unofficial]:
    $$ f = \sum_i f_i r(p_i) $$
    where $f$ is the continuous signal, $f_i$ are the discrete sample values, and $r(p_i)$ is the local reconstruction kernel centered at point $p_i$ [S1, unofficial].
2.  **Blend and Project:** Blend the discrete samples with local deterministic blurring kernels to project and blend individual features into a 2D feature map [S1, unofficial]. In neural rendering contexts, this step effectively projects and blends learnable features into the 2D feature map [S1, unofficial].
3.  **Generate Final Image:**
    *   If features have predefined semantic meaning (e.g., colors, normals), use a fixed shading function or BRDF to generate the final image [S1, unofficial].
    *   If features are learned neural descriptors, deploy a 2D neural network to transform the 2D feature map to an RGB image [S1, unofficial].

**Parameters:**
*   Kernel type: Spatially-invariant and isotropic kernels are commonly used for performance reasons [S1, unofficial].
*   Kernel radius: In methods like Pulsar, radii can be optimized to compensate for artifacts [S1, unofficial].

**Edge Cases and Pitfalls:**
*   Using simplified spatially-invariant and isotropic kernels results in rendering artifacts such as holes, blurred edges, and aliasing [S1, unofficial].
*   These artifacts can be compensated in the neural shading step or through optimization of radii [S1, unofficial].
*   The gradient provided by the reconstruction kernel is confined spatially within the support of the local reconstruction [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def soft_point_splatting(points, kernel_func, feature_map_shape, neural_net=None):
    """
    Render a point cloud using soft point splatting.
    
    Args:
        points: List of tuples (position, feature_value).
        kernel_func: Function r(p) that returns the kernel value at position p.
        feature_map_shape: Tuple (height, width) of the output feature map.
        neural_net: Optional 2D neural network for transforming feature map to RGB.
        
    Returns:
        feature_map: 2D array of accumulated features.
        image: Optional RGB image if neural_net is provided.
    """
    # Initialize feature map
    feature_map = np.zeros(feature_map_shape)
    
    # Step 1 & 2: Construct continuous signal and blend into 2D feature map
    for position, feature_value in points:
        # Project point to screen space (assuming pre-computed screen coordinates)
        # Note: The source does not specify the exact projection math, 
        # but implies blending based on kernel support.
        screen_pos = project_to_screen(position)
        
        # Determine kernel support and blend
        # The kernel r(p) defines the influence of the point.
        # For isotropic kernels, this is typically a function of distance.
        # We accumulate feature_value * kernel_value into the feature map.
        # Implementation detail: Iterate over pixels within kernel support.
        support_radius = get_kernel_support_radius(kernel_func)
        for dy in range(-support_radius, support_radius + 1):
            for dx in range(-support_radius, support_radius + 1):
                pixel_x = int(screen_pos.x) + dx
                pixel_y = int(screen_pos.y) + dy
                
                if 0 <= pixel_x < feature_map_shape[1] and 0 <= pixel_y < feature_map_shape[0]:
                    # Calculate distance from pixel center to point center
                    dist = np.sqrt(dx**2 + dy**2)
                    kernel_value = kernel_func(dist)
                    
                    # Accumulate feature
                    feature_map[pixel_y, pixel_x] += feature_value * kernel_value
                    
    # Step 3: Generate final image
    if neural_net is not None:
        image = neural_net(feature_map)
        return feature_map, image
    else:
        # Use fixed shading function or BRDF if features have semantic meaning
        # This step is not detailed in the source, so we return the feature map
        return feature_map
```

## Subtopics

### SinSyn
A neural point rendering method that adopts soft point splatting [S1, unofficial].

### Pulsar
A neural point rendering method that uses spatially-invariant and isotropic kernels and optimizes radii to compensate for artifacts [S1, unofficial].

### KPLD21
A method that uses strategies for camera selection and probabilistic depth testing within the soft point splatting framework, tackling IBR, stylization, and harmonization [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Provides point position gradients via the reconstruction kernel, enabling differentiation with respect to point positions [S1, unofficial].
*   **Limitations:** Simplified kernels can result in artifacts such as holes, blurred edges, and aliasing [S1, unofficial]. The gradient is confined spatially within the support of the local reconstruction [S1, unofficial].
*   **When to use:** When gradients with respect to point positions are required, such as in neural rendering optimization [S1, unofficial].
*   **When not to use:** When high-fidelity, artifact-free rendering is required without relying on neural compensation or radius optimization [S1, unofficial].

## Background
Soft point splatting is based on the theory of classic signal processing and is similar to the soft rasterizer in mesh rendering [S1, unofficial]. It is used in neural rendering methods such as SinSyn and Pulsar [S1, unofficial]. The approach addresses the limitation of conventional point renderers by providing gradients with respect to point positions, which are confined to the kernel support [S1, unofficial].

## Sources
[S1] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
