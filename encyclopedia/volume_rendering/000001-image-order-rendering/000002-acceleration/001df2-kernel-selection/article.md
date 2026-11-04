# Kernel Selection
Also called: Coarse Rasterization for Ellipsoid Selection, Ellipsoid Kernel Selection, Gaussian Kernel Selection, Optional coarse rasterization stage, Ray Tracing Kernel Selection

## Overview
Kernel selection is a pre-processing and filtering strategy used in volume rendering with Gaussian ellipsoids to reduce computational cost by tracing only a subset of primitives that contribute non-trivially to the final image [S1, unofficial]. It solves the problem of expensive full-scene ray tracing by discarding distant or low-density ellipsoids before the blending stage [S1, unofficial]. Unlike [Ray Intersection Selector (RIS)](3092-ray-intersection-selector-ris.md) which analytically computes precise ray-primitive intersections to identify high-density regions, kernel selection relies on a coarse rasterization of bounding boxes and density thresholds to select a fixed number of nearest kernels per pixel [S1, unofficial].

## How it works
**Inputs:**
- Set of $N$ Gaussian ellipsoids, each defined by center $\mathbf{c}_k$, covariance $\boldsymbol{\Sigma}_k$, and opacity/density parameters.
- Camera parameters: focal length $F$, image dimensions $h \times w$.
- Selection parameters: density threshold $\eta$, maximum number of kernels to trace $K'$.

**Outputs:**
- A mapping from each pixel (or low-resolution tile) to a subset of at most $K'$ ellipsoid indices to be traced.

**Data Structures:**
- `PixelKernels`: A 2D array of lists, indexed by $(u, v)$, storing indices of selected ellipsoids. Dimensions are $W/8 \times H/8$ (low-resolution grid).
- `CandidateList`: A list of tuples `(distance, kernel_index)` for ellipsoids passing the density check.

**Steps:**

1. **Projection and Bounding Box Calculation:**
   For each ellipsoid $k$:
   a. Project the center $\mathbf{c}_k$ to screen coordinates.
   b. Compute the distance $d_z$ from the camera to the center of the ellipsoid.
   c. Compute the height $b_h$ and width $b_w$ of the maximum bounding box of the ellipsoid in 2D screen coordinates using:
      $$ [b_h \quad b_w \quad \cdot] = \frac{\log(-\eta)}{d_z} \cdot \boldsymbol{\Omega} \cdot \boldsymbol{\Sigma}_k^{-1} \cdot \boldsymbol{\Omega} $$
      where $\boldsymbol{\Omega}$ is the projection matrix from camera coordinate to screen:
      $$ \boldsymbol{\Omega} = \begin{bmatrix} \frac{2\cdot F}{h} & 0 & 0 \\ 0 & \frac{2\cdot F}{w} & 0 \\ 0 & 0 & 1 \end{bmatrix} $$
      [S1, unofficial]

2. **Coarse Rasterization:**
   a. Initialize `PixelKernels` for a resolution $8\times$ smaller than the final image (i.e., width $W/8$, height $H/8$).
   b. Rasterize the bounding boxes $[b_h, b_w]$ onto this low-resolution grid.
   c. For each pixel $(u, v)$ in the low-res grid, record all ellipsoids whose bounding boxes cover it. This creates a preliminary set of candidate kernels for each pixel.

3. **Density Filtering:**
   a. For each candidate kernel $k$ associated with a pixel, evaluate if it has a non-trivial contribution.
   b. Record ellipsoids where the maximum density $e q_k > \eta$. The term $e q_k$ refers to the peak density value of the Gaussian kernel [S1, unofficial].
   c. If the source implies $q_k$ is the density value, the condition is $q_k > \eta$. The text states "maximum density $e q_k > \eta$", which likely denotes the exponential density term or simply the density value $q_k$ compared against threshold $\eta$. Given the formula uses $\log(-\eta)$, $\eta$ is a small positive fraction (e.g., 0.01), so $-\eta$ is negative, making $\log(-\eta)$ undefined in real numbers unless $\eta$ is treated as a magnitude or the formula implies $\log(1/\eta)$ or similar. However, we must stick to the source text: "maximum density $e q_k > \eta$". We interpret this as checking if the kernel's density exceeds $\eta$.

