# Deformable Volume Rendering

Also called: EndoNeRF Rendering

## Overview
Deformable Volume Rendering is a volumetric rendering method that computes color along rays at a specified time $t$ by sampling a neural network that outputs density and color, then performing differentiable integration [S1]. It differs from static NeRF variants by conditioning the radiance field on a temporal parameter $t$ [G].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, time parameter $t$, and a neural network $\mathcal{N}$ parameterized by weights $\theta$ [G].
*   **Outputs:** Rendered color $\hat{\mathbf{C}}$ for the ray at time $t$ [S1].

**Data Structures:**
*   **Neural Network $\mathcal{N}$:** A multi-layer perceptron that takes a 3D position $\mathbf{x}$ and time $t$ to output density $\sigma$ and color $\mathbf{c}$ [G].
*   **Sampling Points:** A set of $M$ points $\{\mathbf{x}_j\}_{j=1}^M$ sampled along the ray $\mathbf{r}(z) = \mathbf{o} + z\mathbf{d}$ within scene bounds, each associated with time $t$ [G].

**Numbered Steps:**
1.  **Ray Definition:** Define the ray $\mathbf{r}(z) = \mathbf{o} + z\mathbf{d}$ for $z \in [z_{near}, z_{far}]$ [G].
2.  **Sampling:** Sample $M$ points $\mathbf{x}_j = \mathbf{r}(z_j)$ along the ray. The depths $z_j$ are determined by the coarse/fine sampling strategy inherited from the parent method [979-nerf-volumetric-rendering-coarse-fine-sampling.md] [G].
3.  **Network Query:** For each sample point $\mathbf{x}_j$ at time $t$, query the neural network: $[\sigma_j, \mathbf{c}_j] = \mathcal{N}(\mathbf{x}_j, t)$ [G].
4.  **Step Lengths:** Compute the distance between adjacent samples: $\delta_j = z_{j+1} - z_j$ [G].
5.  **Weight Calculation:** Compute the weight $w_j$ for each sample using the formula from [S1]:
    $$w_j = \exp \bigg (-\sum _{i=1}^{j-1}\sigma _i \delta _i \bigg )\bigg (1-\exp \big (-\sigma _j \delta _j \big )\bigg )$$
6.  **Color Integration:** Compute the final rendered color by summing the weighted colors:
    $$\hat{\textbf{C}}(\textbf{r}(z), t) = \sum ^M_{j=1} w_j \textbf{c}_j$$ [S1].

**Parameters:**
*   $M$: Number of sampled points per ray. Typical values depend on the coarse/fine stages of the parent method [G].
*   $t$: Time parameter. Typically normalized to $[0, 1]$ or mapped to the duration of the deformation sequence [G].

**Edge Cases and Pitfalls:**
*   **Temporal Consistency:** The network must ensure smooth transitions between time steps to avoid flickering [G].
*   **Deformation Artifacts:** If the underlying deformation field is not smooth, density discontinuities may arise, leading to visual artifacts [G].
*   **Boundary Conditions:** The ray bounds $[z_{near}, z_{far}]$ must encompass the deformed object at all times $t$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_deformable_volume(ray_o, ray_d, t, network, M, z_near, z_far):
    # 1. Ray definition and sampling
    z = torch.linspace(z_near, z_far, M)
    x = ray_o + z.unsqueeze(1) * ray_d.unsqueeze(0) # Shape: (M, 3)
    
    # 2. Network query
    sigma, c = network(x, t) # Returns density and color for each sample
    
    # 3. Step lengths
    delta = torch.diff(z, prepend=z[:1]) # Shape: (M,)
    
    # 4. Weight calculation
    # Accumulated transmittance: exp(-sum_{i=1}^{j-1} sigma_i * delta_i)
    accumulated = torch.cumsum(sigma[:-1] * delta[:-1], dim=0)
    accumulated = torch.cat([torch.zeros_like(accumulated[:1]), accumulated], dim=0)
    transmittance = torch.exp(-accumulated)
    
    # Opacity for current step: 1 - exp(-sigma_j * delta_j)
    opacity = 1.0 - torch.exp(-sigma * delta)
    
    # Weights: transmittance * opacity
    w = transmittance * opacity
    
    # 5. Color integration
    C = torch.sum(w.unsqueeze(1) * c, dim=0)
    return C
