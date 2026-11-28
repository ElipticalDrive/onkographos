# Dynamic Neural Radiance Fields

Also called: 4D Facial Avatar Reconstruction, Dynamic NeRF, Dynamic Neural Radiance Field (Dynamic NeRF) for Facial Avatars, Dynamic Neural Radiance Fields for Monocular 4D Facial Avatar Reconstruction, Dynamic Neural Scene Representation Network, Dynamic Radiance Field

## Overview
Dynamic Neural Radiance Fields (Dynamic NeRF) is a method for modeling the appearance and dynamics of a human face using an implicit volumetric representation [S1]. It combines a neural scene representation network with a low-dimensional morphable model to handle facial dynamics, allowing novel viewpoint and expression synthesis from monocular input [S1]. Unlike static NeRF methods, this approach handles dynamically changing surfaces by conditioning the neural network on morphable model expression parameters and a per-frame latent code [S1]. It differs from other volumetric approaches by requiring only a single view from a fixed camera (e.g., webcam) instead of an expensive calibrated multi-view rig [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A monocular portrait video sequence of a subject, a static background image $B$ (captured without the subject), and a 3D morphable model (3DMM) [S1].
*   **Output:** A rendered RGB image for a specified head pose and facial expression [S1].

**Data Structures**
*   **Morphable Model Parameters:**
    *   Pose $P_i \in \mathbb{R}^{4 \times 4}$: Rigid pose matrix (rotation and translation) for frame $i$ [S1].
    *   Expression $\delta_i \in \mathbb{R}^{76}$: Coefficients of a low-dimensional delta-blendshape basis [S1].
*   **Latent Code:**
    *   $\gamma_i \in \mathbb{R}^{32}$: A learnable per-frame latent code vector [S1].
*   **Network Weights:**
    *   $\theta_{coarse}$: Weights for the coarse dynamic radiance field MLP [S1].
    *   $\theta_{fine}$: Weights for the fine dynamic radiance field MLP [S1].
*   **Positional Encoding:**
    *   Position $p$: Encoded with 10 frequencies [S1].
    *   Viewing direction $\vec{v}$: Encoded with 4 frequencies [S1].

**Numbered Steps**

1.  **Preprocessing (Tracking):** Apply facial expression tracking using a 3D morphable model to estimate pose parameters $P_i$ and expression parameters $\delta_i$ for each frame $i$ in the training video [S1].

2.  **Ray Dispatch:** For each frame to be synthesized, dispatch rays in a canonical space shared by all frames. Transform the rays from camera space to this canonical space using the pose parameters $P_i$ from the morphable model [S1].

3.  **Coarse Sampling:** Perform stratified sampling of 64 points along each viewing ray [S1].
    *   95% of rays correspond to pixels within the head bounding box [S1].
    *   Sample 2048 viewing rays per training image/iteration [S1].

4.  **Coarse Network Evaluation:** For each sample point $t$ on ray $r$, input its canonical position $p$, view direction $\vec{v}$, expression parameters $\delta_i$, and latent code $\gamma_i$ into the coarse dynamic radiance field MLP $D_{\theta_{coarse}}$ [S1].
    *   The MLP outputs density $\sigma$ and color $RGB$ [S1].

5.  **Importance Sampling:** Use the density distribution predicted by the coarse network to guide importance sampling for the fine network [S1].

6.  **Fine Sampling:** Re-sample 64 points along the ray based on the coarse density distribution [S1].

7.  **Fine Network Evaluation:** For each fine sample point, input $p$, $\vec{v}$, $\delta_i$, and $\gamma_i$ into the fine dynamic radiance field MLP $D_{\theta_{fine}}$ [S1].
    *   The MLP outputs density $\sigma$ and color $RGB$ [S1].

8.  **Volumetric Integration:** Compute the expected color $C(r)$ for the ray by integrating the density and color values predicted by the fine network along the ray [S1]. The transmittance $T(t)$ is accumulated along the ray [S1].

9.  **Background Handling:** For the last sample point of each ray (representing the background), set the color to the corresponding value from the static background image $B$ [S1]. The network learns to predict low density for foreground samples if the ray hits a background pixel, and high density for foreground pixels, achieving self-supervised foreground-background decomposition [S1].

10. **Loss Computation:** Compute the reconstruction loss for both coarse and fine networks against the ground truth image $I_i$ [S1].

11. **Optimization:** Update $\theta_{coarse}$, $\theta_{fine}$, and $\gamma_i$ using the Adam optimizer [S1].

12. **Test Time Synthesis:** At test time, always employ the latent code $\gamma$ from the first frame of the training set [S1].

**Formulas**

The dynamic radiance field MLP $D_\theta$ takes position $p$, view direction $\vec{v}$, facial expression dynamics $\delta$, and a per-frame latent code $\gamma$ as inputs, and outputs color (RGB) and density ($\sigma$) [S1]:

$$ D_\theta (p, \vec{v} , \delta, \gamma) = (RGB, \sigma) $$

The expected color $C(r)$ of a camera ray $r(t) = c + t\vec{d}$ with camera center $c$, viewing direction $\vec{d}$, and near/far bounds $z_{near}, z_{far}$ is evaluated as [S1]:

$$ C(r; \theta, P, \delta, \gamma) = \int_{z_{near}}^{z_{far}} \sigma_\theta(r(t)) \cdot RGB_\theta(r(t), \vec{d}) \cdot T(t) \, dt $$

Where $T(t)$ is the accumulated transmittance along the ray [S1]. Note: The source text contains a typo in the integrand argument for $RGB$ ($\vec{d} \cdot T(t)$), but standard volumetric rendering and the context of $T(t)$ being transmittance imply $T(t)$ is a scalar multiplier outside the color function or part of the transmittance term. The standard definition is $\sigma \cdot RGB \cdot T$. The source writes: $\sigma_\theta (r (t)) \cdot RGB_\theta r (t) , \vec{d} \cdot T (t)dt$. Given $T(t)$ is transmittance, it is a scalar. The dot product $\vec{d} \cdot T(t)$ is invalid. It is interpreted as $RGB_\theta(\dots) \cdot T(t)$ [S1, G].

The total loss is the sum of losses for the coarse and fine networks [S1]:

$$ L_{total} = \sum_{i=1}^M (L_i(\theta_{coarse}) + L_i(\theta_{fine})) $$

Where the loss for a single image $i$ is [S1]:

$$ L_i(\theta) = \sum_{j \in pixels} ||C(r_j; \theta, P_i, \delta_i, \gamma_i) - I_i[j]||^2 $$

**Parameters**
*   **Positional Encoding:** 10 frequencies for position $p$, 4 frequencies for viewing direction $\vec{v}$ [S1].
*   **Latent Code Regularization:** L2 loss with weight decay $\lambda = 0.05$ [S1].
*   **Optimizer:** Adam with learning rate $lr = 0.0005$ [S1].
*   **Training Resolution:** $512 \times 512$ images [S1].
*   **Training Duration:** 400k iterations [S1].
*   **Sampling:** 64 points for coarse, 64 points for fine [S1].
*   **Ray Batch:** 2048 rays per iteration [S1].

**Pitfalls and Edge Cases**
*   **Eye Dynamics:** The morphable model does not model eye blinks or eye movements explicitly. These cannot be directly controlled, though they are implicitly correlated with other expression parameters [S1].
*   **Data Requirements:** Requires more training data than static NeRF (which needs ~100 posed images) because it must generalize over the space of expression vectors [S1].
*   **Test Time Latent Code:** At test time, the latent code from the first frame of the training set must be used, not a new random or optimized code [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class DynamicNeRF:
    def __init__(self, num_frames, num_expression_coeffs=76, latent_dim=32):
        """
        Initialize the Dynamic NeRF model.
        
        Args:
            num_frames: Number of frames in the training sequence.
            num_expression_coeffs: Dimension of the expression vector (delta-blendshapes).
            latent_dim: Dimension of the per-frame latent code.
        """
        # Backbone: 8 linear layers, 256 neurons, ReLU
        # Input to backbone: Position (encoded) + Expression + Latent Code
        # Output of backbone: Feature vector
        
        # Density Head: 1 linear layer from backbone output
        self.density_head = nn.Linear(256, 1)
        
        # Color Head: 4 linear layers, 128 neurons, ReLU
        # Input to color head: Backbone output + View Direction (encoded)
        self.color_head = nn.Sequential(
            nn.Linear(256 + 24, 128), # 24 = 4 freq * 3 dims for view dir
            nn.ReLU(),
            nn.Linear(128, 128),
            nn.ReLU(),
            nn.Linear(128, 128),
            nn.ReLU(),
            nn.Linear(128, 3) # RGB
        )
        
        # Two instances of the network: Coarse and Fine
        # Note: In practice, these are separate MLPs or the same architecture with different weights
        # For simplicity, we define one architecture and instantiate twice
        self.coarse_net = DynamicNeRFNetwork(
            density_head=self.density_head,
            color_head=self.color_head
        )
        self.fine_net = DynamicNeRFNetwork(
            density_head=self.density_head,
            color_head=self.color_head
        )
        
        # Learnable latent codes for each frame
        self.latent_codes = nn.Parameter(torch.randn(num_frames, latent_dim))
        
        # Optimizer
        self.optimizer = optim.Adam(
            list(self.coarse_net.parameters()) + 
            list(self.fine_net.parameters()) + 
            [self.latent_codes],
            lr=0.0005
        )
        
        # Background image (static)
        self.background_image = None # Shape: (H, W, 3)

    def positional_encoding(self, x, num_frequencies):
        """
        Apply positional encoding to input x.
        
        Args:
            x: Input tensor of shape (..., 3) for position or (..., 3) for view direction.
            num_frequencies: Number of frequencies to use.
            
        Returns:
            Encoded tensor of shape (..., 2 * num_frequencies * 3)
        """
        # Standard positional encoding: sin(2^i * pi * x) and cos(2^i * pi * x)
        # x shape: (..., 3)
        # freq_bands shape: (num_frequencies,)
        freq_bands = 2.0 ** torch.arange(num_frequencies, device=x.device).float()
        # Expand x to (..., 1, 3) and freq_bands to (1, num_frequencies, 1)
        x = x.unsqueeze(-2) # (..., 1, 3)
        freq_bands = freq_bands.view(1, -1, 1) # (1, num_frequencies, 1)
        
        # Compute sin and cos
        x_sin = torch.sin(x * freq_bands * torch.pi)
        x_cos = torch.cos(x * freq_bands * torch.pi)
        
        # Concatenate
        encoded = torch.cat([x_sin, x_cos], dim=-1) # (..., num_frequencies * 2, 3)
        # Flatten last two dims
        encoded = encoded.view(*x.shape[:-1], -1) # (..., num_frequencies * 6)
        return encoded

    def forward(self, position, view_dir, expression, latent_code, network):
        """
        Forward pass through the dynamic radiance field network.
        
        Args:
            position: Canonical position (x, y, z), shape (..., 3).
            view_dir: Viewing direction (x, y, z), shape (..., 3).
            expression: Expression coefficients, shape (..., 76).
            latent_code: Per-frame latent code, shape (..., 32).
            network: Either 'coarse' or 'fine' to select the appropriate MLP weights.
            
        Returns:
            density: Density values, shape (..., 1).
            color: RGB color values, shape (..., 3).
        """
        # Encode position (10 frequencies)
        pos_encoded = self.positional_encoding(position, 10)
        
        # Encode view direction (4 frequencies)
        view_encoded = self.positional_encoding(view_dir, 4)
        
        # Concatenate position, expression, and latent code for backbone input
        # Input to backbone: pos_encoded + expression + latent_code
        backbone_input = torch.cat([pos_encoded, expression, latent_code], dim=-1)
        
        # Backbone: 8 linear layers, 256 neurons, ReLU
        # We assume the backbone is part of the network's layers
        # For this pseudocode, we simulate the backbone as a sequential module
        # In the actual implementation, this would be self.coarse_net.backbone or self.fine_net.backbone
        
        # Since we can't easily separate backbone/color head in this simple class structure
        # without defining sub-modules, we will assume the network object has these layers.
        # Let's refine the class structure for clarity.
        pass

class DynamicNeRFNetwork(nn.Module):
    def __init__(self, density_head, color_head):
        super().__init__()
        # Backbone: 8 linear layers, 256 neurons, ReLU
        layers = []
        layers.append(nn.Linear(10*6 + 76 + 32, 256)) # 60 + 76 + 32 = 168
        for _ in range(7):
            layers.append(nn.ReLU())
            layers.append(nn.Linear(256, 256))
        self.backbone = nn.Sequential(*layers)
        
        self.density_head = density_head
        self.color_head = color_head

    def forward(self, position, view_dir, expression, latent_code):
        # Encode position (10 frequencies) -> 60 dims
        pos_encoded = positional_encoding(position, 10)
        
        # Encode view direction (4 frequencies) -> 24 dims
        view_encoded = positional_encoding(view_dir, 4)
        
        # Concatenate position, expression, and latent code
        backbone_input = torch.cat([pos_encoded, expression, latent_code], dim=-1)
        
        # Backbone
        backbone_output = self.backbone(backbone_input)
        
        # Density
        density = self.density_head(backbone_output)
        
        # Color: Backbone output + encoded view direction
        color_input = torch.cat([backbone_output, view_encoded], dim=-1)
        color = self.color_head(color_input)
        
        return density, color

def positional_encoding(x, num_frequencies):
    """
    Apply positional encoding to input x.
    
    Args:
        x: Input tensor of shape (..., 3).
        num_frequencies: Number of frequencies to use.
        
    Returns:
        Encoded tensor of shape (..., 2 * num_frequencies * 3)
    """
    freq_bands = 2.0 ** torch.arange(num_frequencies, device=x.device).float()
    x = x.unsqueeze(-2) # (..., 1, 3)
    freq_bands = freq_bands.view(1, -1, 1) # (1, num_frequencies, 1)
    
    x_sin = torch.sin(x * freq_bands * torch.pi)
    x_cos = torch.cos(x * freq_bands * torch.pi)
    
    encoded = torch.cat([x_sin, x_cos], dim=-1) # (..., num_frequencies * 2, 3)
    encoded = encoded.view(*x.shape[:-1], -1) # (..., num_frequencies * 6)
    return encoded

def train_step(model, frame_index, rays, ground_truth_image, background_image, pose, expression):
    """
    Perform one training step.
    
    Args:
        model: DynamicNeRF instance.
        frame_index: Index of the current frame.
        rays: List of rays, each ray is a tuple (origin, direction).
        ground_truth_image: Ground truth image for the frame, shape (H, W, 3).
        background_image: Static background image, shape (H, W, 3).
        pose: Pose matrix P_i, shape (4, 4).
        expression: Expression vector delta_i, shape (76,).
        
    Returns:
        loss: Scalar loss value.
    """
    model.optimizer.zero_grad()
    
    # Sample 2048 rays
    # 95% within head bounding box
    # Stratified sampling of 64 points along each ray for coarse network
    coarse_points, coarse_transmittances = sample_rays(rays, num_samples=64, stratified=True)
    
    # Transform points to canonical space using pose
    # canonical_points = transform_to_canonical(coarse_points, pose)
    
    # Get latent code for this frame
    latent_code = model.latent_codes[frame_index]
    
    # Expand expression and latent code to match ray batch size
    expression_expanded = expression.unsqueeze(0).expand(coarse_points.shape[0], -1)
    latent_code_expanded = latent_code.unsqueeze(0).expand(coarse_points.shape[0], -1)
    
    # Coarse network forward pass
    coarse_density, coarse_color = model.coarse_net(
        coarse_points, 
        rays.directions, 
        expression_expanded, 
        latent_code_expanded
    )
    
    # Importance sampling for fine network based on coarse density
    fine_points, fine_transmittances = importance_sample(
        coarse_points, 
        coarse_density, 
        num_samples=64
    )
    
    # Fine network forward pass
    fine_density, fine_color = model.fine_net(
        fine_points, 
        rays.directions, 
        expression_expanded, 
        latent_code_expanded
    )
    
    # Volumetric rendering for fine network
    # C(r) = integral(sigma * RGB * T) dt
    # T(t) = exp(-integral(sigma) dt)
    fine_color_rendered = volumetric_render(fine_points, fine_density, fine_color, fine_transmittances)
    
    # Background handling: Last sample point color is from background image
    # This is handled implicitly by the network learning low density for background rays
    # But explicitly, we can set the color of the last sample to background
    # However, the source says "set the color of the last sample point... to the corresponding value of the background image"
    # And "network learns to predict low density... if ray hits background pixel"
    # So we don't force the color, we let the network learn it via the background image input?
    # Actually, the source says: "The last sample on the ray r is assumed to lie on the background with a fixed color"
    # This implies we replace the last sample's color with the background color, but keep the density prediction?
    # Or do we just use the background color for the final pixel if the ray doesn't hit the foreground?
    # The source says: "Since the volumetric rendering is fully differentiable, the network picks up on this signal"
    # This suggests the background image is used as a target or a fixed value for the last sample.
    # Let's assume the last sample's color is replaced by the background color for the loss calculation.
    
    # Get background color for each ray
    # This requires mapping ray indices to pixel coordinates
    bg_colors = get_background_colors(rays, background_image)
    
    # Replace last sample color with background color
    # fine_color[-1] = bg_colors
    
    # Compute loss
    # L_i = sum || C(r_j) - I_i[j] ||^2
    loss = compute_loss(fine_color_rendered, ground_truth_image)
    
    # Add latent code regularization
    latent_loss = torch.norm(latent_code, p=2) ** 2 * 0.05
    loss = loss + latent_loss
    
    loss.backward()
    model.optimizer.step()
    
    return loss

def sample_rays(rays, num_samples, stratified=False):
    """
    Sample points along rays.
    
    Args:
        rays: List of rays.
        num_samples: Number of samples per ray.
        stratified: Whether to use stratified sampling.
        
    Returns:
        points: Sampled points, shape (num_rays, num_samples, 3).
        transmittances: Transmittance values, shape (num_rays, num_samples).
    """
    # Implementation of ray sampling
    # If stratified, add jitter to uniform samples
    # Return points and transmittances
    pass

def importance_sample(points, densities, num_samples):
    """
    Perform importance sampling based on densities.
    
    Args:
        points: Original points, shape (num_rays, num_orig_samples, 3).
        densities: Densities at original points, shape (num_rays, num_orig_samples, 1).
        num_samples: Number of new samples per ray.
        
    Returns:
        new_points: New sampled points, shape (num_rays, num_samples, 3).
        new_transmittances: Transmittance values for new points.
    """
    # Implementation of importance sampling
    pass

def volumetric_render(points, densities, colors, transmittances):
    """
    Perform volumetric rendering.
    
    Args:
        points: Sample points, shape (num_rays, num_samples, 3).
        densities: Densities, shape (num_rays, num_samples, 1).
        colors: Colors, shape (num_rays, num_samples, 3).
        transmittances: Transmittance values, shape (num_rays, num_samples).
        
    Returns:
        rendered_colors: Rendered colors, shape (num_rays, 3).
    """
    # C(r) = sum(sigma_i * color_i * T_i)
    # T_i = exp(-sum_{j<i} sigma_j * delta_j)
    # This is typically done with cumulative sum of densities
    pass

def compute_loss(rendered_colors, ground_truth_image):
    """
    Compute reconstruction loss.
    
    Args:
        rendered_colors: Rendered colors, shape (num_rays, 3).
        ground_truth_image: Ground truth image, shape (H, W, 3).
        
    Returns:
        loss: Scalar loss value.
    """
    # MSELoss between rendered colors and ground truth pixels
    pass

def get_background_colors(rays, background_image):
    """
    Get background colors for each ray.
    
    Args:
        rays: List of rays.
        background_image: Static background image, shape (H, W, 3).
        
    Returns:
        bg_colors: Background colors, shape (num_rays, 3).
    """
    # Map ray indices to pixel coordinates and lookup background image
    pass

def transform_to_canonical(points, pose):
    """
    Transform points from camera space to canonical space.
    
    Args:
        points: Points in camera space, shape (..., 3).
        pose: Pose matrix, shape (4, 4).
        
    Returns:
        canonical_points: Points in canonical space, shape (..., 3).
    """
    # Apply inverse pose transformation
    pass
```

## Operating characteristics
*   **Time and Memory Complexity:** The method uses two MLPs (coarse and fine), each with 8 backbone layers of 256 neurons and a color branch of 4 layers of 128 neurons. The memory usage is dominated by the network weights and the per-frame latent codes. The training requires 400k iterations [S1].
*   **Measured Performance:** On a dataset of monocular portrait videos, the method achieves an LPIPS of 0.06 (full training data) [S1]. With latent codes, LPIPS is 0.059, compared to 0.068 without [S1]. PSNR is 26.85, SSIM is 0.95, and L1 is 0.019 [S1].
*   **Scaling:** The method requires more training data than static NeRF. Training on half or quarter of the data significantly degrades quality (LPIPS increases to 0.07 and 0.09, respectively) [S1].
*   **Strengths:** Captures arbitrary geometry and topology such as hair, eyeware, and hats, which explicit geometry methods struggle with [S1]. Generates 3D-consistent results with view-dependent effects (e.g., glasses reflections) [S1]. Allows explicit control over head pose and expressions [S1]. Requires only a single view from a fixed camera [S1].
*   **Limitations:** Does not explicitly model eye blinks or eye movements [S1]. Requires a large training corpus to generalize over the space of expression vectors [S1]. Focuses on the human head; upper body dynamics are not modeled [S1].

## Background
Dynamic Neural Radiance Fields were introduced in the paper "Dynamic Neural Radiance Fields for Monocular 4D Facial Avatar Reconstruction" [S1]. The method builds on the static NeRF approach by Mildenhall et al. [S1]. It was developed to address the challenge of reconstructing 4D models of humans, especially the human face, for AR and VR telepresence applications [S1]. The method allows for the reconstruction of 4D facial avatars from monocular video sequences, making it suitable for capturing avatars of end users at home using only 2 minutes of their time [S1].

## Sources
[S1] Dynamic Neural Radiance Fields for Monocular 4D Facial Avatar Reconstruction, https://doi.org/10.1109/cvpr46437.2021.00854, fetched 2026-10-26
