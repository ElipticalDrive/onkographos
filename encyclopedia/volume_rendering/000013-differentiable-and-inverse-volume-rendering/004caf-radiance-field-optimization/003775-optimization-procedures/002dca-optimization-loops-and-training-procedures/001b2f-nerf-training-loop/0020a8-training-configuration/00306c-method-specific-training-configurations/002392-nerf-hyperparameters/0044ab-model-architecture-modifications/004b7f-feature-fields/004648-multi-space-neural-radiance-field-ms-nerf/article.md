# Multi-Space Neural Radiance Field (MS-NeRF)
Also called: MS-Mip-NeRF 360, MS-NeRFB, Multi-space NeRF

## Overview
MS-NeRF addresses the failure of standard single-space Neural Radiance Fields (NeRF) to render reflective and refractive objects, which cause multi-view consistency violations and blurry artifacts [S1]. It replaces the single radiance field with a group of parallel feature fields in distinct sub-spaces, allowing virtual images to be stored in sub-spaces visible only from specific viewpoints [S1]. Unlike single-space methods that often require manual labeling of reflective surfaces to avoid sub-optimal convergence, MS-NeRF automatically handles these effects without additional loss terms [S1].

## How it works
MS-NeRF operates as an enhancement module that replaces the output layer of a backbone NeRF method (e.g., MLP-based NeRF, Mip-NeRF 360, or grid-based TensoRF) with a multi-space module [S1].

**Inputs and Outputs**
*   **Input:** A ray $r$ defined by origin and direction, sampled at $N$ points along the ray. The backbone network provides latent features and densities for each point.
*   **Output:** A final rendered color $\hat{C}(r)$ for the ray.

**Data Structures**
The method introduces $K$ parallel sub-spaces. For each sampled point $i$ along the ray and each sub-space $k \in \{1, \dots, K\}$:
*   $\sigma_{ik}$: Volume density for sub-space $k$ at point $i$.
*   $f_{ik}$: Feature vector of dimension $d$ for sub-space $k$ at point $i$.
*   $T_i$: Transmittance up to point $i$, calculated as $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_{jk} \delta_j)$ (Note: The source uses a specific accumulation formula below).

**Parameters**
*   $K$: Total number of sub-spaces.
*   $d$: Feature dimension of the neural feature field.
*   Typical values: On Mip-NeRF 360, $K$ and $d$ are chosen to add only 0.5% extra parameters [S1]. On TensoRF, they add 0.046% extra parameters [S1].

**Algorithm**
1.  **Ray Casting and Feature Collection:** For each sub-space $k$, accumulate features along the ray. The feature map $F_k$ for sub-space $k$ is calculated as [S1]:
    $$
    \hat{F}_k(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_{ik} \delta_i)) f_{ik}
    $$
    where $\delta_i$ is the distance between sample points. $T_i$ represents the transmittance. The source notation implies $T_i$ is computed similarly to standard volumetric rendering but applied to the specific density $\sigma_{ik}$ of the sub-space.

2.  **Multi-Head Processing:** The collected feature maps $\{F_k\}_{k=1}^K$ are processed by two separate Multi-Layer Perceptrons (MLPs), each with one hidden layer [S1]:
    *   **Decoder MLP ($\Theta_D$):** Takes $\{F_k\}$ as input and outputs RGB color vectors $\{C_k\}$.
        $$
        \Theta_D(\{F_k\}) \rightarrow \{C_k\}
        $$
    *   **Gate MLP ($\Theta_G$):** Takes $\{F_k\}$ as input and outputs scalar weights $\{w_k\}$ controlling the visibility/contribution of each sub-space.
        $$
        \Theta_G(\{F_k\}) \rightarrow \{w_k\}
        $$

