# DASNeRF

Also called: DASNeRF: depth consistency optimization, adaptive sampling, and hierarchical structural fusion for sparse view neural radiance fields

## Overview
DASNeRF is a sparse-view Neural Radiance Field (NeRF) optimization method that improves reconstruction detail and robustness by integrating accurate monocular depth priors and a dynamic sampling mechanism [S1]. It distinguishes itself from standard NeRF and sibling methods like SfMNeRF by explicitly enforcing relative depth ordering fidelity and structural consistency via a pre-trained DepthAnythingV2 model, while simultaneously reducing lighting artifacts through a three-layer dynamic sampling strategy [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sparse set of $M$ RGB images $\{I_m\}_{m=1}^M$ with corresponding camera poses $\{p_m\}_{m=1}^M$.
*   **Outputs:** A trained implicit radiance field model $F$ that predicts density $\sigma$ and color $\mathbf{c}$, and rendered RGB images and depth maps for novel views.

**Data Structures:**
1.  **Depth Prior Map ($d_{V2}$):** A 2D array of depth values estimated by the external DepthAnythingV2 model for each input image. The source specifies processing this data using "relative inverse depth" [S1].
2.  **Sampled Points:** A set of 5-dimensional vectors. Unlike standard NeRF which uses 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$, DASNeRF extracts "five-dimensional information" from a dynamic sampling mechanism [S1]. The source states this includes "coordinates and ray information" but does not explicitly define the 5 components (e.g., whether it is $(x,y,z, d_x, d_y)$ or similar).
3.  **MLP State:** A Multi-Layer Perceptron with per-layer input fusion. Each layer $l$ takes two inputs: the feature vector from the previous layer $h_{l-1}$ and the processed five-dimensional input information $v$ [S1].

**Numbered Steps:**

1.  **Depth Prior Estimation:**
    For each input image $I_m$, use the pre-trained DepthAnythingV2 model to estimate the depth map $d_{V2}^m$. The source notes that DepthAnythingV2 is preferred over MiDaS due to better handling of transparent/reflective objects and 10x faster inference [S1]. The depth data is processed using relative inverse depth [S1].

2.  **Dynamic Optimal Sampling:**
    For a ray $r$ defined by origin $o$ and direction $d$, intersecting the scene within near/far bounds $[t_n, t_f]$:
    *   Employ 3D cone frustum sampling from Mip-NeRF to account for antialiasing [S1].
    *   Apply a "three-layer dynamic sampling mechanism" to determine the number and placement of samples $N$ along the ray [S1]. This mechanism is designed to reduce the impact of lighting on detail reconstruction [S1].
    *   Extract the five-dimensional information $v$ for each sampled point $t_i$ along the ray [S1].

3.  **Neural Radiance Field Forward Pass:**
    For each sampled point with five-dimensional input $v$:
    *   Pass $v$ through the MLP with per-layer input fusion.
    *   The MLP outputs volumetric density $\sigma_i$ and color $\mathbf{c}_i$ [S1].
    *   Note: The source describes the standard NeRF mapping $(\sigma, \mathbf{c}) = f(\gamma(\mathbf{x}), \gamma(\mathbf{d}))$ as the baseline, but DASNeRF modifies the input to the MLP to be the five-dimensional vector derived from the dynamic sampling layer [S1].

4.  **Volume Rendering:**
    Compute the transmittance $T_i$ and alpha $\alpha_i$ for each sample $i$:
    $$
    \alpha_i = 1 - \exp(-\sigma_i \delta_i)
    $$
    $$
    T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)
    $$
    where $\delta_i$ is the distance between adjacent samples along the ray [S1].
    
    Render the pixel color $\hat{C}(r)$:
    $$
    \hat{C}(r) = \sum_{i=1}^{N} \omega_i \mathbf{c}_i, \quad \text{where } \omega_i = T_i \alpha_i
    $$
    [S1]

    Render the predicted depth $d_r$ for the ray:
    $$
    d_r = \sum_{i=1}^{N} \omega_i t_i
    $$
    [S1]