4. **Distance Sorting and Selection:**
   a. For each pixel, collect all recorded ellipsoids from Step 3.
   b. Calculate the distance $l_k$ from the ray origin (camera center) to the 1D Gaussian center of each ellipsoid.
   c. Sort the recorded kernels by $l_k$ in ascending order.
   d. Select the $K'$ nearest ellipsoids.

5. **Assignment:**
   Assign the selected $K'$ kernels to the corresponding pixel in `PixelKernels`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_bounding_box_dims(d_z, eta, Omega, Sigma_inv):
    """
    Computes height and width of the bounding box in screen coordinates.
    d_z: distance from camera to ellipsoid center
    eta: density threshold
    Omega: 3x3 projection matrix
    Sigma_inv: 2x2 inverse covariance matrix (projected)
    Returns: (b_h, b_w)
    """
    # Note: The source formula [bh bw .] = ... suggests a vector multiplication.
    # Assuming Sigma_inv is the relevant 2x2 part or the formula implies extracting the first two components.
    # The source writes: [bh bw .] = (log(-eta)/dz) * Omega * Sigma_inv * Omega
    # This implies Sigma_inv is 3x3 or the result is projected.
    # We assume standard 3D covariance projection to 2D screen space for the bounding box.
    # However, to be strictly faithful to the provided text which uses Sigma^{-1} directly:
    
    term = math.log(-eta) / d_z
    # The source formula is ambiguous regarding dimensions. 
    # Omega is 3x3. Sigma^{-1} is likely 3x3 (3D covariance inverse).
    # The result [bh bw .] is a 3-vector where the third component is ignored or used for depth.
    
    # Let's assume Sigma_inv is 3x3.
    # Vector v = Omega @ Sigma_inv @ Omega
    # This results in a 3x3 matrix. The formula likely implies taking the diagonal or specific elements.
    # However, the text says "height bh and width bw ... computed via: [bh bw .] = ..."
    # This notation usually means the first two elements of the resulting vector/matrix diagonal.
    # Let's assume the operation yields a vector where the first two components are bh and bw.
    
    # Re-reading carefully: "compute the height bh and width bw ... via: [bh bw .] = ..."
    # It is highly likely that Sigma^{-1} here refers to the 2D projected covariance inverse, 
    # or the formula is a simplified representation. 
    # Given the constraint to implement from text, we follow the literal matrix mult.
    
    # If Sigma is 3x3, Omega is 3x3.
    # M = Omega @ Sigma_inv @ Omega
    # bh = M[0,0] * term ? Or is it a vector result?
    # The notation [bh bw .] suggests a column vector.
    # Let's assume the result of the matrix multiplication is treated as a vector of scales?
    # Actually, bounding box size is usually derived from the eigenvalues or diagonal of the projected covariance.
    # The source formula is specific. We implement it as written.
    
    # To avoid guessing the dimensionality mismatch, we note that in PyTorch3D-style rendering,
    # the bounding box is often computed from the projected covariance.
    # We will implement the matrix multiplication as given.
    
    res = term * (Omega @ Sigma_inv @ Omega)
    # The source says [bh bw .] = res. 
    # This implies bh = res[0,0] and bw = res[1,1] if res is diagonal, 
    # or perhaps res is a vector. 
    # Given Omega is diagonal-like in the source (mostly), and Sigma is symmetric.
    # Let's assume the result is a vector where the first two elements are the dims.
    # But the RHS is a matrix. 
    # Correction: The source might mean the diagonal elements.
    # "height bh and width bw ... computed via: [bh bw .] = ..."
    # We will extract the diagonal elements of the resulting matrix scaled by the term.
    
    bh = res[0, 0]
    bw = res[1, 1]
    return bh, bw

