# Warp-Space Affine Mixture

Also called: Affine mixture warp model

## Overview
Warp-Space Affine Mixture is a warping method where affine mixture weights are computed in warped space, outperforming models with no warping, convolutional warps, or world-space mixture [S1]. It addresses the limitations of single affine warps and freely-varying template points by using a spatial mixture of affine transformations to produce an inverse warp field, effectively extending linear blend skinning to volumetric space [S1].

## How it works
The method constructs an inverse warp field $\mathbf{W}^{-1}(\mathbf{x})$ as a weighted sum of individual affine transformations. The weights are determined by sampling a weight volume *after* the affine transformation has been applied to the coordinate, ensuring the weighting function resides in the warped space [S1].

**Inputs and Outputs:**
*   Input: A 3D coordinate $\mathbf{x}$ in the target space.
*   Output: A transformed coordinate $\mathbf{x}' = \mathbf{W}^{-1}(\mathbf{x})$ in the source space.
*   Parameters: A set of $N$ affine transformations defined by rotation $R_i$, scale $s_i$, and translation $t_i$, and corresponding weight volumes $w_i$.

**Data Structures:**
*   **Affine Parameters:** For each warp $i \in \{1, \dots, N\}$:
    *   $R_i$: A $3 \times 3$ rotation matrix, derived from a normalized quaternion vector.
    *   $s_i$: A 3D vector representing element-wise scaling factors.
    *   $t_i$: A 3D translation vector.
*   **Weight Volumes:** $w_i(\mathbf{x})$ is a 3D voxel grid storing the unnormalized weight for warp $i$ at coordinate $\mathbf{x}$. The resolution is typically $32^3$ to ensure smoothness via trilinear interpolation [S1].
*   **Global Warp:** An additional affine transformation $\{R_g, s_g, t_g\}$ applied to $\mathbf{x}$ before the mixture [S1].

**Formulas:**
The inverse warp is defined as [S1]:
$$
\mathbf{W}^{-1}(\mathbf{x}) = \sum_{i=1}^{N} \mathbf{A}_i(\mathbf{x}) a_i(\mathbf{x})
$$
where $\mathbf{A}_i(\mathbf{x})$ is the $i$-th affine transformation:
$$
\mathbf{A}_i(\mathbf{x}) = R_i(s_i \circ (\mathbf{x} - t_i))
$$
Here, $\circ$ denotes element-wise multiplication. The mixture weight $a_i(\mathbf{x})$ is computed as:
$$
a_i(\mathbf{x}) = \frac{w_i(\mathbf{A}_i(\mathbf{x}))}{\sum_{j=1}^{N} w_j(\mathbf{A}_j(\mathbf{x}))}
$$
Note that $w_i$ is sampled at the transformed location $\mathbf{A}_i(\mathbf{x})$, not at $\mathbf{x}$ [S1].

**Steps:**
1.  **Global Warp:** Apply the global affine transformation to the input coordinate:
    $$ \mathbf{x}_{global} = R_g(s_g \circ (\mathbf{x} - t_g)) $$
2.  **Initialize Accumulator:** Set $\mathbf{x}' = \mathbf{0}$ and $Z = 0$.
3.  **Iterate Warps:** For each warp $i$ from $1$ to $N$:
    a. Compute the affine transformation of the global coordinate:
       $$ \mathbf{x}_i = R_i(s_i \circ (\mathbf{x}_{global} - t_i)) $$
    b. Sample the weight volume $w_i$ at $\mathbf{x}_i$. If $\mathbf{x}_i$ is outside the bounds of the weight volume grid, clamp the sample to the surface value of the volume [S1].
    c. Add to the weighted sum: $\mathbf{x}' \leftarrow \mathbf{x}' + w_i(\mathbf{x}_i) \cdot \mathbf{x}_i$.
    d. Add to the denominator: $Z \leftarrow Z + w_i(\mathbf{x}_i)$.
