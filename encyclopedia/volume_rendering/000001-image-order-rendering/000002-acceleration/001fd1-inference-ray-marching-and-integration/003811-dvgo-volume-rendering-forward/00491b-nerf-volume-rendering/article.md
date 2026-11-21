# NeRF Volume Rendering

Also called: NeRF Rendering Equation, Neural Radiance Field rendering, Vanilla NeRF Volume Rendering, Volume Rendering from Voxel Grid, classical volume rendering with MLP

## Overview
NeRF Volume Rendering is a differentiable method for synthesizing photorealistic novel views of a scene by estimating volume density and RGB color along camera rays using a neural network [S4, unofficial]. It solves the problem of continuous 3D scene representation and rendering by parameterizing the scene as an implicit function, typically implemented via Multi-Layer Perceptrons (MLPs) [S3]. Unlike explicit voxel-based methods such as DVGO, which store features in a grid and use interpolation, the classical NeRF approach queries a continuous field directly at sampled points along the ray [S3]. This allows for high-fidelity view-dependent effects but incurs significant computational costs due to the large number of MLP queries required [S4, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   Camera pose: Origin $\mathbf{o} \in \mathbb{R}^3$ and view direction $\mathbf{d} \in \mathbb{R}^3$ (unit vector) [S3][S4, unofficial].
*   Neural Network Model $\mathcal{F}_\Phi$ (or $\mathcal{H}_\Theta$): An MLP that maps spatial coordinates and view directions to density and color [S2][S4, unofficial].
*   Sampling parameters: Number of sample points $K$ (or $N$) and the distribution of samples along the ray [S2][S3].

**Outputs:**
*   Pixel color $C(\mathbf{r}) \in \mathbb{R}^3$ for the ray $\mathbf{r}$ [S2][S3].

### Data Structures
*   **MLP Network $\mathcal{F}_\Phi$**: A feed-forward neural network [S3].
    *   **Input**: Concatenated spatial coordinate $\mathbf{x} = (x, y, z)$ and view direction $\mathbf{d}$ [S3].
    *   **Output**: Density $\sigma \in \mathbb{R}$ and color $\mathbf{c} \in \mathbb{R}^3$ [S3].
*   **Sample Points**: A list of $K$ 3D coordinates $\{\mathbf{x}_1, \dots, \mathbf{x}_K\}$ along the ray [S3].

### Numbered Steps

1.  **Ray Generation**: For a given pixel, generate a camera ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the camera origin and $\mathbf{d}$ is the viewing direction [S3][S4, unofficial].
2.  **Sampling**: Sample $K$ points $\mathbf{x}_i$ along the ray within a defined interval $[t_{near}, t_{far}]$ [S4, unofficial]. The points are typically spaced by a distance $\delta_i$ [S3].
3.  **MLP Query**: For each sampled point $\mathbf{x}_i$, query the MLP $\mathcal{F}_\Phi$ with input $(\mathbf{x}_i, \mathbf{d})$ to obtain the volume density $\sigma_i$ and the RGB color $\mathbf{c}_i$ [S2][S3][S4, unofficial].
4.  **Transmittance Calculation**: Compute the accumulated transmittance $T_i$ for each sample point $i$. $T_i$ represents the probability that the ray travels from the camera origin to sample $i$ without being absorbed or scattered [S2][S4, unofficial].
    $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$
    where $\delta_j$ is the distance between sample $j$ and $j+1$ [S3]. For the first sample, $T_1 = 1$ [S3].
5.  **Alpha Compositing**: Compute the weight $\alpha_i$ for each sample, representing the probability that the ray is absorbed at sample $i$:
    $$ \alpha_i = 1 - \exp(-\sigma_i \delta_i) $$
    [S3][S1].
6.  **Color Accumulation**: Compute the final pixel color $C(\mathbf{r})$ as a weighted sum of the colors of the sampled points:
    $$ C(\mathbf{r}) = \sum_{i=1}^{K} T_i \alpha_i \mathbf{c}_i = \sum_{i=1}^{K} T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i $$
    [S2][S3].
    *   *Continuous Formulation*: The process can also be viewed as a discretization of the continuous volume rendering integral [S4, unofficial]:
        $$ C(\mathbf{r}) = \int_{t_1}^{t_2} T(t) \sigma(\mathbf{r}(t)) \mathbf{c}(\mathbf{r}(t), \mathbf{d}) \, dt $$
        where $T(t) = \exp\left(-\int_{t_1}^{t} \sigma(\mathbf{r}(u)) \, du\right)$ [S5].

### Formulas and Definitions

*   **Ray**: $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ [S2][S3][S4, unofficial].
*   **Transmittance**: $T_i = \prod_{j=1}^{i-1} \exp(-\sigma_j \delta_j) = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ [S2][S3].
*   **Segment Transmittance**: $\omega_i = \exp(-\sigma_i \delta_i)$ [S2].
*   **Alpha (Absorption Probability)**: $\alpha_i = 1 - \omega_i = 1 - \exp(-\sigma_i \delta_i)$ [S2][S3].
*   **Final Color**: $C(\mathbf{r}) = \sum_{i=1}^{K} T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i$ [S2][S3].
*   **Loss Function**: The model is trained by minimizing the photometric loss (MSE) between rendered color $C(\mathbf{r})$ and ground truth color $C_{gt}(\mathbf{r})$ [S1][S4, unofficial]:
    $$ L_{NeRF} = \sum_{\mathbf{r} \in \mathcal{R}} \| C(\mathbf{r}) - C_{gt}(\mathbf{r}) \|^2 $$
    [S4, unofficial].

### Parameters
*   **$K$ (or $N$)**: Number of sample points per ray. A typical value is 192 [S2].
*   **$\delta_i$**: Distance between adjacent samples $x_i$ and $x_{i+1}$ [S3].
*   **$\mathbf{x}$**: Spatial coordinate $(x, y, z)$ [S2].
*   **$\mathbf{d}$**: View direction. Can be represented as a 3D Cartesian unit vector [S4, unofficial] or spherical coordinates $(\phi, \theta)$ [S2].
*   **$\sigma$**: Volume density (non-negative scalar) [S2][S3].
*   **$\mathbf{c}$**: RGB color (3D vector) [S2][S3].

### Edge Cases and Pitfalls
*   **Precision**: The exponential function in transmittance calculation can lead to underflow if density is high or distances are large, resulting in $T_i \approx 0$ [G].
*   **Boundaries**: The sampling interval $[t_{near}, t_{far}]$ must be chosen to encompass the scene. Points outside this range are ignored [S4, unofficial].
*   **Degenerate Input**: If $\sigma_i = 0$ for all samples, the color is black (or background color if explicitly handled) [G].
*   **Computational Cost**: The method requires a large number of MLP queries, leading to high training and rendering times [S4, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def nerf_volume_rendering(ray_origin, ray_direction, mlp_network, num_samples, near, far):
    """
    Renders a single pixel color using NeRF volume rendering.
    
    Args:
        ray_origin (tuple): (ox, oy, oz) camera origin.
        ray_direction (tuple): (dx, dy, dz) unit view direction.
        mlp_network (function): MLP function F_phi(x, d) -> (sigma, color).
        num_samples (int): Number of sample points K.
        near (float): Near clipping plane distance t_near.
        far (float): Far clipping plane distance t_far.
        
    Returns:
        tuple: (r, g, b) final pixel color.
    """
    
    # Step 1: Sample points along the ray
    t_vals = [near + i * (far - near) / num_samples for i in range(num_samples)]
    
    # Step 2: Initialize accumulators
    accumulated_transmittance = 1.0
    final_color = [0.0, 0.0, 0.0]
    
    for i in range(num_samples):
        # Calculate 3D point position
        t = t_vals[i]
        point = (
            ray_origin[0] + t * ray_direction[0],
            ray_origin[1] + t * ray_direction[1],
            ray_origin[2] + t * ray_direction[2]
        )
        
        # Step 3: Query MLP for density and color
        sigma, color = mlp_network(point, ray_direction)
        
        # Step 4: Compute delta (distance to next sample)
        if i < num_samples - 1:
            delta = t_vals[i+1] - t_vals[i]
        else:
            # For the last sample, delta is undefined by the formula.
            # Standard practice uses the previous interval or a small epsilon. [G]
            delta = t_vals[i] - t_vals[i-1] if i > 0 else 0.01
        
        # Step 5: Compute alpha and accumulate color
        alpha = 1.0 - math.exp(-sigma * delta)
        weight = accumulated_transmittance * alpha
        final_color[0] += weight * color[0]
        final_color[1] += weight * color[1]
        final_color[2] += weight * color[2]
        
        # Step 6: Update transmittance for next sample
        accumulated_transmittance *= math.exp(-sigma * delta)
        
        # Early termination optimization [G]
        if accumulated_transmittance < 1e-4:
            break
            
    return tuple(final_color)
```

### Subtopics

*   **DVGO Volume Rendering Forward**: Uses a voxel grid to store features and trilinear interpolation, reducing MLP queries [S2]. Choose when memory is constrained and high resolution is needed, but view-dependent effects are less critical or approximated [G]. [DVGO](8157-dvgo-volume-rendering-forward.md)
*   **Instant NGP**: Uses multi-resolution hash encoding to accelerate NeRF training and rendering [S4, unofficial]. Choose when fast training and real-time rendering are required [G]. [Instant NGP](8158-instant-ngp.md)
*   **TensoRF**: Decomposes the feature grid into low-rank factors for storage efficiency [S4, unofficial]. Choose when memory footprint is a primary concern [G]. [TensoRF](8159-tensorf.md)

## Operating characteristics

*   **Time Complexity**: Training and rendering times are typically very large due to the infinite and continuous nature of space requiring a multitude of giant MLP queries [S4, unofficial]. Each pixel requires $K$ MLP forward passes [G].
*   **Memory Complexity**: The MLP weights are stored in memory [G]. The size depends on the network architecture [G]. No explicit voxel grid is stored in the classical NeRF formulation [S3].
*   **Measured Performance**: With $K=192$ samples, rendering high-resolution images can be slow without acceleration techniques [S2].
*   **Strengths**: High-quality view-dependent effects due to direct MLP query of color based on view direction [S3]. Continuous representation allows for arbitrary resolution [S3].
*   **Limitations**: Slow rendering speed. High memory usage for large networks. Requires significant computational resources for training [S4, unofficial].
*   **When to use**: When photorealism and view-dependent effects are paramount, and computational resources are available [G].
*   **When not to use**: When real-time rendering is required, or memory is severely constrained [G].

## Background

NeRF (Neural Radiance Fields) was introduced by Mildenhall et al. in 2020 [S3]. It uses MLP networks to learn the radiance field of arbitrary scenes [S3]. Once sufficiently optimized, NeRF can estimate both the volume density $\sigma$ and RGB color $\mathbf{c}$ of the continuous function that describes the radiance field [S3]. It is then able to reconstruct a photorealistic view from an unseen pose using a differentiable volume rendering method [S4, unofficial]. The method has become a foundational technique in 3D computer vision and graphics, inspiring numerous acceleration methods and applications in novel view synthesis, geometry reconstruction, and stylization [S2][S4, unofficial].

## Sources

[S1] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
[S2] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
[S3] NeRF-Art: Text-Driven Neural Radiance Fields Stylization, https://doi.org/10.1109/tvcg.2023.3283400, fetched 2026-10-26
[S4] MF-NeRF: Memory Efficient NeRF with Mixed-Feature Hash Table, https://core.ac.uk/download/564091854.pdf, fetched 2026-10-27 (unofficial)
[S5] High-fidelity 3D Reconstruction of Plants using Neural Radiance Field, https://arxiv.org/abs/2311.04154v1, fetched 2026-10-27
