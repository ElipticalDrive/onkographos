# Warp-Space Affine Mixture

Also called: Affine mixture warp model

## Overview
Warp-Space Affine Mixture is a warping method where affine mixture weights are computed in warped space, outperforming models with no warping, convolutional warps, or world-space mixture [S1]. It addresses the limitations of single affine warps and freely-varying template points by using a spatial mixture of affine transformations to produce an inverse warp field, effectively extending linear blend skinning to volumetric space [S1].

## How it works
The method constructs an inverse warp field $\mathbf{W}^{-1}(\mathbf{x})$ as a weighted sum of individual affine transformations [S1]. The weights are determined by sampling a weight volume *after* the affine transformation has been applied to the coordinate, ensuring the weighting function resides in the warped space [S1].

**Inputs and Outputs:**
*   Input: A 3D coordinate $\mathbf{x}$ in the target space [S1].
*   Output: A transformed coordinate $\mathbf{x}' = \mathbf{W}^{-1}(\mathbf{x})$ in the source space [S1].
*   Parameters: A set of $N$ affine transformations defined by rotation $R_i$, scale $s_i$, and translation $t_i$, and corresponding weight volumes $w_i$ [S1].

**Data Structures:**
*   **Affine Parameters:** For each warp $i \in \{1, \dots, N\}$: $R_i$ is a $3 \times 3$ rotation matrix derived from a normalized quaternion vector, $s_i$ is a 3D vector of element-wise scaling factors, and $t_i$ is a 3D translation vector [S1].
*   **Weight Volumes:** $w_i(\mathbf{x})$ is a 3D voxel grid storing the unnormalized weight for warp $i$ at coordinate $\mathbf{x}$ [S1]. The resolution is typically $32^3$ to ensure smoothness via trilinear interpolation [S1].
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
Here, $\circ$ denotes element-wise multiplication [S1]. The mixture weight $a_i(\mathbf{x})$ is computed as:
$$
a_i(\mathbf{x}) = \frac{w_i(\mathbf{A}_i(\mathbf{x}))}{\sum_{j=1}^{N} w_j(\mathbf{A}_j(\mathbf{x}))}
$$
Note that $w_i$ is sampled at the transformed location $\mathbf{A}_i(\mathbf{x})$, not at $\mathbf{x}$ [S1].

**Steps:**
1.  **Global Warp:** Apply the global affine transformation to the input coordinate: $\mathbf{x}_{\text{global}} = R_g(s_g \circ (\mathbf{x} - t_g))$ [S1].
2.  **Initialize Accumulator:** Set $\mathbf{x}' = \mathbf{0}$ and $Z = 0$ [G].
3.  **Iterate Warps:** For each warp $i$ from $1$ to $N$:
    a. Compute the affine transformation of the global coordinate: $\mathbf{x}_i = R_i(s_i \circ (\mathbf{x}_{\text{global}} - t_i))$ [S1].
    b. Sample the weight volume $w_i$ at $\mathbf{x}_i$. If $\mathbf{x}_i$ is outside the bounds of the weight volume grid, clamp the sample to the surface value to prevent warps from getting "stuck" early in training [S1].
    c. Add to the weighted sum: $\mathbf{x}' \leftarrow \mathbf{x}' + w_i(\mathbf{x}_i) \cdot \mathbf{x}_i$ [S1].
    d. Add to the denominator: $Z \leftarrow Z + w_i(\mathbf{x}_i)$ [S1].
4.  **Normalize:** Compute the final warped coordinate: $\mathbf{W}^{-1}(\mathbf{x}) = \mathbf{x}' / Z$ [S1].

**Parameters:**
*   Number of warps ($N$): 16 provides sufficient expressiveness [S1].
*   Weight volume resolution: $32^3$ voxel grid [S1].

