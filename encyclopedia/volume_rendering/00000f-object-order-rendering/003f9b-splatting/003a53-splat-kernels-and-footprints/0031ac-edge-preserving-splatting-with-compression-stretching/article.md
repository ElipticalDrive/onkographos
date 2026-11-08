# Edge-Preserving Splatting with Compression/Stretching

Also called: C1 Continuous Splatting, Edge-Enhanced Splatting

## Overview
Edge-Preserving Splatting with Compression/Stretching is a volume rendering technique that sharpens discontinuities in low-resolution or undersampled data by applying non-linear compression and stretching functions to the standard splatting kernel [S1]. Unlike conventional splatting, which produces smooth, blurred edges, this method ensures C1 continuity (continuous first derivative) across splat boundaries, allowing for the reconstruction of sharp features such as cavity walls and small protrusions [S1]. It distinguishes itself from sibling methods by explicitly modeling edge direction and strength to drive view-dependent footprint selection, rather than relying solely on binary classification or fixed kernel shapes [S1].

## How it works
The method modifies the spatial footprint of each splat based on the local edge characteristics of the underlying voxel data. It operates by classifying voxels as "primary" or "secondary" relative to an edge threshold and applying asymmetric compression/stretching to the kernel profile to enhance the gradient at the edge boundary.

### Inputs and Outputs
*   **Inputs:**
    *   Volume data $V(x,y,z)$.
    *   Camera parameters (view direction, projection matrix).
    *   Edge threshold value $T$ (used to distinguish edge regions from interior/exterior).
    *   Precomputed footprint table (texture memory) containing splat profiles for various edge positions and strengths [S1].
*   **Outputs:**
    *   Final 2D image with enhanced edge sharpness.

### Data Structures
1.  **Voxel Attributes:** For each voxel $k$, the following attributes are computed and stored:
    *   Position $\mathbf{p}_k$.
    *   Data value $v_k$.
    *   Gradient $\nabla v_k$ (computed via central differences) [S1].
    *   Edge direction $\mathbf{d}_k = \frac{\nabla v_k}{\|\nabla v_k\|}$ [S1].
    *   Contour distance $d_k$: The signed distance from the voxel center to the iso-surface defined by $T$. Calculated as $d_k = \frac{T - v_k}{\|\nabla v_k\|}$ [S1].
    *   Normalized edge position $x_k$: A value in $[0, 1]$ representing the relative position of the voxel within the edge transition region. This is derived from $d_k$ and the estimated edge width [S1].
    *   Edge strength $s_k$: Derived from the projected length of the edge direction onto the viewing plane. If the viewing direction is parallel to the edge, $s_k$ is negligible [S1].
    *   Classification:
        *   **Primary Kernel:** If $v_k > T$ (or consistently defined as the "higher" value side) [S1].
        *   **Secondary Kernel:** If $v_k \le T$ [S1].

2.  **Precomputed Footprint Table:**
    *   A set of 1D kernel profiles $g(x)$ indexed by edge position $x \in [0,1]$ and edge strength $s$.
    *   These profiles are stored in texture memory for hardware-accelerated sampling [S1].

### Algorithm Steps

1.  **Preprocessing (Edge Detection):**
    *   Compute the gradient $\nabla v_k$ for all voxels using central differences [S1].
    *   Estimate the contour distance $d_k$ using the gradient and the difference between the voxel value and the threshold $T$:
        $$ d_k = \frac{T - v_k}{\|\nabla v_k\|} $$
    *   Estimate the edge position $x_k$ by normalizing $d_k$ within the expected edge width. Trilinear interpolation is used to refine position estimates if necessary [S1].

2.  **View-Dependent Processing:**
    *   For each splat, project the normalized edge direction $\mathbf{d}_k$ onto the viewing plane.
    *   Calculate the projected edge length to determine the edge strength $s_k$.
    *   If the viewing direction is parallel to the edge direction, the edge is imperceptible; treat the splat as a standard non-edge splat [S1].

