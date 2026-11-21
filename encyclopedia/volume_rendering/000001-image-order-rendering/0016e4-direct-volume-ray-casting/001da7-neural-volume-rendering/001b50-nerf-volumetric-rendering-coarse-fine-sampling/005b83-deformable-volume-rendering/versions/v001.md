# Deformable Volume Rendering

Also called: EndoNeRF Rendering

## Overview
Deformable Volume Rendering extends standard NeRF volumetric rendering to handle dynamic scenes by incorporating time-dependent deformations into the radiance field estimation [S1]. It solves the problem of rendering non-rigidly moving objects (such as biological tissues in surgical simulation) by conditioning the neural network's density and color outputs on both spatial coordinates and a temporal parameter, allowing for differentiable integration of color along rays that account for motion [S1]. It differs from static NeRF variants by explicitly modeling temporal deformation fields, and from rigid transformation methods (like RT-NeRF) by supporting continuous, non-rigid deformation rather than discrete bone-based transformations [S1, S18944-rigidly-transformed-neural-radiance-field-rt-nerf.md].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, time parameter $t$, and a neural network $\mathcal{N}$ parameterized by weights $\theta$. The network takes as input a 3D position $\mathbf{x}$ and a time embedding $t$ (and potentially other latent codes) to output density $\sigma$ and color $\mathbf{c}$.
*   **Outputs:** Rendered color $\hat{\mathbf{C}}$ for the ray at time $t$.

**Data Structures:**
*   **Neural Network $\mathcal{N}$:** A Multi-Layer Perceptron (MLP) or similar architecture. The input vector is typically $[\mathbf{x}, t]$. The output vector is $[\sigma, \mathbf{c}]$, where $\sigma \in \mathbb{R}$ is the volume density and $\mathbf{c} \in \mathbb{R}^3$ is the view-dependent or view-independent color. [G] The density $\sigma$ is typically unbounded, while color is clamped to $[0, 1]$.
*   **Sampling Points:** A set of $M$ points $\{\mathbf{x}_j\}_{j=1}^M$ sampled along the ray $\mathbf{r}(z) = \mathbf{o} + z\mathbf{d}$ within the scene bounds. Each point is associated with a time $t$.

**Numbered Steps:**
1.  **Ray Definition:** Define the ray $\mathbf{r}(z) = \mathbf{o} + z\mathbf{d}$ for $z \in [z_{near}, z_{far}]$.
2.  **Sampling:** Sample $M$ points $\mathbf{x}_j = \mathbf{r}(z_j)$ along the ray. The depths $z_j$ are determined by the coarse/fine sampling strategy inherited from the parent method [979-nerf-volumetric-rendering-coarse-fine-sampling.md].
3.  **Deformation/Positioning:** For each sample point $\mathbf{x}_j$, determine its effective position in the canonical or deformed space. In EndoNeRF, the input to the network is the spatial coordinate $\mathbf{x}_j$ and the time $t$. The network implicitly learns the deformation field mapping the current time $t$ to the canonical geometry [S1]. [G] If explicit deformation fields are used, $\mathbf{x}_j$ might be transformed by a learned displacement vector $\mathbf{d}(\mathbf{x}_j, t)$ before querying the network.
4.  **Network Query:** For each sample point $\mathbf{x}_j$ at time $t$, query the neural network:
    $$ [\sigma_j, \mathbf{c}_j] = \mathcal{N}(\mathbf{x}_j, t) $$
    where $\sigma_j$ is the density and $\mathbf{c}_j$ is the color.
5.  **Alpha Conversion:** Convert density $\sigma_j$ and step size $\delta_j = z_{j+1} - z_j$ to opacity $\alpha_j$:
    $$ \alpha_j = 1 - \exp(-\sigma_j \delta_j) $$
6.  **Weight Calculation:** Compute the weight $w_j$ for each sample, representing the probability that the ray terminates at sample $j$:
    $$ w_j = \alpha_j \exp \left( -\sum_{i=1}^{j-1} \sigma_i \delta_i \right) $$
    This is equivalent to the formula provided in [S1]:
    $$ w_j = \exp \bigg (-\sum _{i=1}^{j-1}\sigma _i \delta _i \bigg )\bigg (1-\exp \big (-\sigma _j \delta _j \big )\bigg ) $$
