# Contour Rendering

Also called: Contour Lines Rendering, Gradient-based Contouring, Non-Photorealistic Volume Rendering for Contours, Object Contour Visualization

## Overview
Contour Rendering is a non-photorealistic volume rendering technique that visualizes object boundaries in volumetric data by leveraging gradient magnitude and viewing angle, thereby avoiding the need for manual transfer function tuning or explicit iso-surface extraction [S1][S2, unofficial]. Unlike Direct Volume Rendering (DVR), which accumulates color and opacity along rays, Contour Rendering produces a sparse representation where only voxels exhibiting significant gradient changes and favorable surface orientation contribute to the final image, typically via Maximum Intensity Projection (MIP) or alpha-blending [S1]. It is distinct from [Geometric Buffers (G-buffers)](781-geometric-buffers-g-buffers.md) as it does not rely on pre-computed geometric primitives but derives surface cues directly from scalar field derivatives [S1].

## How it works
**Inputs:**
*   Volumetric scalar field $f(x,y,z)$.
*   Viewing direction vector $V$ (normalized).
*   Gradient magnitude threshold $t_g$.
*   Sharpness exponent $n$.
*   Windowing function parameters (defining the range of interest for gradient magnitudes).

**Outputs:**
*   Final pixel color $C_{pixel}$.

**Data Structures:**
*   **Volume Grid:** A 3D array of scalar values.
*   **Gradient Field:** A 3D array of 3D vectors $\nabla f$, computed from the volume grid.
*   **Intensity Field:** A 3D array of floats $I(P, V)$, computed per sample point along rays.

**Step 1: Gradient Reconstruction**
Compute the gradient $\nabla f(P)$ for each voxel $P$. High-quality reconstruction is critical; 4D linear regression is preferred over central differences to reduce noise and artifacts [S1][S2, unofficial].

**Step 2: Windowing Function Application**
Define a windowing function $g(|\nabla|)$ that characterizes the "surfaceness" of a voxel based on its gradient magnitude $|\nabla(P)|$. This function determines the range of gradient magnitudes considered significant [S1].
$$ g(|\nabla(P)|) = \begin{cases} 0 & \text{if } |\nabla(P)| < t_{min} \\ \text{interpolated value} & \text{if } t_{min} \le |\nabla(P)| \le t_{max} \\ 1 & \text{if } |\nabla(P)| > t_{max} \end{cases} $$
*Note: The sources describe $g(|\nabla|)$ generally as a windowing function determining the range of interest [S1]. Specific implementation details for the interpolation curve are not provided in the sources [G].*

**Step 3: View-Dependent Weight Calculation**
Calculate the view-dependent weight $s(P, V)$ for each sample point $P$ along the viewing ray. This weight assigns higher values to voxels where the gradient is perpendicular to the view direction (i.e., facing the viewer) [S1].
$$ s(P, V) = (1 - |\nabla(P) \cdot V|)^n $$
Where:
*   $\nabla(P)$ is the normalized gradient vector at $P$.
*   $V$ is the normalized viewing direction vector.
*   $n$ is an exponent controlling the sharpness of the contour (higher $n$ results in sharper, thinner contours) [S1].

**Step 4: Intensity Calculation**
Compute the view-dependent intensity $I(P, V)$ for each sample point:
$$ I(P, V) = g(|\nabla(P)|) \cdot s(P, V) $$
$$ I(P, V) = g(|\nabla(P)|) \cdot (1 - |\nabla(P) \cdot V|)^n $$
Samples with gradient magnitude $|\nabla(P)| < t_g$ are discarded or assigned zero intensity [S2, unofficial].

**Step 5: Compositing**
There are two primary compositing strategies described:

*   **Strategy A: Maximum Intensity Projection (MIP)**
    For each pixel, project the highest intensity value $I(P, V)$ found along the viewing ray to the image plane. This results in a sparse volume where only significant contours are visible [S1].
    $$ C_{pixel} = \max_{P \in \text{ray}} (I(P, V)) $$
    *Depth Cueing:* To mitigate the lack of depth information in simple MIP, sample intensities can be weighted by their distance $d$ from the viewpoint:
    $$ I_{cued}(P, V) = I(P, V) \cdot e^{-\alpha d} $$
    Where $\alpha$ is a damping factor [S1].
    *Local Maximum Intensity Projection (LMIP):* To prevent stronger, farther contours from hiding weaker, closer ones, LMIP projects the first local maximum intensity along the ray that exceeds a predefined threshold [S1].

*   **Strategy B: Alpha-Blending**
    Assign the value of $g(|\nabla(P)|)$ as both color and opacity, modulated by the intensity $I(P, V)$. Perform standard front-to-back alpha blending. This emphasizes low-curvature surfaces and is suitable for data sets with many high-frequency regions, though fine details may be less visible [S1].
    $$ C_{pixel} = \sum_{i=1}^{N} c_i \cdot \alpha_i \cdot \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    Where $c_i = g(|\nabla(P_i)|)$ and $\alpha_i = I(P_i, V)$ [S1].

