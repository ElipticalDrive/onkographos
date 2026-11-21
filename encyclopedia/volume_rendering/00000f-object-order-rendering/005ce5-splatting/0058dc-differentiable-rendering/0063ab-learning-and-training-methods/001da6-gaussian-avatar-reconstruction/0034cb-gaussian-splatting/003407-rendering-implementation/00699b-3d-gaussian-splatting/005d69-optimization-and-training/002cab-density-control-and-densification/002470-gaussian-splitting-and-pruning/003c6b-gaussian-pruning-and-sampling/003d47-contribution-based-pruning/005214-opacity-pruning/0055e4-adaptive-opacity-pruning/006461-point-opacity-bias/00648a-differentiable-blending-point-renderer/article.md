# Differentiable Blending Point Renderer

Also called: Differentiable Point Rasterizer with Blending, Differentiable Point-based Rasterizer with Blending

## Overview
The Differentiable Blending Point Renderer is a point-based rasterization method that replaces fuzzy depth-testing with front-to-back alpha blending to enable the direct optimization of point opacities via backpropagation [S1]. It allows the system to automatically identify and remove redundant or outlier points by driving their alpha values toward zero, serving as the core rendering engine for the Visual Error Tomography (VET) pipeline [S1]. Unlike sibling methods that rely on fixed depth tests or non-differentiable sorting, this approach maintains differentiability through a sigmoid parameterization of opacity, facilitating joint optimization of geometry and appearance [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of points, each defined by a 3D position, a feature vector (encoding color and potentially other attributes), and a raw opacity parameter $\alpha_{raw}$ [S1].
*   Camera parameters (view and projection matrices) [S1].
*   Reference images for loss computation [S1].

**Outputs:**
*   Rendered color images $C_{u,v}$ [S1].
*   Gradients with respect to point positions, feature vectors, and raw opacities $\alpha_{raw}$ [S1].

### Data Structures
*   **Point Cloud:** A collection of points. Each point stores:
    *   Position: 3D vector $(x, y, z)$ [S1].
    *   Feature Vector: Encodes color $c_m$ and other properties [S1].
    *   Raw Opacity: A scalar $\alpha_{raw} \in (-\infty, \infty)$ [S1].
*   **Per-Pixel Linked Lists:** For each pixel $(u, v)$, a list of indices pointing to points projected onto that pixel [S1].
*   **Sorted Lists:** The per-pixel lists sorted by depth (distance from camera) [S1].

### Parameters
*   **Sigmoid Scaling:** The opacity transformation uses a scaling factor. The formula provided is $\alpha = \text{sigmoid}((10 + f \cdot t) \cdot \alpha_{raw})$ [S1].
    *   $f$: A factor (value not explicitly defined in the provided text, likely a learnable or fixed scaling constant) [S1].
    *   $t$: A time or epoch-dependent factor (value not explicitly defined in the provided text) [S1].
    *   Note: The sources state alpha is stored in $(-\infty, \infty)$ and converted to $[0, 1]$ using sigmoid for differentiability [S1]. The specific coefficients $(10 + f \cdot t)$ are cited from the source [S1].
*   **Transparency Threshold:** Points with high transparency (low opacity) are considered outliers and removed. The exact threshold value is not specified in the sources [S1].

### Formulas
The final color $C_{u,v}$ for a pixel is computed using front-to-back alpha blending [S1]:

$$
C_{u,v} = \sum_{m=1}^{|\lambda|} T_m \cdot \alpha_m \cdot c_m
$$

Where:
*   $\lambda$ is the sorted per-pixel linked list of points [S1].
*   $m$ is the index in the sorted list, from $1$ to $|\lambda|$ [S1].
*   $\alpha_m$ is the alpha value of point $m$, computed as $\alpha_m = \text{sigmoid}((10 + f \cdot t) \cdot \alpha_{raw, m})$ [S1].
*   $c_m$ is the color of point $m$ [S1].
*   $T_m$ is the transmission variable, representing the amount of light passing through all previous points in the list:
    $$
    T_m = \prod_{i=1}^{m-1} (1 - \alpha_i)
    $$
    With $T_1 = 1$ [S1].

### Steps
1.  **Initialization:** Initialize the point cloud with positions, opacities (as raw values $\alpha_{raw}$), and feature vectors [S1].
2.  **Projection:** Project all 3D points to 2D screen space to determine which pixels they affect [S1].
3.  **Collection:** Render points to per-pixel linked lists. This is implemented efficiently using a custom CUDA per-pixel counting and collection pass [S1].
4.  **Sorting:** Sort the points in each per-pixel linked list by depth (front-to-back) using fast GPU bitonic sorting [S1].
5.  **Opacity Parameterization:** Store opacities as raw values $\alpha_{raw} \in (-\infty, \infty)$ [S1].
6.  **Blending:** For each pixel, iterate through the sorted list. Compute $\alpha_m$ from $\alpha_{raw, m}$ using the sigmoid function. Compute $T_m$ and accumulate the color contribution $T_m \cdot \alpha_m \cdot c_m$ [S1].
7.  **Loss Computation:** Compute the loss between the rendered image and the reference image [S1].
8.  **Backpropagation:** Backpropagate the loss to optimize point descriptors, positions, and raw opacities $\alpha_{raw}$ [S1].
9.  **Pruning (Post-Optimization):** During or after optimization, points that receive high transparency (low $\alpha_m$) are identified as outliers and removed from the point cloud [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid(x):
    """Standard sigmoid function."""
    return 1.0 / (1.0 + math.exp(-x))

def render_points(points, cameras, resolution):
    """
    Renders points using differentiable front-to-back alpha blending.
    
    Args:
        points: List of point objects with attributes:
            - pos: (x, y, z)
            - color: (r, g, b)
            - alpha_raw: float in (-inf, inf)
        cameras: Camera parameters for projection
        resolution: (width, height)
        
    Returns:
        rendered_image: (height, width, 3) color image
    """
    width, height = resolution
    image = np.zeros((height, width, 3))
    
    # Step 1: Project points and collect into per-pixel lists
    # This step involves GPU-based counting and collection [S1]
    # For pseudocode, we assume a helper that returns a list of lists
    # pixel_lists[u][v] = [point_index_1, point_index_2, ...]
    pixel_lists = project_and_collect(points, cameras, resolution)
    
    # Step 2: Sort each list by depth (front-to-back) [S1]
    # This step involves GPU-based bitonic sorting [S1]
    sorted_lists = []
    for u in range(width):
        sorted_lists.append([])
        for v in range(height):
            plist = pixel_lists[u][v]
            # Sort points by depth (distance from camera)
            sorted_points = sorted(plist, key=lambda p: depth(p.pos, cameras))
            sorted_lists[u][v] = sorted_points
            
    # Step 3: Blend colors front-to-back [S1]
    for u in range(width):
        for v in range(height):
            sorted_pts = sorted_lists[u][v]
            if not sorted_pts:
                continue
                
            # Transmission variable T_m
            # T_m = product_{i=1}^{m-1} (1 - alpha_i)
            # We accumulate the color contribution
            
            current_transmission = 1.0
            pixel_color = np.array([0.0, 0.0, 0.0])
            
            for point in sorted_pts:
                # Convert raw opacity to [0, 1] using sigmoid [S1]
                # Formula: alpha = sigmoid((10 + f * t) * alpha_raw)
                # Note: f and t are parameters from the source [S1]
                # For implementation, these must be defined or learned.
                # Here we assume a generic scaling factor 'scale' for clarity,
                # but the source specifies (10 + f * t) [S1].
                scale = 10.0 # Placeholder for (10 + f * t) if f,t not provided
                alpha = sigmoid(scale * point.alpha_raw)
                
                color = point.color
                
                # Contribution: T_m * alpha_m * c_m
                contribution = current_transmission * alpha * color
                pixel_color += contribution
                
                # Update transmission for next point
                current_transmission *= (1.0 - alpha)
                
            image[v, u] = pixel_color
            
    return image

def prune_points(points, threshold=0.01):
    """
    Removes points with opacity below a threshold.
    Points with high transparency (low opacity) are considered outliers [S1].
    """
    surviving_points = []
    for point in points:
        # Compute current opacity for pruning decision
        # Using the same formula as rendering
        scale = 10.0 # Placeholder for (10 + f * t)
        alpha = sigmoid(scale * point.alpha_raw)
        
        if alpha > threshold:
            surviving_points.append(point)
            
    return surviving_points
```

### Edge Cases and Pitfalls
*   **Precision:** The use of sigmoid on raw opacities ensures gradients flow through the $[0, 1]$ range, but extreme raw values may saturate, leading to vanishing gradients [S1].
*   **Sorting Stability:** The correctness of the blending depends on the correct depth sorting. Incorrect sorting leads to incorrect occlusion handling [S1].
*   **Outlier Removal:** The threshold for pruning is not specified in the sources. Choosing an inappropriate threshold may remove valid geometry or fail to remove outliers [S1].
*   **Multi-resolution:** The rendering is performed four times with progressively lower resolutions, fed into a U-Net. This requires careful handling of resolution scaling and memory [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time:** The rendering involves per-pixel sorting, which is typically $O(N \log N)$ per pixel in the worst case, but optimized with GPU bitonic sorting [S1]. The multi-resolution rendering adds overhead [S1].
*   **Memory:** Requires storing per-pixel linked lists and sorted lists. The backward pass requires storing the sorted per-pixel lists [S1].

### Measured Performance
*   Not given in the sources.

### Scaling
*   The method uses a multi-resolution approach (four resolutions) and a U-Net, which may scale differently than single-resolution renderers [S1].

### Strengths and Limitations
*   **Strengths:** Enables automatic optimization of point opacities, allowing for effective outlier removal and point cloud cleaning without manual topological operations [S1]. Integrates well with Visual Error Tomography for point spawning [S1].
*   **Limitations:** Requires custom CUDA implementations for efficient per-pixel counting, collection, and sorting [S1]. The exact parameters for opacity scaling are not fully detailed in the sources [S1].

### When to Use
*   Use when the goal is to optimize a point cloud for rendering quality while simultaneously cleaning it of outliers, particularly in the context of the VET pipeline [S1].

## Background

### Origin
*   The method is based on the work of Rückert et al. [2022] and was introduced by Linus Franke, Darius Rückert, Laura Fink, Matthias Innmann, and Marc Stamminger in the paper "VET: Visual Error Tomography for Point Cloud Completion and High-Quality Neural Rendering" published at Siggraph Asia 2023 [S1].

### History and Development
*   The approach was developed to address deficiencies in point clouds obtained from structure-from-motion, such as outliers and missing geometry [S1]. It combines differentiable rendering with Visual Error Tomography to iteratively clean and spawn points [S1].

### Usage
*   Used as the core rendering and optimization engine in the VET pipeline to clean initial point clouds and refine geometry during training [S1].

## Sources

[S1] VET: Visual Error Tomography for Point Cloud Completion and High-Quality Neural Rendering, https://doi.org/10.1145/3610548.3618212, fetched 2026-10-26
