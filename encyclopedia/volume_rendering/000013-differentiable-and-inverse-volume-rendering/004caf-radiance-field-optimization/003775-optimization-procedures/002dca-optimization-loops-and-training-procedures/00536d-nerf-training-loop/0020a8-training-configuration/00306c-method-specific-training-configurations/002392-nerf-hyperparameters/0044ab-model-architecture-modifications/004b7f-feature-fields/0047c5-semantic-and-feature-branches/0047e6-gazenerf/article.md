# GazeNeRF

Also called: GazeNeRF: 3D-Aware Gaze Redirection with Neural Radiance Fields

## Overview
GazeNeRF is a 3D-aware method for gaze redirection that reformulates the task as a neural volume rendering problem [S1]. It solves the limitation of existing 2D methods by explicitly modeling the rigid 3D rotation of eyeballs and the non-rigid deformation of the face separately [S1]. Unlike sibling methods that mix facial features or lack 3D consistency, GazeNeRF uses a two-stream MLP architecture to predict volumetric features for the face (excluding eyes) and eyes independently, applying a rigid 3D rotation matrix to the eye features before compositing [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A reference face image $I_{ref}$ (resized to $512 \times 512$ pixels), a target gaze label $g$ (2D vector of pitch and yaw angles in radians), and latent codes $z^*$ initialized from 3D Morphable Model (3DMM) parameters [S1].
*   **Outputs:** A rendered 2D image $I_{wf}$ of the whole face with redirected gaze, along with intermediate images $I_{fw/o}$ (face-only) and $I_e$ (eyes-only) [S1].

**Data Structures:**
1.  **Two-Stream MLPs:**
    *   $h_{\theta_{fw/o}}$: Face-only stream. Parameters $\theta_{fw/o}$.
    *   $h_{\theta_{e}}$: Eye stream. Parameters $\theta_{e}$ [S1].
    *   Both map $(\gamma(\mathbf{x}), \gamma(\mathbf{d}), z_{sh}, z_{ex}, z_{te}, z_{il}, g) \rightarrow (\sigma, \mathbf{f})$ [S1].
    *   $\gamma$ is positional encoding [S1].
    *   $z_{sh}, z_{ex}, z_{te}, z_{il}$ are latent codes for shape, expression, texture, and illumination [S1].
    *   $\mathbf{f} \in \mathbb{R}^{258}$ is the feature vector [S1].
2.  **Volume Feature Maps:**
    *   $F_{fw/o} \in \mathbb{R}^{64 \times 64 \times 258}$: Rendered feature map for the face-only region [S1].
    *   $F_{e} \in \mathbb{R}^{64 \times 64 \times 258}$: Rendered feature map for the eyes region [S1].
3.  **Rotation Matrix:**
    *   $R \in \mathbb{R}^{3 \times 3}$: 3D rotation matrix computed from gaze label $g$ [S1].
4.  **Neural Rendering Module:**
    *   Shared weights $\theta_{render}$ for upsampling and color prediction [S1].

**Numbered Steps:**

1.  **Encode Inputs:** Apply positional encoding $\gamma$ to 3D position $\mathbf{x}$ and view direction $\mathbf{d}$. Concatenate $\gamma(\mathbf{x})$ with latent codes $z_{sh}, z_{ex}, z_{te}, z_{il}$ and gaze label $g$ [S1].
2.  **Face Stream Processing:** Pass encoded inputs through $h_{\theta_{fw/o}}$ to predict volume density $\sigma_{fw/o}$ and feature vector $\mathbf{f}_{fw/o}$ for the face region (excluding eyes) [S1].
3.  **Eye Stream Processing:** Pass encoded inputs through $h_{\theta_{e}}$ to predict volume density $\sigma_{e}$ and feature vector $\mathbf{f}_{e}$ for the eye region [S1].
4.  **Volume Rendering:** Use differentiable volume rendering on the outputs of both MLPs to generate low-resolution volume feature maps $F_{fw/o}$ and $F_{e}$ with dimensions $\mathbb{R}^{64 \times 64 \times 258}$ [S1].
    *   The rendering follows the standard NeRF integration:
        $$ \hat{\mathbf{F}}(r) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{f}(x_i) $$
        where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ and $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ [S1, Parent].
5.  **3D Rotation:** Reshape eye feature map $F_{e}$ from $\mathbb{R}^{64 \times 64 \times 258}$ to $\mathbb{R}^{64 \times 64 \times 86 \times 3}$. Apply rigid rotation using the gaze-derived matrix $R$:
    $$ F_{e-rot} = R F_{e} $$
    where $F_{e-rot} \in \mathbb{R}^{64 \times 64 \times 86 \times 3}$ [S1].
6.  **Feature Composition:** Map the rotated eye feature map $F_{e-rot}$ to the segmented eyes region and the face feature map $F_{fw/o}$ to the segmented face-only region. Merge them using element-wise maximum:
    $$ F_{wf} = \max(F_{fw/o}, F_{e-rot}) $$
    [S1].
7.  **Final Rendering:** Use the neural rendering module with shared weights $\theta_{render}$ to render three images:
    *   $I_{fw/o}$ from $F_{fw/o}$
    *   $I_e$ from $F_{e-rot}$
    *   $I_{wf}$ from $F_{wf}$ [S1].

**Loss Functions:**
The total loss is:
$$ L_{Overall} = \lambda_R L_R + \lambda_P L_P + \lambda_F L_F + \lambda_D L_D $$
where $\lambda_R = \lambda_P = \lambda_F = \lambda_D = 1$ [S1].

1.  **Reconstruction Loss ($L_R$):**
    $$ L_{R_{whole face}}(I_{wf}, I_{gt}) = \frac{1}{|M_{wf}|} ||M_{wf}(I_{wf} - I_{gt})||_1 $$
    where $M_{wf}$ is the whole face mask [S1].
    $$ L_R = L_{R_{whole face}} + L_{R_{face only}} + L_{R_{eyes}} $$
    using masks $M_{f}$ and $M_{e}$ for the respective regions [S1].

2.  **Perceptual Loss ($L_P$):**
    $$ L_{P_{whole face}} = \sum_{i} \frac{1}{||\phi_i(I_{gt})||} ||\phi_i(I_{wf}) - \phi_i(I_{gt})||_1 $$
    where $\phi_i$ is the $i$-th layer of VGG16 [S1].
    $$ L_P = L_{P_{whole face}} + L_{P_{face only}} + L_{P_{eyes}} $$
    [S1].

3.  **Functional Loss ($L_F$):**
    $$ L_{F_{content}}(I_{wf}, I_{gt}) = E_{ang}(\psi_g(I_{wf}), \psi_g(I_{gt})) $$
    where $\psi_g$ is a gaze estimator network and $E_{ang}$ is angular error:
    $$ E_{ang}(v, \tilde{v}) = \arccos \frac{v \cdot \tilde{v}}{|v| |\tilde{v}|} $$
    [S1].
    $$ L_F = \lambda_{F_{content}} L_{F_{content}} $$
    where $\lambda_{F_{content}}$ starts at $1 \times 10^{-3}$ and increases by $1 \times 10^{-3}$ after each epoch [S1].

4.  **Disentanglement Loss ($L_D$):**
    $$ L_D = \sum_{w^*} \frac{w^*}{|z^*_0|} ||z^* - z^*_0||_2^2 $$
    where $z^*$ are learned latent codes, $z^*_0$ are initial 3DMM values, and $w^*$ are weights [S1].
    Weights: $w_{sh} = w_{te} = w_{il} = 1 \times 10^{-3}$ and $w_{ex} = 1.0$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class GazeNeRF:
    def __init__(self, latent_codes_init, gaze_label, face_mask, eyes_mask):
        """
        Initialize GazeNeRF.
        :param latent_codes_init: Dict with keys 'sh', 'ex', 'te', 'il' (shape, exp, tex, ill)
        :param gaze_label: Tensor of shape (2,) containing pitch and yaw in radians
        :param face_mask: Binary mask for face-only region
        :param eyes_mask: Binary mask for eyes region
        """
        # Two-stream MLPs
        self.mlp_face = MLP_stream() # h_theta_fw/o
        self.mlp_eye = MLP_stream()  # h_theta_e
        
        # Neural rendering module with shared weights
        self.render_module = NeuralRenderingModule()
        
        # Learnable latent codes initialized from 3DMM
        self.z_sh = nn.Parameter(latent_codes_init['sh'])
        self.z_ex = nn.Parameter(latent_codes_init['ex'])
        self.z_te = nn.Parameter(latent_codes_init['te'])
        self.z_il = nn.Parameter(latent_codes_init['il'])
        
        self.face_mask = face_mask
        self.eyes_mask = eyes_mask
        self.gaze_label = gaze_label
        
        # Loss weights
        self.lambda_R = 1.0
        self.lambda_P = 1.0
        self.lambda_D = 1.0
        self.lambda_F_content = 1e-3
        
    def get_rotation_matrix(self, gaze_label):
        """Compute 3D rotation matrix R from gaze label (pitch, yaw)."""
        pitch, yaw = gaze_label
        # Implementation details for R depend on specific coordinate system conventions.
        # Generally involves rotation matrices around X and Y axes.
        # Placeholder for specific rotation logic derived from gaze label g [S1].
        R = compute_rotation_from_pitch_yaw(pitch, yaw)
        return R

    def forward(self, rays_x, rays_d):
        """
        Forward pass for a batch of rays.
        :param rays_x: 3D positions, shape (N, 3)
        :param rays_d: View directions, shape (N, 3)
        :return: Rendered images I_wf, I_fw/o, I_e
        """
        # 1. Encode Inputs
        gamma_x = positional_encoding(rays_x)
        gamma_d = positional_encoding(rays_d)
        
        # Concatenate with latent codes and gaze label
        # Note: Latent codes are broadcasted or repeated to match batch size if necessary
        input_face = torch.cat([gamma_x, gamma_d, self.z_sh, self.z_ex, self.z_te, self.z_il, self.gaze_label], dim=-1)
        input_eye = input_face.clone() # Same inputs for both streams
        
        # 2. Face Stream Processing
        sigma_fw_o, f_fw_o = self.mlp_face(input_face)
        
        # 3. Eye Stream Processing
        sigma_e, f_e = self.mlp_eye(input_eye)
        
        # 4. Volume Rendering to get feature maps
        # Render to 64x64 grid. This step aggregates ray samples into feature maps.
        # F_fw/o and F_e are in R^(64, 64, 258)
        F_fw_o = volume_render_feature_map(sigma_fw_o, f_fw_o, rays_x, rays_d)
        F_e = volume_render_feature_map(sigma_e, f_e, rays_x, rays_d)
        
        # 5. 3D Rotation
        R = self.get_rotation_matrix(self.gaze_label)
        # Reshape F_e to (64, 64, 86, 3) for rotation
        F_e_reshaped = F_e.view(64, 64, 86, 3)
        # Apply rotation: F_e_rot = R * F_e
        # Matrix multiplication over the last dimension
        F_e_rot_reshaped = torch.matmul(F_e_reshaped, R.T) # Assuming R is 3x3
        F_e_rot = F_e_rot_reshaped.view(64, 64, 258)
        
        # 6. Feature Composition
        # Map to segmented regions and merge
        # F_fw/o is mapped to face-only region, F_e_rot to eyes region
        # Merge using element-wise max
        F_wf = torch.maximum(F_fw_o, F_e_rot)
        
        # 7. Final Rendering
        # Render images from feature maps using shared neural rendering module
        I_fw_o = self.render_module.render(F_fw_o)
        I_e = self.render_module.render(F_e_rot)
        I_wf = self.render_module.render(F_wf)
        
        return I_wf, I_fw_o, I_e

    def compute_loss(self, I_wf, I_fw_o, I_e, I_gt):
        """
        Compute total loss.
        :param I_wf: Rendered whole face image
        :param I_fw_o: Rendered face-only image
        :param I_e: Rendered eyes image
        :param I_gt: Ground truth image
        :return: Total loss scalar
        """
        # Reconstruction Loss
        L_R_wf = masked_l1_loss(I_wf, I_gt, self.face_mask) # Using whole face mask M_wf
        L_R_fw_o = masked_l1_loss(I_fw_o, I_gt, self.face_mask) # Using face mask M_f
        L_R_e = masked_l1_loss(I_e, I_gt, self.eyes_mask) # Using eyes mask M_e
        L_R = L_R_wf + L_R_fw_o + L_R_e
        
        # Perceptual Loss (VGG16)
        L_P_wf = perceptual_loss(I_wf, I_gt)
        L_P_fw_o = perceptual_loss(I_fw_o, I_gt)
        L_P_e = perceptual_loss(I_e, I_gt)
        L_P = L_P_wf + L_P_fw_o + L_P_e
        
        # Functional Loss
        # Estimate gaze from rendered and ground truth images
        est_gaze_wf = gaze_estimator(I_wf)
        est_gaze_gt = gaze_estimator(I_gt)
        L_F_content = angular_error(est_gaze_wf, est_gaze_gt)
        L_F = self.lambda_F_content * L_F_content
        
        # Disentanglement Loss
        L_D = 0
        weights = {'sh': 1e-3, 'ex': 1.0, 'te': 1e-3, 'il': 1e-3}
        for key, w in weights.items():
            z_star = getattr(self, f'z_{key}')
            z_0 = self.latent_codes_init[key]
            L_D += w * torch.mean((z_star - z_0)**2)
            
        L_Overall = self.lambda_R * L_R + self.lambda_P * L_P + L_F + self.lambda_D * L_D
        return L_Overall

def masked_l1_loss(pred, target, mask):
    """Compute L1 loss with mask."""
    masked_diff = mask * (pred - target)
    return torch.mean(torch.abs(masked_diff)) / torch.mean(mask)

def perceptual_loss(pred, target):
    """Compute perceptual loss using VGG16 features."""
    loss = 0
    vgg = get_vgg16()
    vgg.eval()
    with torch.no_grad():
        pred_features = vgg(pred)
        target_features = vgg(target)
    for p_feat, t_feat in zip(pred_features, target_features):
        loss += torch.mean(torch.abs(p_feat - t_feat)) / torch.mean(torch.abs(t_feat))
    return loss

def angular_error(v1, v2):
    """Compute angular error between two gaze vectors."""
    dot_product = torch.sum(v1 * v2, dim=-1)
    norm1 = torch.norm(v1, dim=-1)
    norm2 = torch.norm(v2, dim=-1)
    cos_theta = dot_product / (norm1 * norm2 + 1e-8)
    cos_theta = torch.clamp(cos_theta, -1.0, 1.0)
    return torch.acos(cos_theta)
```

## Operating characteristics
*   **Time Complexity:** Training involves optimizing two MLPs and latent codes via Adam optimizer. The explicit 3D rotation and feature compositing add minimal overhead to the standard NeRF volume rendering pipeline [S1].
*   **Memory Complexity:** Requires storing two sets of MLP parameters and four latent codes per subject. Feature maps are low-resolution ($64 \times 64 \times 258$) [S1].
*   **Performance:** On the ETH-XGaze dataset, GazeNeRF achieves superior gaze redirection accuracy (lower gaze error) compared to STED and HeadNeRF, while maintaining comparable image quality (SSIM, PSNR, LPIPS) [S1].
*   **Strengths:** Explicit 3D awareness of eyeball rotation; disentangled features prevent identity loss and blurry eyes; high-fidelity rendering [S1].
*   **Limitations:** Requires face parsing masks for training; relies on 3DMM initialization for latent codes; 2D gaze label input limits head pose control compared to full 3D head pose conditioning [S1].
*   **When to use:** When 3D-consistent gaze redirection is required, especially for applications needing accurate eye rotation without distorting the rest of the face [S1].
*   **When not to use:** When high-fidelity relighting is the primary goal (EyeNeRF is better suited) or when 2D-only processing is sufficient and 3D awareness is not critical [S1].

## Background
GazeNeRF was proposed in 2023 by the authors of "GazeNeRF: 3D-Aware Gaze Redirection with Neural Radiance Fields" [S1]. It builds on the intuition that the face and eyeballs are separate 3D structures that move independently [S1]. It differs from 2D methods like STED and FAZE, which apply weak rotation priors to mixed 2D latent features, and from EyeNeRF, which focuses on high-fidelity rendering rather than gaze redirection accuracy [S1]. It also improves upon HeadNeRF by separating the eyes into a distinct stream for explicit 3D rotation [S1]. The method was evaluated on the ETH-XGaze, ColumbiaGaze, MPIIFaceGaze, and GazeCapture datasets [S1].

## Sources
[S1] GazeNeRF: 3D-Aware Gaze Redirection with Neural Radiance Fields, https://doi.org/10.1109/CVPR52729.2023.00933, fetched 2026-10-29
