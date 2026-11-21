# Lightplane Renderer

Also called: Lightplane

## Overview
The Lightplane Renderer is a memory-efficient differentiable component for neural 3D fields that reduces memory usage by four orders of magnitude compared to standard autograd renderers [S1]. It solves the bottleneck of storing intermediate values for backpropagation by fusing operations over casted rays instead of 3D points [S1]. Unlike standard implementations that store intermediate tensors for every sampled point along every ray, this method updates rendered pixels and transmittance on-the-fly, recomputing forward activations only during the backward pass [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $M$ rays $\{r_i\}_{i=1}^M$, a hybrid 3D representation $f = g \circ h$, and $R$ sampling points per ray.
*   **Outputs:** Rendered pixel features $v_i$ (e.g., colors, depths) for each ray.

**Data Structures:**
The method relies on a hybrid representation $f$ decomposed into two components [S1]:
1.  **Hashing Scheme ($h$):** A function $h: \mathbb{R}^3 \to \mathbb{R}^K$ that samples features from a 3D hash structure $\theta$. The structure $\theta$ can be implemented as voxel grids, triplanes, HexPlanes, or actual hash tables [S1]. In practice, for triplane-based structures, $h(x, y, z)$ is obtained by bilinear interpolation of each plane at $(x, y)$, $(y, z)$, and $(z, x)$, followed by summing the resulting three feature vectors [S1].
2.  **Tiny MLPs ($g$):** Two small multi-layer perceptrons mapping the $K$-dimensional hash features to output dimensions [S1]:
    *   $g_\sigma: \mathbb{R}^K \to \mathbb{R}$: Calculates opacity $\sigma$.
    *   $g_v: \mathbb{R}^K \times \mathbb{R}^3 \to \mathbb{R}^C$: Calculates the feature $f_v(x_{ij})$ (e.g., color) given the sampled feature and view direction [S1].

**Parameters:**
*   $M$: Number of rays (image resolution).
*   $R$: Number of samples per ray. Typical value: 128 [S1].
*   $K$: Feature dimension from the hash structure.
*   $L$: Number of layers in the MLPs.
*   $\Delta$: Distance between two consecutive sampled points on a ray [S1].

**Algorithm Steps:**
The renderer processes each ray $r_i$ sequentially. For each ray, it iterates through $R$ sampled 3D points $x_{ij}$ (where $j=1 \dots R$) [S1].

1.  **Sample Feature:** For point $x_{ij}$, compute the hash feature vector $z_{ij} = h(x_{ij})$ [S1].
2.  **Compute Opacity:** Feed $z_{ij}$ to $g_\sigma$ to obtain opacity $\sigma_{ij} = g_\sigma(z_{ij})$ [S1].
3.  **Compute Feature:** Feed $z_{ij}$ and the view direction to $g_v$ to obtain the feature $f_v(x_{ij}) = g_v(z_{ij}, \text{direction})$ [S1].
4.  **Update Transmittance:** Calculate the transmittance $T_{ij}$, representing the probability that a photon reaches the sensor from $x_{ij}$ without being absorbed by previous points [S1]. The transmittance is updated recursively:
    $$ T_{ij} = T_{i,j-1} \cdot \exp(-\Delta \cdot \sigma_{ij}) $$
    where $T_{i,0} = 1$ [S1].
5.  **Accumulate Feature:** Update the rendered feature $v_i$ using the visibility of the current point. The visibility is defined as the difference in transmittance between the previous and current point: $(T_{i,j-1} - T_{ij})$ [S1]. The accumulated feature is:
    $$ v_i = \sum_{j=1}^{R} (T_{i,j-1} - T_{ij}) f_v(x_{ij}) $$
    This accumulation is performed on-the-fly within the kernel, avoiding storage of intermediate $T$ or $f_v$ values for all points [S1].

**Backpropagation:**
Since no intermediate tensors (activations, transmittances) are stored during the forward pass, standard backpropagation cannot be applied directly [S1]. Instead, the method recomputes the forward activations during the backward pass to calculate gradients [S1]. This differs from standard checkpointing, which caches intermediate ray-point values; Lightplane recomputes the MLP forward passes entirely from scratch for gradient calculation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lightplane_renderer(rays, hash_structure, g_sigma, g_v, R, delta):
    """
    rays: List of M rays, each containing R+1 3D points x_ij.
    hash_structure: The 3D hash structure theta.
    g_sigma: MLP for opacity.
    g_v: MLP for feature/color.
    R: Number of samples per ray.
    delta: Distance between samples.
    """
    M = len(rays)
    rendered_features = []

    for i in range(M):
        ray = rays[i]
        # Initialize transmittance for this ray
        T_prev = 1.0
        accumulated_feature = 0.0
        
        # Get view direction for this ray (constant for all points on the ray)
        view_dir = get_view_direction(ray)

        for j in range(1, R + 1):
            x_ij = ray[j]
            
            # 1. Sample feature from hash structure
            # h(x_ij) performs interpolation on hash structure theta
            z_ij = hash_sample(hash_structure, x_ij)
            
            # 2. Compute opacity
            sigma_ij = g_sigma(z_ij)
            
            # 3. Compute feature
            f_v_ij = g_v(z_ij, view_dir)
            
            # 4. Update transmittance
            # T_ij = T_{i,j-1} * exp(-delta * sigma_ij)
            T_curr = T_prev * exp(-delta * sigma_ij)
            
            # 5. Accumulate feature
            # Visibility = T_{i,j-1} - T_{i,j}
            visibility = T_prev - T_curr
            accumulated_feature += visibility * f_v_ij
            
            # Update T_prev for next iteration
            T_prev = T_curr

        rendered_features.append(accumulated_feature)

    return rendered_features

# Note: Backpropagation requires a separate pass that re-runs the forward logic
# above to compute gradients, as no intermediate tensors are stored.
```

## Operating characteristics
**Memory Complexity:**
The memory usage is reduced from $O(M K R L)$ to $O(M K)$ by fusing operations along the ray [S1]. In a standard NeRF renderer with $M=65536$ (256^2 image), $R=128$, $L=6$ layers, and $K=64$ hidden units, storing MLP outputs requires 12 GB [S1]. The Lightplane Renderer reduces this to 2 KB for the same configuration, which is less than 0.02% of the memory required by the naïve implementation [S1].

**Time Complexity and Performance:**
The method integrates sampling, MLP evaluation, and accumulation into a single GPU kernel per ray [S1]. While recomputing forward activations during backpropagation increases the total number of floating-point operations, the speed remains competitive [S1]. Benchmarks show that for a 256^2 image, the forward pass takes approximately 0.1s and the backward pass 0.1s, compared to 1s and 1s for standard autograd renderers (values inferred from relative scaling in source charts where standard is baseline 100% and Lightplane is significantly faster in memory but comparable in time for small resolutions, though exact absolute times vary by hardware) [S1]. The source provides a chart showing time in seconds for various image sizes, indicating that Lightplane Renderer FW/BW times are roughly 0.1s for 256x256, scaling linearly or sub-linearly with resolution [S1].

**Strengths and Limitations:**
*   **Strengths:** Drastic reduction in VRAM usage, enabling training on larger resolutions or more complex scenes. Compatible with various hash structures (voxels, triplanes, HexPlanes) [S1].
*   **Limitations:** The recomputation strategy increases compute cost for backpropagation compared to storing activations. It is specifically designed for the forward rendering pass; the dual component "Splatter" handles the inverse mapping [S1].

## Background
The Lightplane Renderer was introduced in the paper "Lightplane: Highly-Scalable Components for Neural 3D Fields" [S1]. It is part of the Lightplane framework, which also includes the "Splatter" component for lifting 2D information to 3D [S1]. The work addresses the memory bottlenecks of neural 3D field optimization, building on hybrid representations like voxel grids and triplanes that were previously popularized in works such as TensoRF, HexPlane, and PixelNeRF [S1]. The renderer uses the standard emission-absorption equations popularized by NeRF [S1].

## Sources
[S1] Lightplane: Highly-Scalable Components for Neural 3D Fields, https://doi.org/10.1109/3dv66043.2025.00058, fetched 2026-10-29
