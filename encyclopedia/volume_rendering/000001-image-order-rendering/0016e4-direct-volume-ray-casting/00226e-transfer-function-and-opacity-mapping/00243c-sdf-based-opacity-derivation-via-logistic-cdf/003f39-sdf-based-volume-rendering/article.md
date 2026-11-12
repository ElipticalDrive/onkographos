# SDF-based Volume Rendering

Also called: NeuS Opacity Derivation, NeuS Volume Rendering with SDF, SDF-to-Opacity via Sigmoid

## Overview
SDF-based Volume Rendering is a neural rendering technique that represents a scene as an implicit Signed Distance Field (SDF) parameterized by a neural network, converting SDF values into opacity for volume integration [S1, unofficial][S2]. It solves the problem of reconstructing high-quality surfaces with fine details while rendering photo-realistic images, distinguishing itself from standard NeRF by using a logistic mapping of the SDF to ensure the opacity weight function peaks at the surface intersection [S1, unofficial][S2]. This method builds on [SDF-based Opacity Derivation via Logistic CDF](3233-sdf-based-opacity-derivation-via-logistic-cdf.md) to correctly model surface opacity, avoiding the bias of naive SDF-to-density mapping.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera center $\mathbf{o}$, viewing direction $\mathbf{v}$, a neural network $f(\mathbf{x})$ predicting the SDF value at 3D point $\mathbf{x}$, a steepness parameter $s$, and $N$ discrete sample points $\mathbf{p}_i$ along the ray.
*   **Outputs:** Rendered color $\hat{C}(\mathbf{r})$ for the pixel.

**Data Structures:**
*   **Ray:** $\mathbf{p}(t) = \mathbf{o} + t\mathbf{v}$ for $t \ge 0$.
*   **Samples:** $N$ points $\mathbf{p}_i = \mathbf{o} + t_i\mathbf{v}$ where $t_1 < t_2 < \dots < t_N$.
*   **SDF Network:** $f: \mathbb{R}^3 \to \mathbb{R}$, typically an MLP.

**Formulas:**
1.  **Sigmoid Function:**
    $$ \Phi_s(x) = \frac{1}{1 + e^{-sx}} $$
    where $s$ is a learned or manually updated steepness parameter [S1, unofficial][S2].

2.  **Opacity:**
    For each sample interval $[t_i, t_{i+1}]$, the opacity $\alpha_i$ is computed as:
    $$ \alpha_i = \max \left( \frac{\Phi_s(f(\mathbf{p}_i)) - \Phi_s(f(\mathbf{p}_{i+1}))}{\Phi_s(f(\mathbf{p}_i))}, 0 \right) $$
    where $f(\mathbf{p}_i)$ is the SDF value at point $\mathbf{p}_i$ [S1, unofficial][S2].

3.  **Transmittance:**
    $$ T_i = \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    with $T_1 = 1$ [S1, unofficial].

4.  **Rendered Color:**
    $$ \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i c_i $$
    where $c_i$ is the predicted color at $\mathbf{p}_i$ [S1, unofficial].