5.  **Loss Calculation:**
    The total loss $L$ is the sum of the NeRF color loss, Depth Ordering Fidelity Loss ($L_{DOF}$), and Depth Structural Consistency Regularization ($R_{DSC}$):
    $$
    L = L_{nerf} + \lambda L_{DOF} + \gamma R_{DSC}
    $$
    [S1]

    *   **NeRF Color Loss ($L_{nerf}$):** Mean squared error between the ground truth pixel color $C(r)$ and the rendered color $\hat{C}(r)$:
        $$
        L_{nerf} = \sum_{r \in R} \| C(r) - \hat{C}(r) \|_2^2
        $$
        where $R$ is the set of all rays [S1].

    *   **Depth Ordering Fidelity Loss ($L_{DOF}$):** Enforces that the relative depth order of the predicted depth $d_r$ matches the depth prior $d_{V2}$. For two pixels $k_1$ and $k_2$ in a local region $P$:
        $$
        L_{DOF} = \sum_{d_{V2}^{k1} \leq d_{V2}^{k2}} \max(d_r^{k1} - d_r^{k2} + m_{ar}, 0)
        $$
        where $d_{V2}^{k1}$ and $d_{V2}^{k2}$ are depth values from the prior, $d_r^{k1}$ and $d_r^{k2}$ are predicted depths, and $m_{ar}$ is a dynamic boundary adjustment [S1].

    *   **Dynamic Boundary Adjustment ($m_{ar}$):**
        $$
        m_{ar} = \alpha \cdot \frac{1}{N} \sum_{i=1}^{N} |d_r^{k1}[i] - d_r^{k2}[i]|
        $$
        where $\alpha$ is a tunable hyperparameter, $N$ is the number of depth samples in the computation area, and $d_r^{k1}[i]$ denotes the depth value of the $i$-th sample layer for pixel $k_1$ [S1].

    *   **Depth Structural Consistency Regularization ($R_{DSC}$):** Enforces continuity between neighboring pixels. For a pixel $k_1$ and its k-nearest neighbors $k_2$ in the depth prior:
        $$
        R_{DSC} = \sum_{k1} \sum_{d_{V2}^{k2} \in KNN(d_{V2}^{k2})} \max(|d_r^{k1} - d_r^{k2}| - m_{ar}', 0)
        $$
        where $KNN(\cdot)$ returns k-nearest neighbors based on depth values within a small region, and $m_{ar}'$ is a small boundary value for minor variations [S1].

6.  **Optimization:**
    Minimize $L$ using gradient descent. Typical parameter values cited are $\lambda = 0.2$ and $\gamma = 0.02$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DASNeRF:
    def __init__(self, num_samples=128, lambda_dof=0.2, gamma_dsc=0.02, alpha_margin=0.1):
        """
        Initialize DASNeRF model.
        
        Args:
            num_samples: Number of samples per ray (N).
            lambda_dof: Weight for Depth Ordering Fidelity Loss.
            gamma_dsc: Weight for Depth Structural Consistency Regularization.
            alpha_margin: Hyperparameter for dynamic boundary adjustment.
        """
        self.num_samples = num_samples
        self.lambda_dof = lambda_dof
        self.gamma_dsc = gamma_dsc
        self.alpha_margin = alpha_margin
        
        # MLP with per-layer input fusion
        # Input dimension is 5 (five-dimensional information from dynamic sampling)
        # Hidden layers and output heads are not fully specified in source [S1]
        # Assuming a standard MLP structure for density and color
        self.mlp = nn.Sequential(
            nn.Linear(5, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            # ... additional layers as per source architecture (not fully detailed)
            nn.Linear(256, 4) # Output: density (1) + color (3)
        )
        
        # Pre-trained DepthAnythingV2 model
        self.depth_estimator = load_depth_anything_v2()

    def get_depth_prior(self, image):
        """
        Get depth prior from DepthAnythingV2.
        Source specifies processing using relative inverse depth.
        """
        # Placeholder for actual DepthAnythingV2 inference
        depth_map = self.depth_estimator(image)
        # Process using relative inverse depth as per source [S1]
        depth_prior = 1.0 / (depth_map + 1e-6) 
        return depth_prior

    def dynamic_sampling(self, ray_o, ray_d, near, far):
        """
        Perform 3D cone frustum sampling and dynamic layer sampling.
        Returns sampled points and their 5D features.
        """
        # 1. Cone frustum sampling from Mip-NeRF
        # Calculate intervals based on ray length and cone angle
        # This step determines the distribution of t_i along [near, far]
        t_vals = torch.linspace(near, far, self.num_samples)
        
        # 2. Three-layer dynamic sampling mechanism
        # Source mentions this reduces lighting impact but does not give the exact algorithm
        # for the "three-layer" logic. We assume uniform or stratified sampling 
        # adjusted by the dynamic mechanism.
        # [G] Standard stratified sampling is used as a baseline for volumetric rendering
        # when specific dynamic layer logic is not provided.
        t_vals = t_vals + torch.rand_like(t_vals) * (far - near) / self.num_samples
        
        # 3. Extract 5D information
        # Source states this includes "coordinates and ray information"
        # Coordinates: x = ray_o + t * ray_d
        # Ray info: likely direction d or positional encoding components
        # Since the exact 5 dimensions are not defined, we construct a placeholder
        # based on standard NeRF inputs (x, d) which is 6D, or (x) which is 3D.
        # The source explicitly says 5D. 
        # [G] Assuming the 5D vector consists of the 3D position and 2D normalized 
        # viewing direction components (e.g., azimuth/elevation or x/y of direction),
        # or potentially positional encoding components. Without explicit definition,
        # we pass the raw 3D position and direction components.
        points_3d = ray_o.unsqueeze(1) + t_vals.unsqueeze(0).unsqueeze(-1) * ray_d.unsqueeze(0).unsqueeze(-1)
        
        # Construct 5D feature vector
        # Note: This is an implementation guess for the undefined 5D vector.
        # In practice, this would be processed by the specific dynamic sampling layer.
        # We flatten the direction to 2 components if normalized, or use specific encodings.
        # Let's assume the 5D vector is [x, y, z, d_x, d_y] for a normalized direction.
        d_normalized = ray_d / (torch.norm(ray_d, dim=-1, keepdim=True) + 1e-6)
        features_5d = torch.cat([points_3d, d_normalized.unsqueeze(1).expand_as(points_3d)], dim=-1)
        
        return features_5d, t_vals

    def forward(self, ray_o, ray_d, near, far, image, depth_prior_map):
        """
        Forward pass for a single ray and its corresponding image patch.
        """
        # 1. Dynamic Sampling
        features_5d, t_vals = self.dynamic_sampling(ray_o, ray_d, near, far)
        
        # 2. MLP Forward Pass with Per-Layer Input Fusion
        # Source: "Each MLP layer takes two inputs: the features from the previous layer 
        # and the processed five-dimensional input information."
        # This implies a residual connection or concatenation at every layer.
        # Standard MLP implementation:
        x = features_5d
        # Assuming 2 hidden layers for demonstration, as source doesn't specify count
        # Layer 1
        h1 = self.mlp[0](x) # Linear(5, 256)
        h1 = self.mlp[1](h1) # ReLU
        # Per-layer fusion: Add input features to hidden representation?
        # Source says "takes two inputs". Usually implies concatenation or addition.
        # [G] Concatenation is standard for "input fusion".
        x_fused_1 = torch.cat([h1, x], dim=-1)
        h2 = self.mlp[2](x_fused_1) # Linear(256+5, 256)
        h2 = self.mlp[3](h2) # ReLU
        
        # Output Head
        # The source says "After the MLP output, obtain color information c and volumetric density sigma"
        # Assuming the final linear layer outputs 4 values: [sigma, r, g, b]
        out = self.mlp[4](h2) # Linear(256, 4)
        
        sigma = out[..., 0]
        color = torch.sigmoid(out[..., 1:]) # Color is typically bounded [0,1]
        
        # 3. Volume Rendering
        deltas = t_vals[1:] - t_vals[:-1]
        # Add infinity to the last delta for the transmittance calculation of the last sample
        deltas = torch.cat([deltas, torch.tensor([1e10], device=deltas.device)], dim=-1)
        
        alpha = 1.0 - torch.exp(-sigma * deltas)
        transmittance = torch.cumprod(torch.cat([torch.ones_like(alpha[..., :1]), 1.0 - alpha[..., :-1]], dim=-1), dim=-1)
        
        weights = alpha * transmittance
        rendered_color = torch.sum(weights.unsqueeze(-1) * color, dim=-2)
        rendered_depth = torch.sum(weights * t_vals, dim=-1)
        
        # 4. Loss Calculation
        # Ground truth color from image (projected to ray)
        gt_color = self.get_pixel_color_from_image(image, ray_o, ray_d) # Helper not defined in source
        
        loss_nerf = F.mse_loss(rendered_color, gt_color)
        
        # Depth Regularization
        # Requires local region P and corresponding depth prior d_V2
        # This is computed over a batch of rays or a local window
        loss_dof, loss_dsc = self.compute_depth_losses(rendered_depth, depth_prior_map, ray_o, ray_d)
        
        total_loss = loss_nerf + self.lambda_dof * loss_dof + self.gamma_dsc * loss_dsc
        
        return rendered_color, rendered_depth, total_loss

    def compute_depth_losses(self, rendered_depth, depth_prior_map, ray_o, ray_d):
        """
        Compute L_DOF and R_DSC.
        """
        # Extract local region P from rendered depth and depth prior
        # Source: "crop a local depth map d_V2 that corresponds to the same spatial location"
        # This requires mapping ray origins/directions to 2D pixel coordinates.
        # [G] Assuming we have a function to project 3D rays to 2D pixels (u, v)
        pixels_u, pixels_v = self.project_to_2d(ray_o, ray_d)
        
        # Get depth values for pixels k1 and k2
        # Source: k1, k2 are 2D pixel coordinates
        # We need to select pairs where d_V2[k1] <= d_V2[k2]
        
        # Placeholder for local region extraction
        # In practice, this involves iterating over pixels in a window
        d_r_k1 = self.sample_depth(rendered_depth, pixels_u, pixels_v)
        d_v2_k1 = self.sample_depth(depth_prior_map, pixels_u, pixels_v)
        
        # Generate pairs (k1, k2) in local neighborhood
        # Source: KNN based on depth values within a small region
        # This is computationally expensive and source doesn't give the exact loop structure
        # [G] Simplified implementation for demonstration
        
        # Dynamic boundary adjustment m_ar
        # m_ar = alpha * mean(|d_r[k1] - d_r[k2]|) over samples
        # Source: N represents number of depth samples in computation area
        # This implies averaging over the ray samples for the specific pixels?
        # Or averaging over pixels in the local region?
        # Formula: 1/N sum |d_r^k1[i] - d_r^k2[i]| where i is sampling layer
        # This suggests comparing depth profiles along the ray?
        # However, d_r is a single scalar per ray.
        # [G] The formula likely refers to comparing depth values at different 
        # sampling points if depth was volume-rendered per layer, but standard NeRF
        # renders a single depth. The source might imply comparing depth values 
        # of neighboring pixels k1 and k2 across multiple rays or samples.
        # Given the ambiguity, we calculate m_ar based on the mean depth difference 
        # of the selected pixel pairs in the local region.
        
        # Calculate m_ar
        # Assuming we have a batch of pairs (k1, k2)
        depth_diffs = torch.abs(d_r_k1 - d_r_k2)
        m_ar = self.alpha_margin * torch.mean(depth_diffs)
        
        # L_DOF
        # Sum over pairs where d_V2[k1] <= d_V2[k2]
        mask = d_v2_k1 <= d_v2_k2
        loss_dof = torch.mean(torch.relu(d_r_k1[mask] - d_r_k2[mask] + m_ar))
        
        # R_DSC
        # KNN of k2 in depth space
        # [G] Placeholder for KNN computation
        # R_DSC = sum max(|d_r[k1] - d_r[k2]| - m_ar', 0)
        loss_dsc = torch.mean(torch.relu(torch.abs(d_r_k1 - d_r_k2) - m_ar))
        
        return loss_dof, loss_dsc

    def get_pixel_color_from_image(self, image, ray_o, ray_d):
        # Helper to sample ground truth color at ray intersection
        pass

    def project_to_2d(self, ray_o, ray_d):
        # Helper to project 3D rays to 2D pixel coordinates
        pass

    def sample_depth(self, depth_map, u, v):
        # Helper to sample depth map at coordinates u, v
        pass
```

**Parameters:**
*   $\lambda$: Controls the importance of the Depth Ordering Fidelity Loss. Typical value: 0.2 [S1].
*   $\gamma$: Controls the importance of the Depth Structural Consistency Regularization. Typical value: 0.02 [S1].
*   $\alpha$: Tunable hyperparameter controlling the magnitude of the margin in the dynamic boundary adjustment formula [S1]. Value not explicitly given in sources.
*   $N$: Number of samples along the ray. Not explicitly given, but standard NeRF uses 64-256. Source mentions "three-layer dynamic sampling" but not the total count [S1].

**Edge Cases and Pitfalls:**
*   **Depth Prior Accuracy:** The method relies heavily on DepthAnythingV2. If the depth prior is inaccurate (e.g., in textureless regions or transparent objects), the regularization may force the NeRF to learn incorrect geometry [S1].
*   **Dynamic Boundary Adjustment:** The formula for $m_{ar}$ depends on the mean depth difference. If the depth difference is zero or very small, $m_{ar}$ becomes small, potentially leading to over-constraining the depth order [S1].
*   **5D Input Definition:** The exact composition of the "five-dimensional information" is not explicitly defined in the source, only that it includes "coordinates and ray information" [S1]. Implementers must make assumptions (e.g., 3D position + 2D direction components).
*   **KNN Computation:** The Depth Structural Consistency Regularization requires finding k-nearest neighbors in depth space within a local region. This can be computationally intensive and requires careful implementation of the neighborhood search [S1].

## Operating characteristics
*   **Time Complexity:** The method adds overhead for depth prior estimation (DepthAnythingV2 inference) and the calculation of depth regularization losses ($L_{DOF}$ and $R_{DSC}$), which involve pairwise comparisons or KNN searches over local regions [S1]. The dynamic sampling mechanism also adds computational cost compared to uniform sampling [S1].
*   **Memory Complexity:** Requires storing the depth prior maps for all input images and the intermediate features for the MLP with per-layer input fusion [S1].
*   **Performance:** DASNeRF is designed for sparse-view reconstruction, outperforming existing methods in detail preservation when few input images are available [S1]. DepthAnythingV2 provides 10x faster inference than MiDaS, reducing the preprocessing time for depth priors [S1].
*   **Strengths:** Improved detail reconstruction in sparse views; robust handling of transparent/reflective objects via DepthAnythingV2; reduced lighting impact via dynamic sampling [S1].
*   **Limitations:** Relies on the quality of the monocular depth estimator; increased training complexity due to additional loss terms and dynamic sampling; undefined 5D input vector requires implementation assumptions [S1].

## Background
DASNeRF was proposed to address the challenge of sparse-view 3D reconstruction using Neural Radiance Fields. Standard NeRF requires hundreds of images for high-fidelity results, limiting practical use [S1]. The method integrates depth priors from monocular depth estimation (specifically DepthAnythingV2) to guide the learning process, aiming to improve sparse reconstruction details [S1]. It was published in PLOS ONE [S1].

## Sources
[S1] DASNeRF: depth consistency optimization, adaptive sampling, and hierarchical structural fusion for sparse view neural radiance fields., https://doi.org/10.1371/journal.pone.0321878, fetched 2026-10-27