3.  **Footprint Selection:**
    *   Based on the classification (Primary/Secondary), the edge position $x_k$, and the strength $s_k$, select the appropriate precomputed footprint profile from the texture table [S1].
    *   The profile is a function $h(x)$ that modifies the base kernel $g(x)$.

4.  **Kernel Modification (Compression/Stretching):**
    *   The core of the method is the mapping function $map(x)$ that transforms the coordinate $x$ of the base kernel $g$ to produce the enhanced kernel $h$.
    *   The mapping ensures C1 continuity. The condition for C1 continuity at the boundaries $x=0$ and $x=1$ is:
        $$ h'(x) = g'(map(x)) \cdot map'(x) $$
    *   To preserve continuity with the adjacent splat, we require $map'(0) = 1$ and $map'(1) = 1$. Thus:
        $$ h'(0) = g'(map(0)) \cdot 1 = g'(0) $$
        $$ h'(1) = g'(map(1)) \cdot 1 = g'(1) $$
    *   This ensures that the slope of the enhanced kernel matches the slope of the base kernel at the boundaries, preventing artifacts at splat edges [S1].

5.  **Rasterization:**
    *   Transform the voxel position $\mathbf{p}_k$ to screen space.
    *   Render the splat using the selected footprint profile $h(x)$ as a texture or analytical function.
    *   Apply view-dependent lighting *only* to voxels labeled as primary or secondary edge splats [S1].

### Formulas and Definitions

**Base Kernel:**
Let $g(x)$ be the standard radial basis function (e.g., Gaussian) centered at 0.

**Mapping Function:**
The function $map(x)$ compresses the region near the edge and stretches the region away from it (or vice versa) to sharpen the transition.
For a Primary Kernel (higher data value), the profile $h(x,y)$ is defined relative to the edge position. The source provides specific piecewise definitions for different intervals. For example, for a primary kernel with lower data value (note: source text has conflicting labels "primary... lower" vs "primary... larger", but defines the logic as "larger or lower"):

If we define the edge transition from $x=-1$ to $x=1$ (normalized):
For a splat where the edge is at the right boundary (Primary, higher value):
$$ h(x) = \begin{cases} g(map(x+1)-1) & \text{if } -1 \le x \le 0 \\ g(x) & \text{otherwise} \end{cases} $$

For a splat where the edge is at the left boundary (Secondary, lower value):
$$ h(x) = \begin{cases} g(map(x-1)+1) & \text{if } 0 \le x \le 1 \\ g(x) & \text{otherwise} \end{cases} $$

The function $map(x)$ is designed such that $map(0)=0$ and $map(1)=1$ (or equivalent shifts) and $map'(0)=1, map'(1)=1$. The exact shape of $map(x)$ depends on the edge strength; stronger edges result in more aggressive compression/stretching [S1].

