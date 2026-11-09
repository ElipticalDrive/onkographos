# ClipGS

Also called: Clippable Gaussian Splatting

## Overview
ClipGS is a Gaussian Splatting-based framework designed for the real-time interactive cinematic visualization of volumetric medical data with support for dynamic clipping planes [S1]. It solves the problem of maintaining rendering quality and continuity when a user interactsively clips a 3D volume, addressing the artifacts caused by hard visibility truncations and the high computational cost of learning full deformation fields for the additional clipping dimension [S1]. It sets itself apart from sibling methods like [WildGaussians](6064-wildgaussians.md) or [Dynamic and Temporal Modeling](9265-dynamic-and-temporal-modeling.md) by treating the clipping plane not as a temporal dimension requiring a dense deformation field, but as a geometric visibility constraint managed via a learnable truncation scheme and an Adaptive Adjustment Model (AAM) [S1].

## How it works
ClipGS extends the standard 3D Gaussian Splatting representation to support 4D rendering (3D space + clipping plane parameter) by introducing two key components: Learnable Truncation (LT) for visibility control and an Adaptive Adjustment Model (AAM) for continuous deformation [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Volumetric medical data $V$ (e.g., CT scans).
    *   Camera parameters: view direction $\mathbf{v}_i$ and projection matrices.
    *   Clipping plane parameter: defined by normal $\boldsymbol{n}$ and distance to origin $z$.
    *   Ground-truth images $\mathbf{C}_{\text{gt}}$ generated from the volumetric data at specific camera views and clipping plane positions.
*   **Outputs:**
    *   Rendered color image $\mathbf{C}$ for the given view and clipping plane.
    *   Optimized 3D Gaussian primitives with attributes adjusted for the clipping context.

### Data Structures
The method operates on a set of $N$ 3D Gaussian primitives. Each primitive $i$ maintains the following attributes, optimized during training:
*   Position: $\boldsymbol{\mu}_i \in \mathbb{R}^3$
*   Rotation: $\mathbf{r}_i \in \mathbb{R}^3$ (represented as a quaternion or axis-angle, decomposed into rotation matrix $\boldsymbol{R}$)
*   Scale: $s_i \in \mathbb{R}$ (or vector $\boldsymbol{s}$, decomposed into scaling matrix $\boldsymbol{S}$)
*   Color: $\mathbf{c}_i \in \mathbb{R}^3$
*   Opacity: $\alpha_i \in \mathbb{R}$
*   **Learnable Truncation Attribute:** $m_i \in \mathbb{R}$, an optimizable scalar per Gaussian used to control visibility relative to the clipping plane.

The AAM uses a shared MLP structure to predict deformations. It takes positional encoding of the Gaussian position $\boldsymbol{\mu}$ and the clipping plane parameter $z$ as input.

### 3D Gaussian Primitive Definition
A 3D Gaussian primitive is defined as [S1]:
$$
\mathcal{G}(\mathbf{x}; \boldsymbol{\mu}, \boldsymbol{r}, s) = e^{-\frac{1}{2}(\boldsymbol{x}-\boldsymbol{\mu})^T \boldsymbol{\Sigma}^{-1}(\boldsymbol{x}-\boldsymbol{\mu})}
$$
where the covariance matrix $\boldsymbol{\Sigma}$ is decomposed into rotation $\boldsymbol{R}$ and scaling $\boldsymbol{S}$:
$$
\boldsymbol{\Sigma} = \boldsymbol{R} \boldsymbol{S} \boldsymbol{S}^T \boldsymbol{R}^T
$$
The projected 2D Gaussian $\mathcal{G}'_i$ is obtained by projecting the 3D Gaussian onto the image plane using camera intrinsic and extrinsic parameters.

### Learnable Truncation (LT)
Standard hard truncation (HT) uses the mean $\boldsymbol{\mu}$ to determine visibility: if $(\boldsymbol{\mu} \cdot \boldsymbol{n}) < z$, the Gaussian is visible. This causes artifacts because Gaussians intersecting the clipping plane (within their confidence interval) may have means on either side, leading to discontinuous visibility changes [S1].

ClipGS introduces a "barycenter" offset $\boldsymbol{\delta} \in \mathbb{R}^3$ and defines the visibility condition based on the adjusted center [S1]:
$$
\mathcal{M} = (\boldsymbol{\mu} + \boldsymbol{\delta}) \cdot \boldsymbol{n} < z
$$
where $\mathcal{M} \in \{0, 1\}$ is the visibility mask, $\boldsymbol{n}$ is the clipping plane normal, and $z$ is the distance to the origin.

To make this differentiable, ClipGS introduces an optimizable scalar attribute $m_i$ for each Gaussian, where $m_i = (\boldsymbol{\mu}_i + \boldsymbol{\delta}_i) \cdot \boldsymbol{n}$. The visibility function $\mathcal{M}$ is approximated using a straight-through estimator with a sigmoid function [S1]:
$$
\mathcal{M} = \text{sg}(\mathbf{1}[\sigma(m - z) < \epsilon] - \sigma(m - z)) + \sigma(m - z)
$$
where:
*   $\sigma(\cdot)$ is the sigmoid function.
*   $\mathbf{1}[\cdot]$ is the indicator function.
*   $\epsilon$ is a threshold hyperparameter (typical value: 0.5) [S1].
*   $\text{sg}(\cdot)$ is the stop-gradient operator.

This allows gradients to flow through the sigmoid approximation during backpropagation while maintaining a step-like behavior for the forward pass [S1].

### Adaptive Adjustment Model (AAM)
To ensure continuity across different clipping plane positions, ClipGS predicts dynamic deformations for visible Gaussians using an AAM [S1].

1.  **Feature Extraction:** The Gaussian position $\boldsymbol{\mu}$ and clipping plane parameter $z$ are passed through positional encoding $\gamma(\cdot)$ and merged into a feature vector $f$ via an MLP [S1]:
    $$
    f = \text{MLP}(\gamma(\boldsymbol{\mu}), \gamma(z))
    $$
2.  **Deformation Prediction:** Separate MLP heads $\phi_\mu, \phi_r, \phi_s$ predict the deformations for position, rotation, and scale [S1]:
    $$
    \Delta\boldsymbol{\mu} = \phi_\mu(f), \quad \Delta\mathbf{r} = \phi_r(f), \quad \Delta s = \phi_s(f)
    $$
3.  **Rendering with Deformation:** The final rendered color $\mathbf{C}$ is computed by $\alpha$-blending the deformed Gaussians, weighted by the visibility mask $\mathcal{M}_i$ [S1]:
    $$
    \mathcal{C} = \sum_{i=1}^N \mathcal{M}_i T_i \alpha_i \mathcal{G}'_i(\mathbf{x}; \boldsymbol{\mu}_i + \Delta\boldsymbol{\mu}_i, \mathbf{r}_i + \Delta\mathbf{r}_i, s_i + \Delta s_i) \mathbf{c}_i
    $$
    where $T_i = \prod_{j \in i-1} (1 - \mathcal{G}'_j(\mathbf{x}; \boldsymbol{\mu}_i + \Delta\boldsymbol{\mu}_i, \mathbf{r}_i + \Delta\mathbf{r}_i, s_i + \Delta s_i) \alpha_j)$ is the accumulated transmittance.

### Training Procedure
The optimization is performed in two steps [S1]:

1.  **Coarse Initialization (Learnable Truncation):**
    *   Generate a sparse cinematic image sequence $\{I_i\}$ from random camera views $\mathbf{v}_i$ and random clipping planes $z_i$ using a ray-tracing renderer.
    *   Randomly initialize $N$ 3D Gaussians within a cube encompassing the scene (typical $N=100,000$) [S1].
    *   Optimize the Gaussian attributes and the learnable truncation attribute $m$ for 7,000 iterations.
    *   Loss function:
        $$
        \mathcal{L} = (1 - \lambda)\mathcal{L}_1(\mathbf{C}, \mathbf{C}_{\text{gt}}) + \lambda\mathcal{L}_{\text{D-SSIM}}(\mathbf{C}, \mathbf{C}_{\text{gt}})
        $$
        where $\lambda$ is the loss weight (typical value: 0.2) [S1].

2.  **Joint Optimization (AAM):**
    *   Initialize the AAM parameters.
    *   Jointly optimize the Gaussian attributes, learnable truncation attributes, and AAM parameters for 33,000 iterations.
    *   This step refines rendering quality and ensures continuity in the clipping plane dimension [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class ClipGS:
    def __init__(self, num_gaussians, scene_bounds):
        """
        Initialize ClipGS model.
        
        Args:
            num_gaussians (int): Number of Gaussian primitives (e.g., 100,000).
            scene_bounds (tuple): Bounding box of the scene for initialization.
        """
        self.num_gaussians = num_gaussians
        
        # Initialize Gaussian attributes
        # Position: random within scene bounds
        self.mu = torch.rand(num_gaussians, 3) * (scene_bounds[1] - scene_bounds[0]) + scene_bounds[0]
        
        # Rotation: random unit quaternion or axis-angle (simplified here as random 3D vector)
        self.r = torch.rand(num_gaussians, 3) * 2 * torch.pi
        
        # Scale: random positive values
        self.s = torch.rand(num_gaussians, 1) * 0.1 + 0.01
        
        # Color: random RGB
        self.c = torch.rand(num_gaussians, 3)
        
        # Opacity: random initial opacity
        self.alpha = torch.rand(num_gaussians, 1) * 0.5 + 0.1
        
        # Learnable Truncation attribute m
        self.m = torch.zeros(num_gaussians, 1)
        
        # Adaptive Adjustment Model (AAM)
        self.aam = AdaptiveAdjustmentModel()
        
    def learnable_truncation_mask(self, m, z, epsilon=0.5):
        """
        Compute visibility mask using Learnable Truncation.
        
        Args:
            m (Tensor): Learnable truncation attribute (N, 1).
            z (float): Clipping plane distance to origin.
            epsilon (float): Threshold for sigmoid approximation.
            
        Returns:
            Tensor: Visibility mask (N, 1) with values in [0, 1].
        """
        # Sigmoid approximation
        sig = torch.sigmoid(m - z)
        
        # Indicator function (hard threshold)
        indicator = (sig < epsilon).float()
        
        # Straight-through estimator
        # Forward pass: use indicator (hard step)
        # Backward pass: use sigmoid gradient
        mask = indicator + (sig - indicator).detach()
        
        return mask

    def compute_deformation(self, mu, z):
        """
        Compute dynamic deformations using AAM.
        
        Args:
            mu (Tensor): Gaussian positions (N, 3).
            z (float): Clipping plane distance to origin.
            
        Returns:
            Tuple: Deformations (delta_mu, delta_r, delta_s)
        """
        # Positional encoding (simplified placeholder for actual PE implementation)
        # In practice, gamma(mu) and gamma(z) would be high-frequency positional encodings
        mu_pe = positional_encoding(mu)
        z_pe = positional_encoding(torch.tensor([z]).expand(mu.shape[0], -1))
        
        # Merge and pass through MLP
        f = self.aam(mu_pe, z_pe)
        
        # Predict deformations
        delta_mu = self.aam.phi_mu(f)
        delta_r = self.aam.phi_r(f)
        delta_s = self.aam.phi_s(f)
        
        return delta_mu, delta_r, delta_s

    def render(self, camera_params, clipping_plane_z, epsilon=0.5):
        """
        Render the scene with clipping plane support.
        
        Args:
            camera_params (dict): Camera intrinsic and extrinsic parameters.
            clipping_plane_z (float): Clipping plane distance to origin.
            epsilon (float): Threshold for learnable truncation.
            
        Returns:
            Tensor: Rendered color image.
        """
        # 1. Compute visibility mask
        mask = self.learnable_truncation_mask(self.m, clipping_plane_z, epsilon)
        
        # 2. Compute deformations for visible Gaussians
        delta_mu, delta_r, delta_s = self.compute_deformation(self.mu, clipping_plane_z)
        
        # 3. Apply deformations
        mu_deformed = self.mu + delta_mu
        r_deformed = self.r + delta_r
        s_deformed = self.s + delta_s
        
        # 4. Project Gaussians to 2D and compute 2D Gaussian parameters
        # This step involves projecting 3D Gaussians to the image plane
        # and computing the 2D covariance matrix based on camera parameters
        projected_gaussians = project_gaussians(mu_deformed, r_deformed, s_deformed, camera_params)
        
        # 5. Rasterize and alpha-blend
        # Sort Gaussians by depth
        # For each pixel, accumulate color based on alpha blending
        # Only include Gaussians where mask > 0
        color = rasterize_and_blend(projected_gaussians, self.c, self.alpha, mask, camera_params)
        
        return color

    def train_step(self, ground_truth_image, camera_params, clipping_plane_z, lambda_dssim=0.2):
        """
        Perform one training step.
        
        Args:
            ground_truth_image (Tensor): Ground truth color image.
            camera_params (dict): Camera parameters.
            clipping_plane_z (float): Clipping plane distance.
            lambda_dssim (float): Weight for D-SSIM loss.
            
        Returns:
            float: Training loss.
        """
        # Render current frame
        rendered_image = self.render(camera_params, clipping_plane_z)
        
        # Compute L1 loss
        l1_loss = F.l1_loss(rendered_image, ground_truth_image)
        
        # Compute D-SSIM loss
        dssim_loss = dssim_loss_function(rendered_image, ground_truth_image)
        
        # Total loss
        loss = (1 - lambda_dssim) * l1_loss + lambda_dssim * dssim_loss
        
        # Backpropagate and optimize
        loss.backward()
        
        return loss.item()

class AdaptiveAdjustmentModel(nn.Module):
    def __init__(self, input_dim=64, hidden_dim=128, output_dim=3):
        super(AdaptiveAdjustmentModel, self).__init__()
        # MLP for feature extraction
        self.mlp = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU()
        )
        
        # Separate MLP heads for deformations
        self.phi_mu = nn.Linear(hidden_dim, 3)
        self.phi_r = nn.Linear(hidden_dim, 3)
        self.phi_s = nn.Linear(hidden_dim, 1)
        
    def forward(self, mu_pe, z_pe):
        # Merge inputs
        f = torch.cat([mu_pe, z_pe], dim=-1)
        f = self.mlp(f)
        return f

def positional_encoding(x, min_deg=0, max_deg=10, power_factor=2):
    """
    Simple positional encoding for demonstration.
    In practice, use the standard 3DGS positional encoding.
    """
    # Placeholder for actual PE implementation
    # Returns a higher-dimensional representation of x
    return x

def project_gaussians(mu, r, s, camera_params):
    """
    Project 3D Gaussians to 2D image plane.
    """
    # Placeholder for actual projection logic
    # Computes 2D mean, covariance, and bounding box for each Gaussian
    pass

def rasterize_and_blend(projected_gaussians, colors, opacities, masks, camera_params):
    """
    Rasterize Gaussians and perform alpha blending.
    """
    # Placeholder for actual rasterization logic
    # Sort Gaussians by depth, iterate over pixels, and blend
    pass

def dssim_loss_function(rendered, ground_truth):
    """
    Compute D-SSIM loss.
    """
    # Placeholder for actual D-SSIM implementation
    pass
```

### Parameters
*   **$\epsilon$ (Truncation threshold):** Used in the learnable truncation sigmoid approximation. Typical value: 0.5 [S1].
*   **$\lambda$ (Loss weight):** Balances L1 and D-SSIM terms in the training loss. Typical value: 0.2 [S1].
*   **Initial Gaussian count ($N$):** Number of points randomly initialized within a cube encompassing the scene. Typical value: 100,000 [S1].
*   **Training iterations:** 7,000 iterations for the first step (LT only), followed by 33,000 iterations for joint optimization (LT + AAM) [S1].

### Edge Cases and Pitfalls
*   **Hard Truncation Artifacts:** Using the mean $\boldsymbol{\mu}$ directly for visibility (Hard Truncation) causes artifacts at the clipping surface because Gaussians intersecting the plane may have means on either side, leading to discontinuous visibility changes [S1].
*   **Non-Contiguity:** The indicator operation in the visibility mask is mathematically non-contiguous, causing inconsistent rendering when the clipping plane changes. The AAM is designed to mitigate this by predicting continuous deformations [S1].
*   **Model Capacity:** Unlike methods that learn a full deformation field for the clipping dimension (e.g., HexPlane, GauFRe), ClipGS focuses on visible primitives, reducing model capacity and storage requirements [S1].

## Operating characteristics
*   **Time Complexity:** The method is GPU-accelerated and designed for real-time rendering. The training involves two phases: 7,000 iterations for coarse initialization and 33,000 iterations for joint optimization [S1].
*   **Memory Complexity:** The model size is approximately 16.1 MB for the trained Gaussians and AAM parameters [S1]. This is significantly smaller than methods that learn full deformation fields for the clipping dimension [S1].
*   **Performance:** Achieves an average PSNR of 36.635 on five volumetric medical data scenes (Head, Lower Limb, Skull, Thorax, Liver & Kidney) [S1]. Rendering speed is 156 FPS on a single RTX 4090 GPU [S1].
*   **Strengths:** Real-time interactive visualization, low storage cost, high rendering quality, and continuity across clipping planes [S1].
*   **Limitations:** Requires a two-step optimization process. The AAM adds complexity to the training loop compared to standard 3DGS [S1].
*   **When to use:** Suitable for interactive cinematic visualization of volumetric medical data where clipping plane interaction is required [S1].
*   **When not to use:** Not suitable for static scene rendering without clipping plane requirements, or for scenes where the clipping plane is not a primary interaction mode [S1].

## Background
ClipGS was introduced in a paper titled "ClipGS: Clippable Gaussian Splatting for Interactive Cinematic Visualization of Volumetric Medical Data" [S1]. The work addresses the limitations of high computing cost and low rendering speed in traditional volumetric medical data visualization, aiming to enable interactive cinematic rendering [S1]. It builds on the 3D Gaussian Splatting framework [13] and extends it to support clipping plane interactions, which are crucial for medical diagnosis and surgical planning [S1]. The method was validated on datasets from the Visible Human Project and the CVH project [S1].

## Sources
[S1] ClipGS: Clippable Gaussian Splatting for Interactive Cinematic Visualization of Volumetric Medical Data, https://doi.org/10.1007/978-3-032-05127-1_13, fetched 2026-10-26
