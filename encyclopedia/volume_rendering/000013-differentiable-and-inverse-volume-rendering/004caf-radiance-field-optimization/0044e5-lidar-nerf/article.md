# LiDAR-NeRF

Also called: LiDAR Neural Radiance Fields, LiDAR-NeRF Framework, LiDAR-NeRF with iNGP, LiDAR-NeRF with standard NeRF, Neural Radiance Fields for LiDAR

## Overview
LiDAR-NeRF is a differentiable end-to-end framework for novel LiDAR view synthesis that leverages a neural radiance field to jointly learn geometry and 3D point attributes (distance, intensity, ray-drop probability) without explicit 3D reconstruction [S1]. It reformulates NeRF to model LiDAR sensor characteristics, synthesizing realistic geometry (distance), intensity, and ray-drop probability [S1]. Unlike traditional NeRFs which focus on individual pixels and ignore local information leading to poor geometry in low-texture areas, LiDAR-NeRF incorporates structural regularization to preserve local details [S1]. It outperforms explicit 3D reconstruction and ray-tracing baselines because explicit methods neglect crucial LiDAR features like beam penetration through glass and loss due to surface material/normal direction [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of LiDAR point clouds from multiple viewpoints. These are converted into 360-degree range pseudo images (range images) where each pseudo-pixel represents the distance between the LiDAR receiver and a world point hit by a laser beam [S1].
*   **Output:** A synthesized novel LiDAR view consisting of a range image (distance), an intensity map, and a ray-drop probability map [S1].

**Data Structures:**
The radiance field is represented by a function $f(x, \theta)$ that outputs three values:
1.  Density $\sigma$
2.  Expected distance $d$
3.  A two-channel feature map containing intensity $i$ and ray-drop probability $p$ [S1].

**Preprocessing:**
1.  **Centering:** Center the LiDAR point clouds by subtracting the origin of the global world coordinate system from the scene’s central frame [S1].
2.  **Scaling:** Scale the scene frames by a factor such that the region of interest falls within a unit cube, which is required by most positional encodings used in NeRFs [S1].
3.  **Ray Direction Calculation:** For each pseudo-pixel $(h, w)$ in the $H \times W$ range pseudo image, calculate the laser viewing direction $\theta$ using angular coordinates $(\alpha, \beta)$:
    $$
    \begin{pmatrix} \alpha \\ \beta \end{pmatrix} = \begin{pmatrix} |f_{up}| - h f_v H^{-1} \\ -(2w - W) \pi W^{-1} \end{pmatrix}
    $$
    where $f_v = |f_{down}| + |f_{up}|$ is the vertical field-of-view [S1].
4.  **Coordinate Transformation:** Transform ray origin $o$ and viewing direction $\theta$ to the global world coordinate system so sampled points $t$ are consistent across multiple LiDAR frames [S1].

**Rendering Process:**
For a ray $r$, sample $N$ points $t_i$ along the ray [S1]. Calculate the distance between adjacent samples $\delta_i = t_{i+1} - t_i$ [S1]. Compute accumulated transmittance $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ [S1].

The rendered expected depth $\hat{D}(r)$, intensity $I(r)$, and ray-drop probability $\hat{P}(r)$ are computed as:
$$
\hat{D}(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) t_i
$$
$$
I(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) i_i
$$
$$
\hat{P}(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) p_i
$$
where $i_i$ is the intensity feature and $p_i$ is the ray-drop probability feature at sample $i$ [S1].

**Loss Function:**
The total loss is defined as:
$$
L_{total} = L_{distance} + \lambda_1 L_{intensity}(r) + \lambda_2 L_{raydrop}(r) + \lambda_3 L_{reg}
$$
where:
*   $L_{distance}(r) = \sum_{r \in R} || \hat{D}(r) - D(r) ||_1$ [S1]
*   $L_{intensity}(r) = \sum_{r \in R} || I(r) - I_{gt}(r) ||_2^2$ [S1]
*   $L_{raydrop}(r) = \sum_{r \in R} || \hat{P}(r) - P_{gt}(r) ||_2^2$ [S1]
*   $R$ is the set of training rays [S1].

**Structural Regularization:**
Standard geometry regularization (smoothness-loss, TV-loss) is ineffective in large-scale scenes where differences between neighboring points are significant [S1]. Standard gradient loss is dominated by rich-texture areas [S1]. LiDAR-NeRF proposes a structural regularization strategy that restricts gradient loss to low-texture areas (e.g., ground) to guide NeRF geometry [S1].
$$
L_{reg} = || \hat{G}_M(R) - G_M(R) ||_1
$$
where $G_M(\cdot)$ denotes the gradient operation with a low-texture areas mask [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_accumulated_transmittance(sigma, deltas):
    """T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)"""
    T = [1.0]
    for i in range(1, len(sigma)):
        T.append(T[-1] * math.exp(-sigma[i-1] * deltas[i-1]))
    return T

def query_radiance_field(point, direction):
    """Returns (sigma, expected_distance, intensity, ray_drop_probability) from the trained MLPs."""
    # Implementation depends on the specific network architecture (e.g., hash grid + MLP or standard MLP)
    # Returns the four predicted values for the given 3D point and viewing direction.
    pass

def sample_points_along_ray(origin, direction, num_samples):
    """Samples points t_i along the ray and returns positions and inter-sample distances."""
    # Assumes uniform sampling along the ray direction for simplicity
    t_vals = [i * (1.0 / num_samples) for i in range(1, num_samples + 1)]
    samples = [(origin[0] + t * direction[0], origin[1] + t * direction[1], origin[2] + t * direction[2]) for t in t_vals]
    deltas = [t_vals[i] - t_vals[i-1] for i in range(1, len(t_vals))]
    deltas.insert(0, t_vals[0])
    return samples, deltas, t_vals

def render_lidar_view(ray_origins, ray_directions, coarse_samples, fine_samples):
    """Renders depth, intensity, and ray-drop probability for a batch of rays."""
    rendered_depths, rendered_intensities, rendered_ray_drops = [], [], []

    for origin, direction in zip(ray_origins, ray_directions):
        # Coarse pass
        samples, deltas, t_vals = sample_points_along_ray(origin, direction, coarse_samples)
        sigma, d, intensity, ray_drop = [], [], [], []
        for x in samples:
            s, dist, i_val, p_val = query_radiance_field(x, direction)
            sigma.append(s); d.append(dist); intensity.append(i_val); ray_drop.append(p_val)

        T = compute_accumulated_transmittance(sigma, deltas)
        weights = [T[i] * (1.0 - math.exp(-sigma[i] * deltas[i])) for i in range(len(sigma))]

        depth = sum(w * t_vals[i] for w, i in zip(weights, range(len(t_vals))))
        intensity_out = sum(w * intensity[i] for w, i in zip(weights, range(len(intensity))))
        ray_drop_out = sum(w * ray_drop[i] for w, i in zip(weights, range(len(ray_drop))))

        rendered_depths.append(depth)
        rendered_intensities.append(intensity_out)
        rendered_ray_drops.append(ray_drop_out)

    return rendered_depths, rendered_intensities, rendered_ray_drops

def compute_structural_regularization_loss(rendered_depth, ground_truth_depth, low_texture_mask):
    """L_reg = || G_M(R) - G_M(R_gt) ||_1"""
    # Apply gradient mask to restrict regularization to low-texture areas
    grad_rendered = [abs(rendered_depth[i] - rendered_depth[i-1]) if mask else 0.0 for i, mask in enumerate(low_texture_mask)]
    gt_grad = [abs(ground_truth_depth[i] - ground_truth_depth[i-1]) if mask else 0.0 for i, mask in enumerate(low_texture_mask)]
    return sum(abs(a - b) for a, b in zip(grad_rendered, gt_grad)) / len(low_texture_mask)
```

## Subtopics

### [LidaRF Network Architecture](12866-lidarf-network-architecture.md)
A specialized neural network design for LiDAR-based NeRF that jointly learns geometry and 3D point attributes (distance, intensity, ray-drop probability) without explicit 3D reconstruction. It distinguishes itself from standard NeRF variants by integrating a 3D sparse UNet for LiDAR geometry encoding with a multi-resolution hash grid for density prediction, enabling efficient processing of sparse LiDAR point clouds. Choose this architecture when efficient processing of sparse LiDAR data is required for differentiable LiDAR view synthesis. Builds on [LiDAR-NeRF](11289-lidar-nerf.md) by providing the specific network components required for differentiable LiDAR view synthesis.

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
*   **LiDAR-NeRF (w iNGP):** Implemented based on torch-ngp [S1]. Optimized per scene using a single NVIDIA GeForce RTX 3090 GPU [S1]. Total of 30k steps [S1]. Coarse network samples 768 samples per ray; fine network samples 64 samples per ray [S1]. Hash encoding finest resolution: 32768 [S1]. Patch-wise training with patch size 2x8 and mask gradients smaller than threshold 0.1 [S1].
*   **LiDAR-NeRF (w NeRF):** Implemented based on nerf-pytorch [S1]. Optimized for 400k iterations with batch size 2048 [S1]. Coarse network samples 64 times; fine network samples 128 times [S1]. Highest frequency of coordinates set to $2^{15}$ [S1].

**Strengths and Limitations:**
*   **Strengths:** Enables end-to-end optimization without explicit 3D reconstruction or game engine usage [S1]. Produces realistic LiDAR patterns by accounting for beam penetration through glass and loss due to surface material/normal direction [S1]. Outperforms explicit 3D reconstruction and ray-tracing baselines (LiDARsim, PCGen) [S1].
*   **Limitations:** Standard geometry regularization techniques (smoothness-loss, TV-loss) are ineffective in large-scale scenes [S1]. Standard gradient loss is dominated by rich-texture areas, making it insufficient for guiding geometry in low-texture regions without structural regularization [S1].

**When to use:**
Use for novel LiDAR view synthesis where realistic geometry, intensity, and ray-drop probability are required, particularly where explicit reconstruction fails to capture LiDAR-specific artifacts [S1].

## Background
LiDAR-NeRF was introduced by Tang Tao, Longfei Gao, Guangrun Wang, Yixing Lao, Peng Chen, Hengshuang Zhao, Dayang Hao, Xiaodan Liang, Mathieu Salzmann, and Kaicheng Yu in "LiDAR-NeRF: Novel LiDAR View Synthesis via Neural Radiance Fields" (arXiv:2304.10406v2, July 14, 2023) [S1]. The work addresses the challenge that traditional model-based LiDAR simulators suffer from limited scalability and fail to produce realistic LiDAR patterns, while traditional NeRFs are not directly applicable to point clouds [S1]. The authors established an object-centric synthesis approach for traffic objects to validate the framework [S1].

## Sources
[S1] LiDAR-NeRF: Novel LiDAR View Synthesis via Neural Radiance Fields, https://arxiv.org/abs/2304.10406v2, fetched 2026-10-26