3.  **Final Rendering:** The final color $\hat{C}(r)$ is computed by applying a softmax function to the gate weights $\{w_k\}$ to form a convex combination of the sub-space colors $\{C_k\}$ [S1]:
    $$
    \hat{C}(r) = \sum_{k=1}^{K} \left( \frac{\exp(w_k)}{\sum_{i=1}^{K} \exp(w_i)} \right) C_k
    $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ms_nerf_render(ray, backbone_network, K, d):
    """
    Renders a ray using the Multi-Space NeRF module.
    
    Args:
        ray: Ray origin and direction.
        backbone_network: The underlying NeRF backbone (e.g., Mip-NeRF 360, TensoRF).
        K: Number of sub-spaces.
        d: Feature dimension.
        
    Returns:
        final_color: Rendered RGB color.
    """
    # 1. Sample points along the ray
    # The backbone network typically handles sampling. 
    # We assume backbone_network returns densities and features for all K sub-spaces.
    # Let's assume the backbone is modified or wrapped to output K sets of (sigma, feature).
    
    # Sample points i = 1 to N
    # For each point i and sub-space k:
    #   sigma_ik = backbone_network.get_density(ray, point_i, sub_space=k)
    #   f_ik = backbone_network.get_feature(ray, point_i, sub_space=k, dim=d)
    
    # 2. Accumulate feature maps for each sub-space
    feature_maps = []
    for k in range(K):
        accumulated_feature = 0.0
        transmittance = 1.0
        for i in range(N):
            # Calculate opacity for this segment in sub-space k
            sigma_ik = backbone_network.get_density(ray, points[i], sub_space=k)
            f_ik = backbone_network.get_feature(ray, points[i], sub_space=k, dim=d)
            delta_i = distance(points[i], points[i-1]) if i > 0 else distance(ray.origin, points[0])
            
            # Standard volumetric opacity for this segment
            alpha_ik = 1.0 - exp(-sigma_ik * delta_i)
            
            # Transmittance up to this point (before this segment)
            # Note: Source formula uses T_i * (1 - exp(...)). 
            # T_i is typically exp(-sum_{j=1}^{i-1} sigma_j * delta_j).
            # We must maintain transmittance state for sub-space k.
            
            # Contribution of this segment
            # The source formula: sum(T_i * (1 - exp(-sigma_ik * delta_i)) * f_ik)
            # Here T_i is the transmittance BEFORE point i.
            segment_contribution = transmittance * alpha_ik * f_ik
            accumulated_feature += segment_contribution
            
            # Update transmittance for the next point
            transmittance *= (1.0 - alpha_ik)
            
        feature_maps.append(accumulated_feature)
        
    # 3. Process feature maps through MLPs
    # feature_maps is a list of K feature vectors F_k
    colors = decoder_mlp(feature_maps) # Outputs K RGB vectors C_k
    weights = gate_mlp(feature_maps)   # Outputs K scalar weights w_k
    
    # 4. Softmax combination
    # Compute softmax weights
    exp_weights = [exp(w) for w in weights]
    sum_exp_weights = sum(exp_weights)
    softmax_weights = [ew / sum_exp_weights for ew in exp_weights]
    
    # Combine colors
    final_color = sum(softmax_weights[k] * colors[k] for k in range(K))
    
    return final_color

# Helper MLPs definition (conceptual, as per source description)
class DecoderMLP(nn.Module):
    def __init__(self, d):
        super().__init__()
        # One hidden layer
        self.fc1 = nn.Linear(d, hidden_dim)
        self.fc2 = nn.Linear(hidden_dim, 3) # Output RGB
        self.relu = nn.ReLU()
        
    def forward(self, features):
        # features: List of K tensors of shape (d,)
        # Output: List of K tensors of shape (3,)
        return [self.fc2(self.relu(self.fc1(f))) for f in features]

class GateMLP(nn.Module):
    def __init__(self, d):
        super().__init__()
        # One hidden layer
        self.fc1 = nn.Linear(d, hidden_dim)
        self.fc2 = nn.Linear(hidden_dim, 1) # Output scalar
        self.relu = nn.ReLU()
        
    def forward(self, features):
        # features: List of K tensors of shape (d,)
        # Output: List of K scalars
        return [self.fc2(self.relu(self.fc1(f))).squeeze() for f in features]
```

**Edge Cases and Pitfalls**
*   **Precision:** The softmax function is sensitive to large weight values. Numerical stability should be maintained (e.g., subtracting max weight before exp), though the source does not specify implementation details [S1].
*   **Backbone Integration:** The method requires modifying the output layer of the backbone. For grid-based methods like TensoRF, the feature extraction must support $K$ parallel outputs [S1].
*   **No Additional Loss:** The method does not require additional loss terms compared to vanilla NeRF, relying solely on the standard photometric loss [S1].

## Operating characteristics
*   **Computational Overhead:** The method introduces only small computational overheads for training and inference [S1].
*   **Performance:**
    *   On Mip-NeRF 360: Improves PSNR by 4.15 dB on reflective regions with 0.5% extra parameters, without degrading rendering quality on other regions [S1].
    *   On TensoRF: Improves PSNR by 2.71 dB on reflective regions with 0.046% extra parameters, without degrading rendering quality on other regions [S1].
*   **Strengths:** Automatically handles mirror-like objects in 360-degree rendering without manual labeling; overcomes multi-view consistency violations by placing virtual images in view-specific sub-spaces [S1].
*   **Limitations:** Performance on non-reflective regions is not degraded, but the method is specifically designed for scenes with complex reflections and refractions [S1].
*   **When to use:** Scenes with reflective and refractive objects where multi-view consistency is violated in Euclidean space [S1].
*   **When not to use:** Simple scenes without reflective/refractive elements, where the overhead is unnecessary [S1].

## Background
MS-NeRF was proposed by Z.X. Yin, P.Y. Jiao, M.M. Cheng, B. Ren, and J. Qiu [S1]. A preliminary version appeared at CVPR, with the full version published in IEEE Transactions on Pattern Analysis and Machine Intelligence in 2025 (DOI: 10.1109/tpami.2025.3540074) [S1]. The authors are affiliated with Nankai University and Horizon Robotics [S1]. The method was developed to address the limitations of existing NeRF methods in handling reflective objects, which often result in blurry or distorted rendering [S1]. The authors also constructed a novel dataset consisting of 33 synthetic scenes and 7 real captured scenes with complex reflection and refraction to benchmark the robustness of NeRF-based methods [S1].

## Sources
[S1] MS-NeRF: Multi-Space Neural Radiance Fields, https://doi.org/10.1109/tpami.2025.3540074, fetched 2026-10-26