4.  **Normalize:** Compute the final warped coordinate:
    $$ \mathbf{W}^{-1}(\mathbf{x}) = \frac{\mathbf{x}'}{Z} $$

**Parameters:**
*   Number of warps ($N$): 16 provides sufficient expressiveness [S1].
*   Weight volume resolution: $32^3$ voxel grid [S1].

**Pitfalls:**
*   **Clamping:** Samples outside the weighting volume $w_i$ must be clamped to the surface. If not, warps can get "stuck" early in training if they land outside the volume [S1].
*   **Weight Normalization:** Weights must be non-negative. This is ensured by applying $\exp(\cdot)$ to the output of the fully-connected layers that generate the weight volume parameters [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def warp_space_affine_mixture(x, global_params, warp_params, weight_volumes):
    """
    Computes the warp-space affine mixture.
    
    Args:
        x: Input 3D coordinate (vector).
        global_params: Tuple (R_g, s_g, t_g) for global warp.
        warp_params: List of N tuples (R_i, s_i, t_i) for each warp.
        weight_volumes: List of N 3D voxel grids (numpy arrays or similar).
        
    Returns:
        Transformed coordinate x_prime.
    """
    R_g, s_g, t_g = global_params
    
    # Step 1: Global Warp
    # x_global = R_g * (s_g * (x - t_g))
    x_shifted = x - t_g
    x_scaled = s_g * x_shifted  # Element-wise multiplication
    x_global = R_g @ x_scaled   # Matrix-vector multiplication
    
    x_prime = np.zeros(3)
    Z = 0.0
    
    # Step 2: Iterate over warps
    for i in range(len(warp_params)):
        R_i, s_i, t_i = warp_params[i]
        
        # Compute affine transform A_i(x_global)
        x_i_shifted = x_global - t_i
        x_i_scaled = s_i * x_i_shifted
        x_i = R_i @ x_i_scaled
        
        # Sample weight w_i at x_i
        # Note: w_i is sampled at the transformed location A_i(x)
        w_i = sample_weight_volume(weight_volumes[i], x_i)
        
        # Accumulate
        x_prime += w_i * x_i
        Z += w_i
        
    # Step 3: Normalize
    if Z == 0:
        return x_global # Fallback if all weights are zero
        
    return x_prime / Z

def sample_weight_volume(volume_grid, point):
    """
    Samples a 3D volume at a point, clamping to surface if outside.
    Assumes trilinear interpolation.
    """
    # Check bounds. If outside, clamp to surface value.
    # For implementation simplicity, we assume a helper function
    # that handles the clamping logic described in [S1].
    if is_outside_bounds(point, volume_grid):
        return get_surface_value(volume_grid)
    
    return trilinear_interpolate(volume_grid, point)
```

## Subtopics

### Mixture of Affine Warps
This variant refers to the specific implementation where the warp field decoder uses a spatial mixture of affine warps to avoid overfitting and poor generalization associated with freely-varying template sample points [S1]. It involves computing transformation parameters using 2 fully-connected layers after encoding $z$. Rotation is produced as a normalized quaternion vector transformed into a rotation matrix. Weights are computed by applying $\exp(\cdot)$ to the output of the fully-connected layers for the weighting volume $w_i$ to ensure non-negative weights. An additional global warp $\{R_g, s_g, t_g\}$ is learned by an MLP and applied to $x$ before the warping field [S1].

## Operating characteristics
*   **Complexity:** The method requires $N$ affine transformations and $N$ volume samples per query. With $N=16$ and a $32^3$ grid, the per-ray cost is constant but higher than a single affine warp.
*   **Performance:** The model outperforms no-warp, convolutional warp (transposed convolutions), convolutional warp (trilinear upsampling + convolution), and no-warp affine mixture models [S1]. It is particularly effective on complex dynamic data like hair swing, where convolutional warps fail [S1].
*   **Strengths:** Modeling the weighting volume in warp space is better than in world space [S1]. The low-resolution $32^3$ grid provides smoothness from trilinear interpolation, which helps learning [S1].
*   **Limitations:** Requires learning additional parameters for the global warp and the mixture weights.

## Background
The method originates from "Neural Volumes: Learning Dynamic Renderable Volumes from Images" [S1]. It was developed to address overfitting and poor generalization to novel views in dynamic volume rendering. The approach extends linear blend skinning [Lewis et al. 2000] to volumetric space by using a spatial mixture of affine warps.

## Sources
[S1] Neural Volumes: Learning Dynamic Renderable Volumes from Images, https://arxiv.org/abs/1906.07751, fetched 2026-10-26