```

## Subtopics

### [Rigidly Transformed Neural Radiance Field (RT-NeRF)](18944-rigidly-transformed-neural-radiance-field-rt-nerf.md)
RT-NeRF estimates a radiance field in an object's local coordinate system where density is constant with respect to local 3D location, conditioned on bone parameters. It differs from standard NeRF by explicitly incorporating rigid transformations and shape parameters as network inputs, allowing a single unified model to encode multiple parts or shape variations without part dependency. It is chosen when the scene consists of articulated rigid bodies rather than continuous non-rigid deformation. Link: [RT-NeRF](18944-rigidly-transformed-neural-radiance-field-rt-nerf.md).

### [VoISDF Sampling Algorithm](23567-voisdf-sampling-algorithm.md)
The VoISDF Sampling Algorithm generates a set of sample points along a ray for volume rendering by iteratively refining an initial uniform grid based on a rigorous upper bound of the opacity approximation error. It solves the problem of sub-optimal sampling in neural implicit surface rendering, where naive approximations of the opacity function lead to missed or over-extended density regions, causing incorrect radiance approximations and artifacts. Unlike sibling methods like [Rigidly Transformed Neural Radiance Field (RT-NeRF)](18944-rigidly-transformed-neural-radiance-field-rt-nerf.md) which may rely on random or fixed uniform sampling, this method guarantees that the discretization error of the opacity CDF remains below a threshold $\epsilon$, enabling accurate inverse transform sampling for the final render. Link: [VoISDF Sampling Algorithm](23567-voisdf-sampling-algorithm.md).

### Two-Level MVC Interpolation for Volumetric Deformation
A deformation technique using Mean Value Coordinates (MVC) on a tetrahedral mesh to allow interactive, direct volumetric deformations of NeRFs [S2]. The method involves:
1.  Discretizing the selected volume into a tetrahedral mesh [S2].
2.  Storing MVC-interpolated values at the vertices of the tetrahedral mesh [S2].
3.  For any point query $x$ within the volume, performing barycentric interpolation from the pre-computed tetrahedral vertex values [S2].
Unlike Deforming-NeRF, which stores MVC coordinates on a regular lattice causing discretization artifacts, this method uses an adaptive tetrahedral lookup table for direct volumetric deformation [S2]. It is chosen for interactive editing scenarios requiring non-rigid deformation of specific volumetric regions. Link: [NeRFshop](1619-nerfacc-rendering-pipeline.md).

## Operating characteristics
*   **Time Complexity:** Depends on the number of samples $M$ and the cost of the neural network evaluation. Each ray requires $M$ forward passes through the network [G].
*   **Memory Complexity:** Dominated by the neural network weights [G].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Can model complex, continuous non-rigid motions and integrates seamlessly with standard NeRF rendering pipelines [G].
*   **Limitations:** Requires training on dynamic data and may struggle with large, discontinuous deformations if not regularized [G].
*   **When to use:** Dynamic scenes with smooth, continuous deformation [G].
*   **When not to use:** Scenes with rigid body dynamics better suited for RT-NeRF, or scenes with extreme discontinuities that break the smoothness assumption of the deformation field [G].

## Background
Deformable Volume Rendering originates from the EndoNeRF method [S1]. The method builds on the NeRF framework [979-nerf-volumetric-rendering-coarse-fine-sampling.md] by introducing time as an input to the radiance field, allowing the model to capture temporal variations in the scene geometry and appearance [G].

## Sources
[S1] Efficient EndoNeRF reconstruction and its application for data-driven surgical simulation., https://doi.org/10.1007/s11548-024-03114-1, fetched 2026-10-26
[S2] NeRFshop, https://doi.org/10.1145/3585499, fetched 2026-10-26