def kernel_selection(ellipsoids, camera_params, eta, K_prime):
    """
    ellipsoids: list of dicts with keys 'center', 'covariance', 'density'
    camera_params: dict with 'F', 'h', 'w'
    eta: float, density threshold
    K_prime: int, max kernels per pixel
    """
    F = camera_params['F']
    h = camera_params['h']
    w = camera_params['w']
    
    # Projection matrix Omega
    Omega = np.array([
        [2*F/h, 0, 0],
        [0, 2*F/w, 0],
        [0, 0, 1]
    ])
    
    # Low-res grid dimensions
    low_h = h // 8
    low_w = w // 8
    
    # Initialize pixel-to-kernel mapping
    pixel_kernels = [[] for _ in range(low_h * low_w)]
    
    # Step 1 & 2: Project and Rasterize
    for k, ell in enumerate(ellipsoids):
        center = ell['center'] # [x, y, z] in camera coords
        cov = ell['covariance'] # 3x3 matrix
        d_z = center[2] # Distance from camera (assuming camera looks down -z or z is depth)
        
        # Compute inverse covariance
        Sigma_inv = np.linalg.inv(cov)
        
        # Compute bounding box dimensions
        # Note: log(-eta) requires eta < 0 if strictly interpreted, but eta is 0.01.
        # The source likely means log(1/eta) or -log(eta) or eta is negative in their convention.
        # Source says: "eta is the threshold for maximum volume density" and "eta = 0.01".
        # Formula: log(-eta). If eta=0.01, -eta=-0.01, log is complex.
        # This is a known issue in the source text. We assume the magnitude |eta| or 1/eta.
        # However, to be faithful, we use math.log(abs(eta)) or similar if allowed by [G].
        # [G] Use abs(eta) to avoid complex numbers, assuming eta is a positive magnitude threshold.
        log_term = math.log(abs(eta)) / d_z
        
        bh, bw = compute_bounding_box_dims(d_z, eta, Omega, Sigma_inv)
        
        # Determine screen coordinates of center
        # Project center using Omega
        center_hom = np.array([center[0], center[1], center[2]])
        center_screen = Omega @ center_hom
        
        # Map to low-res grid
        # Assuming center_screen[0] and center_screen[1] are in normalized or pixel coords?
        # Omega maps to screen. Usually screen coords are in pixels.
        # Let's assume center_screen[0] is x-pixel, center_screen[1] is y-pixel.
        cx = center_screen[0]
        cy = center_screen[1]
        
        # Convert to low-res grid index
        # Bounding box extends bh/2 and bw/2 from center
        # We need to map pixel coords to low-res indices
        # Low-res pixel size is 8x8 in high-res
        low_cx = int(cx / 8)
        low_cy = int(cy / 8)
        
        # Rasterize bounding box onto low-res grid
        # Calculate extent in low-res pixels
        # bh and bw are in screen coordinates (pixels).
        # Low-res grid cell size is 8 pixels.
        half_bh_lr = (bh / 2) / 8
        half_bw_lr = (bw / 2) / 8
        
        min_cy = max(0, int(low_cy - half_bh_lr))
        max_cy = min(low_h - 1, int(low_cy + half_bh_lr))
        min_cx = max(0, int(low_cx - half_bw_lr))
        max_cx = min(low_w - 1, int(low_cx + half_bw_lr))
        
        for y in range(min_cy, max_cy + 1):
            for x in range(min_cx, max_cx + 1):
                idx = y * low_w + x
                pixel_kernels[idx].append(k)

    # Step 3 & 4: Filter and Select
    final_selection = {} # Map from pixel_idx to list of kernel indices
    
    for idx, candidates in enumerate(pixel_kernels):
        if not candidates:
            continue
            
        # Filter by density
        valid_candidates = []
        for k in candidates:
            ell = ellipsoids[k]
            # Check density condition: max density e qk > eta
            # Assuming ell['density'] holds qk or the peak density value
            # Source says "maximum density e qk > eta". 
            # We assume ell['peak_density'] > eta.
            if ell.get('peak_density', 0) > eta:
                # Calculate distance to 1D Gaussian center
                # l_k is distance from camera to center
                center = ell['center']
                l_k = np.linalg.norm(center)
                valid_candidates.append((l_k, k))
        
        # Sort by distance
        valid_candidates.sort(key=lambda x: x[0])
        
        # Select K' nearest
        selected = [k for _, k in valid_candidates[:K_prime]]
        
        if selected:
            final_selection[idx] = selected
            
    return final_selection