**Pitfalls:**
*   **Clamping:** Samples outside the weighting volume $w_i$ must be clamped to the surface; otherwise, warps can get "stuck" early in training if they land outside the volume [S1].
*   **Weight Normalization:** Weights must be non-negative, which is ensured by applying $\exp(\cdot)$ to the output of the fully-connected layers that generate the weight volume parameters [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def warp_space_affine_mixture(x, global_params, warp_params, weight_volumes):
    """
    Computes the warp-space affine mixture.
    Args:
        x: Input 3D coordinate (list/tuple of 3 floats).
        global_params: Tuple (R_g, s_g, t_g) for global warp.
        warp_params: List of N tuples (R_i, s_i, t_i) for each warp.
        weight_volumes: List of N 3D voxel grids (list of lists of lists).
    Returns:
        Transformed coordinate x_prime.
    """
    R_g, s_g, t_g = global_params
    
    # Step 1: Global Warp
    x_shifted = [x[k] - t_g[k] for k in range(3)]
    x_scaled = [s_g[k] * x_shifted[k] for k in range(3)]
    x_global = [sum(R_g[k][m] * x_scaled[m] for m in range(3)) for k in range(3)]

    x_prime = [0.0, 0.0, 0.0]
    Z = 0.0
    
    # Step 2: Iterate over warps
    for i in range(len(warp_params)):
        R_i, s_i, t_i = warp_params[i]
        
        # Compute affine transform A_i(x_global)
        x_i_shifted = [x_global[k] - t_i[k] for k in range(3)]
        x_i_scaled = [s_i[k] * x_i_shifted[k] for k in range(3)]
        x_i = [sum(R_i[k][m] * x_i_scaled[m] for m in range(3)) for k in range(3)]
        
        # Sample weight w_i at x_i
        w_i = sample_weight_volume(weight_volumes[i], x_i)
        
        # Accumulate
        x_prime = [x_prime[k] + w_i * x_i[k] for k in range(3)]
        Z += w_i
        
    # Step 3: Normalize
    if Z == 0:
        return x_global
        
    return [x_prime[k] / Z for k in range(3)]

def sample_weight_volume(volume, point):
    """
    Samples a 3D volume at a point, clamping to surface if outside.
    Implements standard trilinear interpolation and boundary clamping [G].
    """
    bounds = [len(volume), len(volume[0]), len(volume[0][0])]
    
    # Clamp to surface if outside bounds [G]
    if any(point[k] < 0 or point[k] >= bounds[k] for k in range(3)):
        clamped = [max(0, min(bounds[k] - 1, point[k])) for k in range(3)]
        return volume[int(clamped[0])][int(clamped[1])][int(clamped[2])]
    
    # Trilinear interpolation [G]
    lower = [int(point[k]) for k in range(3)]
    upper = [lower[k] + 1 for k in range(3)]
    frac = [point[k] - lower[k] for k in range(3)]
    
    def lerp(a, b, t): return a + t * (b - a)
    
    v00 = lerp(volume[lower[0]][lower[1]][lower[2]], volume[lower[0]][lower[1]][upper[2]], frac[2])
    v01 = lerp(volume[lower[0]][upper[1]][lower[2]], volume[lower[0]][upper[1]][upper[2]], frac[2])
    v10 = lerp(volume[upper[0]][lower[1]][lower[2]], volume[upper[0]][lower[1]][upper[2]], frac[2])
    v11 = lerp(volume[upper[0]][upper[1]][lower[2]], volume[upper[0]][upper[1]][upper[2]], frac[2])
    v0 = lerp(v00, v01, frac[1])
    v1 = lerp(v10, v11, frac[1])
    return lerp(v0, v1, frac[0])
```

## Subtopics

### Mixture of Affine Warps
This variant refers to the specific implementation where the warp field decoder uses a spatial mixture of affine warps to avoid overfitting and poor generalization associated with freely-varying template sample points [S1]. It involves computing transformation parameters using 2 fully-connected layers after encoding $z$ [S1]. Rotation is produced as a normalized quaternion vector transformed into a rotation matrix [S1]. Weights are computed by applying $\exp(\cdot)$ to the output of the fully-connected layers for the weighting volume $w_i$ to ensure non-negative weights [S1]. An additional global warp $\{R_g, s_g, t_g\}$ is learned by an MLP and applied to $\mathbf{x}$ before the warping field [S1].

## Operating characteristics
*   **Complexity:** The method requires $N$ affine transformations and $N$ volume samples per query [S1]. With $N=16$ and a $32^3$ grid, the per-ray cost is constant but higher than a single affine warp [S1].
*   **Performance:** The model outperforms no-warp, convolutional warp (transposed convolutions), convolutional warp (trilinear upsampling + convolution), and no-warp affine mixture models [S1]. It is particularly effective on complex dynamic data like hair swing, where convolutional warps fail [S1].
*   **Strengths:** Modeling the weighting volume in warp space is better than in world space [S1]. The low-resolution $32^3$ grid provides smoothness from trilinear interpolation, which helps learning [S1].
*   **Limitations:** Requires learning additional parameters for the global warp and the mixture weights [S1].

## Background
The method originates from "Neural Volumes: Learning Dynamic Renderable Volumes from Images" [S1]. It was developed to address overfitting and poor generalization to novel views in dynamic volume rendering [S1]. The approach extends linear blend skinning to volumetric space by using a spatial mixture of affine warps [S1].

## Sources
[S1] Neural Volumes: Learning Dynamic Renderable Volumes from Images, https://arxiv.org/abs/1906.07751, fetched 2026-10-26