7.  **Color Integration:** Compute the final rendered color $\hat{\mathbf{C}}$ by summing the weighted colors:
    $$ \hat{\mathbf{C}}(\mathbf{r}(z), t) = \sum ^M_{j=1} w_j \mathbf{c}_j $$

**Parameters:**
*   $M$: Number of sampled points per ray. Typical values depend on the coarse/fine stages of the parent method. Not given in the sources for specific Deformable Volume Rendering defaults.
*   $t$: Time parameter. Normalized to $[0, 1]$ or mapped to the duration of the deformation sequence.

**Edge Cases and Pitfalls:**
*   **Temporal Consistency:** The network must ensure smooth transitions between time steps. Discontinuities in $t$ can cause flickering.
*   **Deformation Artifacts:** If the deformation field is not smooth, density discontinuities may arise, leading to visual artifacts.
*   **Boundary Conditions:** The ray bounds $[z_{near}, z_{far}]$ must encompass the deformed object at all times $t$.

## Subtopics

### [Rigidly Transformed Neural Radiance Field (RT-NeRF)](18944-rigidly-transformed-neural-radiance-field-rt-nerf.md)
RT-NeRF estimates a radiance field in an object's local coordinate system where density is constant with respect to local 3D location, conditioned on bone parameters. It differs from standard NeRF by explicitly incorporating rigid transformations and shape parameters as network inputs, allowing a single unified model to encode multiple parts or shape variations without part dependency. It is chosen when the scene consists of articulated rigid bodies (e.g., skeletal animation) rather than continuous non-rigid deformation. Link: [RT-NeRF](18944-rigidly-transformed-neural-radiance-field-rt-nerf.md).

### Two-Level MVC Interpolation for Volumetric Deformation
A deformation technique using Mean Value Coordinates (MVC) on a tetrahedral mesh to allow interactive, direct volumetric deformations of NeRFs [S2]. The method involves:
1.  Discretizing the selected volume into a tetrahedral mesh [S2].
2.  Storing MVC-interpolated values at the vertices of the tetrahedral mesh [S2].
3.  For any point query $x$ within the volume, performing barycentric interpolation from the pre-computed tetrahedral vertex values [S2].
Unlike Deforming-NeRF, which stores MVC coordinates on a regular lattice causing discretization artifacts, this method uses an adaptive tetrahedral lookup table for direct volumetric deformation [S2]. It is chosen for interactive editing scenarios requiring non-rigid deformation of specific volumetric regions. Link: [NeRFshop](1619-nerfacc-rendering-pipeline.md) (Note: NeRFshop is the parent project, but the specific technique is described in [S2]).

## Operating characteristics
*   **Time Complexity:** Depends on the number of samples $M$ and the cost of the neural network evaluation. Each ray requires $M$ forward passes through the network. The deformation/time conditioning adds minimal overhead to the network input processing.
*   **Memory Complexity:** Dominated by the neural network weights. No additional large data structures are required for the deformation field itself if it is learned implicitly by the network [S1]. If explicit deformation fields or tetrahedral meshes are used (as in MVC variants), memory scales with the complexity of the mesh or field resolution.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Can model complex, continuous non-rigid motions. Integrates seamlessly with standard NeRF rendering pipelines.
*   **Limitations:** Requires training on dynamic data. May struggle with large, discontinuous deformations if not regularized. Implicit deformation fields can be computationally expensive to evaluate if the network is large.
*   **When to use:** Dynamic scenes with smooth, continuous deformation (e.g., soft tissue, fluid-like motion).
*   **When not to use:** Scenes with rigid body dynamics better suited for RT-NeRF, or scenes with extreme discontinuities that break the smoothness assumption of the deformation field.

## Background
Deformable Volume Rendering, specifically in the context of EndoNeRF, originates from the need for realistic surgical simulation and reconstruction of endoscopic video data [S1]. The method builds on the NeRF framework [979-nerf-volumetric-rendering-coarse-fine-sampling.md] by introducing time as an input to the radiance field, allowing the model to capture temporal variations in the scene geometry and appearance. The specific formulation and application to endoscopic data are detailed in [S1].

## Sources
[S1] Efficient EndoNeRF reconstruction and its application for data-driven surgical simulation., https://doi.org/10.1007/s11548-024-03114-1, fetched 2026-10-26
[S2] NeRFshop, https://doi.org/10.1145/3585499, fetched 2026-10-26