```

**Parameters:**
- $K'$: Number of nearest kernels to select per pixel. Default: 20 [S1, unofficial].
- $\eta$: Density threshold. Default: 0.01 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Inadequate $K'$:** Leads to dark regions around the boundary of kernels due to hard cutoff of the boundary [S1, unofficial].
- **Threshold $\eta$:** Decreasing $\eta$ makes the object appear denser (less transparent) but requires a higher $K'$ to avoid artifacts [S1, unofficial].
- **Formula Domain:** The formula uses $\log(-\eta)$. Since $\eta$ is positive (0.01), this is mathematically undefined in real numbers. The implementation must use $\log(1/\eta)$ or $\log(|\eta|)$ with appropriate sign handling, or assume $\eta$ is negative in the source's convention [G].

## Subtopics

### [Ray Intersection Selector (RIS)](3092-ray-intersection-selector-ris.md)
Replaces stochastic sampling with analytical ray-primitive intersection detection. Chooses this when precise surface reconstruction is needed without auxiliary networks.

### [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md)
Uses orthogonal projections to clip occluded geometry.

### [Adaptive Sampling and Resolution](1412-adaptive-sampling-and-resolution.md)
Adjusts sampling density based on image variance.

### [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md)
Stops ray marching when opacity is sufficient.

### [Inference Ray Marching and Integration](2148-inference-ray-marching-and-integration.md)
Optimizes the integration step during inference.

### [Ray-guided volume rendering with missing brick reporting](2815-ray-guided-volume-rendering-with-missing-brick-reporting.md)
Handles missing data in voxel grids.

### [Intra-ray Parallelism](3135-intra-ray-parallelism.md)
Parallelizes sampling along a single ray.

### [Acceleration Structure Rebuild](3266-acceleration-structure-rebuild.md)
Dynamically updates acceleration structures.

### [Spatiotemporal Importance Sampling](4696-spatiotemporal-importance-sampling.md)
Uses temporal coherence for sampling.

### [Temporal Differential Rendering](5499-temporal-differential-rendering.md)
Renders differentially over time.

### [Visibility Culling](6226-visibility-culling.md)
Culls invisible primitives.

### [Parallelism and Distribution](6265-parallelism-and-distribution.md)
Distributes rendering across devices.

### [Performance Optimization Techniques](6301-performance-optimization-techniques.md)
General optimization methods.

### [Hierarchical Grid Traversal](6306-hierarchical-grid-traversal.md)
Traverses hierarchical voxel grids.

## Operating characteristics
- **Time Complexity:** Reduces the number of ray-tracing operations from $O(N)$ to $O(P \cdot K')$, where $P$ is the number of pixels and $K'$ is the selected kernel count. The coarse rasterization step is $O(N)$ for projection and $O(P)$ for rasterization.
- **Memory Complexity:** Requires storage for the low-resolution pixel-to-kernel mapping, $O(P)$.
- **Measured Performance:** Selecting 10% of ellipsoids significantly reduces computational cost [S1, unofficial].
- **Strengths:** Simple to implement; effective at removing distant/low-density primitives; reduces artifacts compared to random sampling if $K'$ is sufficient.
- **Limitations:** Hard cutoff at $K'$ can cause boundary artifacts (dark regions); sensitive to the choice of $K'$ and $\eta$; coarse rasterization may miss small but important kernels if the low-res grid is too coarse.
- **When to Use:** When rendering large scenes with many Gaussian ellipsoids and computational budget is limited.
- **When Not to Use:** When high fidelity at object boundaries is critical and $K'$ cannot be increased; when kernels are very small and sparse.

## Background
Kernel selection was introduced in the context of differentiable volume rendering using Gaussian ellipsoids, specifically in the VoGE (Volume Gaussian Ellipsoid) renderer [S1, unofficial]. It is part of a coarse-to-fine rendering strategy designed to efficiently render Gaussian ellipsoids by gradually reducing the number of ellipsoids interacting with viewing rays [S1, unofficial]. The method builds on standard rasterization techniques (e.g., PyTorch3D) to perform a coarse selection before the more expensive ray tracing stage [S1, unofficial].

## Sources
[S1] VoGE: A Differentiable Volume Renderer using Gaussian Ellipsoids for Analysis-by-Synthesis, https://doi.org/10.48550/arxiv.2205.15401, fetched 2026-10-26 (unofficial)