**Extension: Level Lines**
To improve spatial impression, additional characteristic lines (level lines) can be rendered. Every $k$-th slice (e.g., $k=20$) is selected. The values of $g(|\nabla|)$ are assigned as intensities to voxels in these slices. These are rendered using MIP in a different color channel to distinguish them from object contours [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def contour_rendering(volume, gradient_field, view_direction, n, t_g, depth_cueing_alpha=0.0):
    """
    Renders contours from volumetric data using gradient magnitude and view direction.
    
    Args:
        volume: 3D array of scalar values.
        gradient_field: 3D array of 3D vectors (gradients).
        view_direction: Normalized 3D vector.
        n: Exponent for view-dependent weight.
        t_g: Gradient magnitude threshold.
        depth_cueing_alpha: Damping factor for depth cueing.
        
    Returns:
        2D array of pixel intensities.
    """
    height, width, depth = volume.shape
    image = np.zeros((height, width))
    
    # Precompute gradient magnitudes and normalized gradients
    grad_mags = np.linalg.norm(gradient_field, axis=-1)
    grad_norms = gradient_field / (grad_mags[:, :, :, np.newaxis] + 1e-8)
    
    for y in range(height):
        for x in range(width):
            max_intensity = 0.0
            
            # Ray traversal along z-axis (simplified for 2D example; 
            # in 3D raycasting, P varies along the ray direction)
            for z in range(depth):
                P = (x, y, z)
                grad_mag = grad_mags[P]
                
                # Step 2 & 3: Filter by threshold and calculate windowing function
                if grad_mag < t_g:
                    continue
                
                # g(|grad|) is assumed to be 1.0 for grad_mag > t_g in simple thresholding,
                # or interpolated based on specific windowing parameters.
                # Here we assume a binary mask for simplicity as per S2's w(|g|) definition,
                # but S1 implies a continuous windowing function g.
                # Using S2's binary threshold for clarity where g is not fully defined:
                g_val = 1.0 if grad_mag > t_g else 0.0
                
                # Step 3: View-dependent weight
                grad_vec = grad_norms[P]
                dot_product = np.abs(np.dot(grad_vec, view_direction))
                s_pv = (1.0 - dot_product) ** n
                
                # Step 4: Intensity
                intensity = g_val * s_pv
                
                # Depth cueing
                if depth_cueing_alpha > 0:
                    distance = np.linalg.norm(np.array(P) - np.array([0, 0, 0])) # Simplified distance
                    intensity *= np.exp(-depth_cueing_alpha * distance)
                
                # Step 5: MIP
                if intensity > max_intensity:
                    max_intensity = intensity
            
            image[y, x] = max_intensity
            
    return image
```

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels along all viewing rays. The sparsity of the volume (only voxels with high gradient and favorable view angle contribute) allows for optimization, such as early ray termination or skipping empty regions, achieving interactive frame rates without specialized hardware [S1].
*   **Memory Complexity:** $O(V)$ for storing the volume and gradient fields, where $V$ is the total number of voxels.
*   **Strengths:**
    *   Does not require transfer function tuning [S1].
    *   Provides clear visualization of multiple object surfaces simultaneously without visual overload [S1].
    *   Interactive performance is achievable on standard hardware [S1].
    *   Effective for abstract, coherent visualization of important information [S2, unofficial].
*   **Limitations:**
    *   Simple MIP lacks depth information, which can be mitigated by depth cueing or LMIP but adds complexity [S1].
    *   High-intensity contours can occlude weaker, closer contours (addressed by LMIP) [S1].
    *   Alpha-blending variant may obscure fine details in high-frequency data [S1].
    *   Quality is highly dependent on accurate gradient estimation [S2, unofficial].
*   **When to use:** When the goal is to visualize object boundaries and structural features in a volumetric dataset without the clutter of direct volume rendering, especially when transfer function tuning is undesirable [S1][S2, unofficial].
*   **When not to use:** When accurate quantitative data values or internal density variations are required, or when the data has low gradient contrast [S1].

## Background
Contour Rendering was introduced as a non-photorealistic rendering technique for volumetric data that relies on gradient magnitude rather than data values [S1]. The method was proposed to address the limitations of Direct Volume Rendering (DVR) and iso-surface extraction, particularly the need for manual parameter tuning and the visual clutter associated with rendering multiple surfaces [S1]. The technique was developed to provide an abstract yet coherent visualization of important information in volumetric datasets, leveraging the fact that object surfaces are characterized by regions with significant changes in data values [S1][S2, unofficial]. The approach was highlighted for its ability to achieve interactive frame rates on standard hardware due to the sparse nature of the resulting volume representation [S1].

## Sources
[S1] Fast Visualization of Object Contours by Non-Photorealistic Volume Rendering, https://doi.org/10.1111/1467-8659.00538, fetched 2026-11-01
[S2] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