**C1 Continuity Condition:**
$$ \frac{dh}{dx} = \frac{dg}{du} \frac{du}{dx} \quad \text{where } u = map(x) $$
At boundaries where $map'(x)=1$, $\frac{dh}{dx} = \frac{dg}{du}$. This matches the derivative of the adjacent splat's base kernel [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def edge_preserving_splatting(volume, camera, threshold):
    """
    Renders volume using edge-preserving splatting.
    """
    # 1. Precompute edge attributes for all voxels
    voxels = []
    for k in volume.voxels:
        # Compute gradient using central differences
        grad = compute_gradient_central_diff(volume, k)
        
        # Compute contour distance
        # d = (T - v) / ||grad||
        dist = (threshold - k.value) / norm(grad)
        
        # Normalize distance to edge position x in [0, 1]
        # Assuming edge width W, x = dist / W + 0.5 (clamped)
        x_edge = clamp(dist / k.edge_width + 0.5, 0.0, 1.0)
        
        # Classify as Primary or Secondary
        is_primary = k.value > threshold
        
        # Project edge direction to view plane for strength
        edge_dir = normalize(grad)
        view_dir = camera.direction
        # Project edge_dir onto plane perpendicular to view_dir
        projected_edge = edge_dir - dot(edge_dir, view_dir) * view_dir
        edge_strength = norm(projected_edge)
        
        # If view is parallel to edge, strength is near 0, treat as normal
        if edge_strength < epsilon:
            edge_strength = 0.0
            
        voxels.append({
            'pos': k.pos,
            'is_primary': is_primary,
            'x_edge': x_edge,
            'strength': edge_strength,
            'value': k.value,
            'grad': grad
        })
        
    # 2. Sort voxels back-to-front (standard splatting requirement)
    voxels = sort_back_to_front(voxels, camera)
    
    # 3. Rasterize
    image = zeros(camera.resolution)
    
    for v in voxels:
        # Select footprint from precomputed table
        # Table indexed by (is_primary, x_edge, strength)
        footprint = get_footprint_from_texture(v.is_primary, v.x_edge, v.strength)
        
        # Transform position to screen space
        screen_pos = project(v.pos, camera)
        
        # Render splat using footprint
        # The footprint function h(x) is sampled here
        render_splat(screen_pos, footprint, v.value, image)
        
        # Apply lighting only to edge splats
        if v.strength > epsilon:
            apply_lighting(screen_pos, v.grad, image)
            
    return image

def compute_gradient_central_diff(volume, voxel):
    # Standard central difference: (f(x+1) - f(x-1)) / 2
    # Returns 3D vector
    pass

def get_footprint_from_texture(is_primary, x_edge, strength):
    # Returns the 1D profile h(x) or texture coordinates to sample it
    # The texture contains precomputed maps based on map(x)
    pass
```

### Edge Cases and Pitfalls
*   **Parallel Viewing Direction:** If the viewing direction is parallel to the edge direction, the edge is imperceptible. The method should fall back to standard splatting or assign zero edge strength to avoid artifacts [S1].
*   **Undersampled Structures:** Small structures (e.g., small protrusions into a cavity) are often undersampled due to high-frequency components in digitized data. This makes edge preservation challenging as the gradient estimation may be inaccurate. Adaptive multivariate representations are suggested as a future improvement [S1].
*   **Edge Detection Accuracy:** The method requires fairly accurate edge detection. Iso-contouring produces sharp discontinuities, which helps in estimating edge direction and contour distance. For noisy data (CT/MRI), advanced edge detection or segmentation is indispensable [S1].
*   **Definition Consistency:** The distinction between "primary" and "secondary" kernels depends on whether the data value is larger or smaller than the threshold. This definition must be consistent throughout the application, as the compression/stretching function is asymmetric [S1].

## Operating characteristics
*   **Time Complexity:** The per-voxel preprocessing (gradient, edge detection) is $O(N)$ where $N$ is the number of voxels. Rendering is $O(M)$ where $M$ is the number of splats rasterized. The footprint lookup is $O(1)$ via texture sampling.
*   **Memory Complexity:** Requires storage for the volume data, gradient fields, and the precomputed footprint texture table. The table size depends on the resolution of edge position and strength discretization.
*   **Performance:** The method is designed for interactive rendering by precomputing splats and loading them into texture memory [S1]. It adds overhead compared to standard splatting due to the edge detection and footprint selection steps, but remains efficient on GPU hardware.
*   **Strengths:** Significantly improves visual quality for low-resolution or undersampled data by producing sharp edges and enhancing details on cavity walls and small structures [S1].
*   **Limitations:** Relies heavily on the accuracy of edge detection. Fails to recover details that are fundamentally undersampled (Nyquist limit violations). Computationally more expensive than basic splatting due to preprocessing and texture lookups.

## Background
The method was introduced in the paper "Edge Preservation in Volume Rendering Using Splatting" [S1]. It was applied to the Sinus Cavity dataset (derived from the Visible Human dataset) and the Hipip (HIgh Potential Iron Protein) dataset. The technique was used for virtual navigation through sinus cavity data to enhance sidewall details and coarse resolution cross-sections. The work addresses the limitation of conventional splatting, which produces smooth curves and blurs sharp edges, particularly in low-resolution data [S1].

## Sources
[S1] Edge Preservation in Volume Rendering Using Splatting, https://web.eecs.utk.edu/~huangj/papers/edgesplat.pdf, fetched 2026-10-28