**Parameters:**
*   **$s$ (steepness):** Controls the sharpness of the sigmoid. Typical values are not explicitly given in the sources, but it is described as being learned or manually updated during training [S1, unofficial].
*   **$N$ (samples):** Number of discrete points along the ray. Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdf_volume_rendering(ray_origin, ray_direction, sdf_network, color_network, s, num_samples):
    """
    Renders a pixel using SDF-based volume rendering.
    
    Args:
        ray_origin: 3D vector, camera center o.
        ray_direction: 3D vector, viewing direction v.
        sdf_network: Function f(x) returning SDF value.
        color_network: Function c(x) returning color.
        s: Float, steepness parameter for sigmoid.
        num_samples: Integer, N, number of sample points.
        
    Returns:
        rendered_color: 3D vector, C_hat(r).
    """
    # Generate sample points along the ray
    # Note: The specific sampling strategy (e.g., uniform, stratified, hierarchical) 
    # is not defined in the provided sources [G: Assume standard uniform or hierarchical sampling 
    # with t_i sorted such that t_1 < t_2 < ... < t_N].
    t_values = generate_sample_points(ray_origin, ray_direction, num_samples)
    
    points = []
    for t in t_values:
        points.append(ray_origin + t * ray_direction)
        
    # Compute SDF values
    sdf_values = []
    for p in points:
        sdf_values.append(sdf_network(p))
        
    # Compute Sigmoid values
    # Phi_s(x) = 1 / (1 + exp(-s * x))
    sigmoid_values = []
    for f_val in sdf_values:
        sigmoid_values.append(1.0 / (1.0 + math.exp(-s * f_val)))
        
    accumulated_transmittance = 1.0
    rendered_color = [0.0, 0.0, 0.0]
    
    for i in range(num_samples - 1):
        # Opacity alpha_i for interval [t_i, t_{i+1}]
        # alpha_i = max((Phi_s(f(p_i)) - Phi_s(f(p_{i+1}))) / Phi_s(f(p_i)), 0)
        phi_i = sigmoid_values[i]
        phi_next = sigmoid_values[i+1]
        
        # Guard against division by zero if phi_i is 0
        if phi_i == 0:
            alpha = 0.0
        else:
            alpha = max((phi_i - phi_next) / phi_i, 0.0)
            
        # Get color at current point p_i
        color_i = color_network(points[i])
        
        # Accumulate color
        rendered_color += accumulated_transmittance * alpha * color_i
        
        # Update transmittance
        accumulated_transmittance *= (1.0 - alpha)
        
        # Early termination if transmittance is negligible (optional optimization)
        if accumulated_transmittance < 1e-5:
            break
            
    return rendered_color
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $\Phi_s(f(\mathbf{p}_i)) = 0$, the opacity formula involves division by zero. The pseudocode includes a guard for this [G].
*   **Negative Opacity:** The $\max(..., 0)$ ensures opacity is non-negative, handling cases where the sigmoid value increases along the ray (which should not happen for a consistent SDF but may occur due to network approximation) [S1, unofficial][S2].
*   **Parameter $s$:** The value of $s$ significantly affects the result. It must be learned or updated during training to adapt to the scene's scale and detail [S1, unofficial].

## Subtopics

### [MPF Volume Rendering](10260-mpf-volume-rendering.md)
MPF Volume Rendering replaces the SDF backbone with Metric-Phase Fields (MPFs) to handle sign-ambiguous geometry and thin structures. It derives opacity from a signed composite field $\phi$ using a NeuS-style logistic mapping, while color is predicted by a separate radiance network, isolating the geometry representation change from the rendering pipeline.

### Sigmoidal OTF
A transfer function that maps normalized feature detector values to opacity using a threshold $T$ and steepness parameter $\gamma$, designed to produce smoothly highlighted edges in volume renderings [S3]. It differs from threshold-based OTF by replacing hard cutoffs with smooth transitions, making it more effective for highlighting edges without abrupt opacity jumps [S3]. It is applied to color and gradient feature detectors (gR, cBRAIN, cR, cBR) for cryo-imaging volume rendering of adult and embryonic mouse datasets [S3]. Parameters: $\gamma$ (typical value 50) controls the steepness, approaching a step function for large $\gamma$; $T$ is the threshold value, enabling uniform definition across datasets [S3].

## Operating characteristics
*   **Time Complexity:** Linear in the number of samples $N$ per ray, $O(N)$, due to the sequential accumulation of transmittance and color [S1, unofficial].
*   **Memory Complexity:** Depends on the neural network size for $f(\mathbf{x})$ and $c(\mathbf{x})$. Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Enables high-fidelity surface reconstruction and photo-realistic rendering by leveraging the geometric priors of SDFs [S1, unofficial][S2]. The logistic mapping ensures opacity peaks at the surface, improving surface quality compared to naive density mappings [S2].
*   **Limitations:** Requires learning or tuning the steepness parameter $s$ [S1, unofficial]. Performance depends on the quality of the SDF network approximation [S2].
*   **When to Use:** When high-quality surface reconstruction is required alongside novel view synthesis, particularly for scenes with fine details [S1, unofficial][S2].
*   **When Not to Use:** When the scene has sign-ambiguous geometry or thin structures, where MPF-based methods might be more suitable [S1, unofficial].

## Background
SDF-based Volume Rendering was introduced by Wang et al. in the NeuS paper (2021) [S1, unofficial][S2]. It was developed to address the limitations of NeRF in surface reconstruction, providing a method to render photo-realistic images while simultaneously reconstructing high-quality surfaces with fine details [S1, unofficial]. The method uses a logistic function to convert SDF predictions to opacity, allowing optimization with neural volume rendering [S2]. It has been adopted in subsequent works such as Neuralangelo for high-fidelity neural surface reconstruction [S2].

## Sources
[S1] Voxurf: Voxel-based Efficient and Accurate Neural Surface Reconstruction, https://doi.org/10.48550/arxiv.2208.12697, fetched 2026-10-26 (unofficial)
[S2] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
[S3] Enhanced volume rendering techniques for high-resolution color cryo-imaging data, https://doi.org/10.1117/12.813756, fetched 2026-11-08
