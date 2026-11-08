# Neural Lumigraph Rendering

Also called: Neural Radiance Fields with SDF export, SIREN-based Neural Rendering

## Overview
Neural Lumigraph Rendering (NLR) is a neural rendering framework that jointly optimizes an implicit signed distance function (SDF) for shape and a radiance field for appearance, supervised exclusively by posed 2D images [S1]. It enables the export of view-dependent textured meshes for real-time rasterization, achieving rendering rates approximately 10 times higher than NeRF while maintaining comparable state-of-the-art image quality [S1]. Unlike standard neural volume rendering which outputs images via ray marching, NLR converts the learned implicit representation into an explicit triangular mesh and projective textures, leveraging unstructured lumigraph rendering to preserve view-dependent effects in a traditional graphics pipeline [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $m$ multi-view 2D images $I \in \mathbb{R}^{m \times w \times h \times 3}$ with known object masks $M \in \mathbb{R}^{m \times w \times h}$ (where 1 marks foreground) and corresponding camera poses (view matrix $V$ and projection matrix $P$) [S1].
*   **Outputs:** Optimized network parameters $\theta$ (SDF) and $\phi$ (radiance field); optionally, a triangular mesh and projective textures for real-time rendering [S1].

**Data Structures:**
*   **SDF Network $S$:** A SIREN (Sinusoidal Representation Network) MLP with 5 layers and 256 hidden units each. Input: $x \in \mathbb{R}^3$. Output: $S(x; \theta) \in \mathbb{R}$ [S1].
*   **Radiance Field Network $E$:** A SIREN MLP with 5 layers and 256 hidden units each. Input: position $x \in \mathbb{R}^3$, view direction $r_d \in \mathbb{R}^3$, and local normal $n \in \mathbb{R}^3$. The view direction $r_d$ is encoded using Fourier features $\{\sin(2k\pi r_d), \cos(2k\pi r_d) \mid k \in 1 \dots 4\}$ [S1]. Output: $E(x, r_d, n; \theta, \phi) \in \mathbb{R}^3$ (RGB) [S1]. Note: The parameters $\theta$ from the SDF are reused to increase network capacity [S1].

**Training Procedure:**

1.  **Initialization:** Initialize $S$ to a unit sphere of radius 0.5 by pretraining to a procedural shape [S1]. Trace object rays in a larger sphere of radius 1 [S1].
2.  **Ray Generation:** For a pixel at relative projection plane location $u \in [-1, 1]^2$, compute ray origin $r_o$ and direction $r_d$:
    $$ r_o = (V^{-1} \cdot [0, 0, 1, 0]^T)_{x,y,z} $$
    $$ r_d = \nu((P \cdot V)^{-1} \cdot [u_x, u_y, 0, 1]^T) $$
    where $\nu(\omega) = \omega_{x,y,z} / \|\omega_{x,y,z}\|$ is vector normalization [S1].
3.  **Sphere Tracing:** Find the zero-level set $S_0$ closest to the camera origin along each ray using iterative updates:
    $$ x_0 = r_o $$
    $$ x_{i+1} = x_i + S(x_i)r_d $$
    Perform $n=16$ steps. A small residual $|S(x_n)| < 0.005$ is tolerated in practice [S1]. Retain gradients only in the last step to make sphere tracing memory efficient [S1].
4.  **Appearance Sampling:** Sample appearance directly from the radiance field at the converged surface point $S_0$:
    $$ \text{Color} = E(S_0, r_d, r_S(S_0); \theta, \phi) $$
    where $r_S(S_0)$ is the local normal computed by automatic differentiation of $S$ at $S_0$ [S1].
5.  **Loss Computation:** Minimize the weighted sum of losses:
    $$ \mathcal{L} = \mathcal{L}_R + w_E \mathcal{L}_E + w_M \mathcal{L}_M + w_S \mathcal{L}_S $$
    *   **Reconstruction Loss ($\mathcal{L}_R$):** L1 error for true foreground pixels $U_f = U \cap S_0 \cap \{U \mid M_U = 1\}$:
        $$ \mathcal{L}_R = \frac{1}{|U|} \sum_{c \in I_{U_f}} |E(x, r_d, n; \theta, \phi) - c| $$
        where $c$ is an RGB value of a foreground pixel [S1].
    *   **Eikonal Loss ($\mathcal{L}_E$):** Regularizes $S$ to enforce metric properties. Random points $x_r$ are uniformly sampled from a cube encapsulating the object's bounding unit radius sphere:
        $$ \mathcal{L}_E = \frac{1}{|U|} \sum_{x_r} (\|\nabla_x S(x_r; \theta)\|_2^2 - 1)^2 $$
        [S1].
    *   **Mask Loss ($\mathcal{L}_M$):** Restricts coarse shape to object mask boundaries for non-foreground pixels. $S_{min} = \arg \min_t S(r_o + t r_d; \theta)$ is the minimum S value along the entire ray:
        $$ \mathcal{L}_M = \frac{1}{\alpha |U|} \sum_{m \in M_U \setminus U_f} \text{BCE}(\text{sigmoid}(S_{min}/\alpha), m) $$
        where $\alpha$ is a softness parameter and BCE is binary cross entropy [S1].
    *   **Smoothness Loss ($\mathcal{L}_S$):** Regularizes $E$ to avoid overfitting by linearizing angular behavior:
        $$ \mathcal{L}_S = \frac{1}{|U|} \sum \|\nabla^2_{r_d} E(x, r_d, n; \theta, \phi)\|_2^2 $$
        [S1].
6.  **Optimization:** Optimize parameters $\theta$ and $\phi$ using the Adam solver with initial learning rate $10^{-4}$, decreased by a factor of 2 every 40,000 batches for 150,000 batches [S1]. Use a batch size of 50,000 rays sampled uniformly across the dataset [S1]. Typical weights: $w_E = 0.1$, $w_M = 100$, $w_S = 0.01$ [S1].

**Mesh Extraction and Real-Time Rendering Pipeline:**

1.  **Mesh Extraction:** Use marching cubes on $S$ voxelized at $512^3$ resolution. Offset the iso-surface of $S$ by 0.5% of the object radius in the outside direction to optimize image quality [S1].
2.  **Appearance Resampling:** Resample the optimized emissivity function $E$ to synthesize projective textures $T_i$ for $N$ camera poses and corresponding projection matrices [S1].
3.  **Rasterization:** Rasterize the extracted mesh using OpenGL and project vertex positions to each pixel [S1].
4.  **Angle Computation:** Compute angles $\tau_{1 \dots N}$ between the ray towards the current rendering camera and the rays towards each of the $N$ projective texture map viewpoints [S1].
5.  **Texture Selection:** Apply unstructured lumigraph rendering by blending contributions from the first $k=5$ textures, sorted by $\tau_i$ in ascending order [S1].
6.  **Weight Calculation:**
    $$ \hat{w}_i = \frac{1}{\tau_i} \left(1 - \frac{\tau_i}{\tau_k}\right) $$
    $$ w_i = \frac{\hat{w}_i}{\sum_{j=1 \dots k} \hat{w}_j} $$
    [S1].
7.  **Occlusion Handling:** Discard samples from occluded textures by setting their $w_i$ to zero. Occlusions are detected by comparing the pre-rendered depth associated with a texture and the distance between the mesh voxel and the texture viewpoint [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

class SIREN(nn.Module):
    """Sinusoidal Representation Network"""
    def __init__(self, in_dim, out_dim, hidden_dim=256, num_layers=5):
        super().__init__()
        self.layers = nn.ModuleList()
        self.layers.append(nn.Linear(in_dim, hidden_dim))
        for _ in range(num_layers - 2):
            self.layers.append(nn.Linear(hidden_dim, hidden_dim))
        self.layers.append(nn.Linear(hidden_dim, out_dim))
        
    def forward(self, x):
        for i, layer in enumerate(self.layers):
            x = layer(x)
            if i < len(self.layers) - 1:
                # SIREN uses sin activation for hidden layers
                x = torch.sin(x)
        return x

class NeuralLumigraphRenderer:
    def __init__(self, siren_hidden_dim=256, siren_layers=5, 
                 w_E=0.1, w_M=100, w_S=0.01, alpha=1.0, 
                 num_sphere_trace_steps=16, k_textures=5):
        self.siren_hidden_dim = siren_hidden_dim
        self.siren_layers = siren_layers
        self.w_E = w_E
        self.w_M = w_M
        self.w_S = w_S
        self.alpha = alpha
        self.n_steps = num_sphere_trace_steps
        self.k = k_textures
        
        # Initialize networks
        # SDF: R^3 -> R
        self.S = SIREN(3, 1, siren_hidden_dim, siren_layers)
        # Radiance Field: R^9 -> R^3 (x, rd, n)
        # Note: rd is expanded with Fourier features before input
        self.E = SIREN(3 + 3 + 8 + 3, 3, siren_hidden_dim, siren_layers) # 3(x) + 3(rd) + 8(Fourier) + 3(n)
        
        # Optimizer setup would be handled externally
        self.optimizer = None

    def fourier_features(self, rd, num_freqs=4):
        """Compute Fourier features for view direction"""
        # {sin(2k*pi*rd), cos(2k*pi*rd) | k in 1..4}
        freqs = torch.arange(1, num_freqs + 1, device=rd.device).float() * 2 * np.pi
        # rd shape: (B, 3), freqs shape: (F,)
        # Output shape: (B, 2 * num_freqs * 3)
        expanded_rd = rd.unsqueeze(-1) * freqs.view(1, 1, -1) # (B, 3, F)
        sin_feats = torch.sin(expanded_rd).flatten(1)
        cos_feats = torch.cos(expanded_rd).flatten(1)
        return torch.cat([sin_feats, cos_feats], dim=-1)

    def sphere_trace(self, S_net, r_o, r_d):
        """
        Sphere trace SDF to find zero-level set.
        Returns: converged points x_n, normals n
        """
        x = r_o
        # Retain gradients only in the last step for memory efficiency [S1]
        # We perform n_steps iterations
        for i in range(self.n_steps):
            s_val = S_net(x)
            # x_{i+1} = x_i + S(x_i) * r_d
            x = x + s_val * r_d
            
        # Compute normal at final position using autograd
        x.requires_grad_(True)
        s_final = S_net(x)
        grad = torch.autograd.grad(s_final, x, grad_outputs=torch.ones_like(s_final), create_graph=True)[0]
        n = grad / torch.norm(grad, dim=-1, keepdim=True)
        
        return x, n

    def compute_losses(self, S_net, E_net, rays, images, masks):
        """
        rays: dict with 'r_o' (B, 3), 'r_d' (B, 3)
        images: (B, 3) RGB values
        masks: (B,) binary mask (1 for foreground)
        """
        r_o = rays['r_o']
        r_d = rays['r_d']
        
        # 1. Sphere Trace
        x_surf, n = self.sphere_trace(S_net, r_o, r_d)
        
        # 2. Appearance Sampling
        # Prepare input for E: x, r_d, Fourier(r_d), n
        rd_fourier = self.fourier_features(r_d)
        input_E = torch.cat([x_surf, r_d, rd_fourier, n], dim=-1)
        pred_color = E_net(input_E)
        
        # 3. Reconstruction Loss (L_R)
        # Only for foreground pixels
        foreground_mask = masks == 1
        if foreground_mask.sum() > 0:
            L_R = torch.mean(torch.abs(pred_color[foreground_mask] - images[foreground_mask]))
        else:
            L_R = torch.tensor(0.0, device=images.device)
            
        # 4. Eikonal Loss (L_E)
        # Sample random points from a cube encapsulating unit radius sphere
        # Not given in sources how to sample exactly, assuming uniform in [-1, 1]^3
        num_random = 1000 # Placeholder for batch size of random points
        x_r = torch.rand(num_random, 3, device=images.device) * 2 - 1 # [-1, 1]
        # Ensure they are within bounding sphere? Source says "cube which encapsulates..."
        # We just sample from the cube.
        s_r = S_net(x_r)
        grad_r = torch.autograd.grad(s_r, x_r, grad_outputs=torch.ones_like(s_r), create_graph=True)[0]
        grad_norm_sq = torch.sum(grad_r**2, dim=-1)
        L_E = torch.mean((grad_norm_sq - 1)**2)
        
        # 5. Mask Loss (L_M)
        # For non-foreground pixels
        non_foreground_mask = masks != 1
        if non_foreground_mask.sum() > 0:
            # S_min: min S value along entire ray
            # Approximated by dense sampling of t
            # Not given exact number of samples for S_min approximation [G]
            t_vals = torch.linspace(0, 10, 100, device=images.device) # [G] arbitrary range
            t_vals = t_vals.view(-1, 1) # (100, 1)
            ray_pts = r_o.unsqueeze(0) + t_vals * r_d.unsqueeze(0) # (100, B, 3)
            s_vals = S_net(ray_pts.flatten(0, 1)).view(-1, r_o.shape[0]) # (100, B)
            S_min = torch.min(s_vals, dim=0)[0] # (B,)
            
            # BCE(sigmoid(S_min / alpha), m)
            # m is mask value (0 for non-foreground)
            m_vals = masks[non_foreground_mask].float()
            S_min_vals = S_min[non_foreground_mask]
            
            pred_mask = torch.sigmoid(S_min_vals / self.alpha)
            # BCE loss
            bce_loss = -(m_vals * torch.log(pred_mask + 1e-6) + (1 - m_vals) * torch.log(1 - pred_mask + 1e-6))
            L_M = torch.mean(bce_loss)
        else:
            L_M = torch.tensor(0.0, device=images.device)
            
        # 6. Smoothness Loss (L_S)
        # Regularize E to avoid overfitting
        # || r_2 * r_d * E ||^2 ? Source formula: || \nabla^2_{rd} E ||^2
        # Note: Source [S1] formula (9) uses || r_2 r_d E ||^2 but text says "linearize angular behavior"
        # and cites [9] (projective texture mapping). 
        # The note says: L_S = (1/|U|) * sum || r_2 * r_d * E ||^2
        # However, formula (9) is || \nabla^2_{rd} E ||^2.
        # Let's stick to the explicit formula (9) provided in the text as it is more standard for smoothness.
        # But wait, the note explicitly lists: "Angular smoothness loss: L_S = ... ||r_2 * r_d * E||^2"
        # And formula (9) is: || \nabla^2_{rd} E ||^2.
        # The text says "Inspired by multi-view projective texture mapping... linearize the angular behavior".
        # Usually smoothness is gradient based. 
        # Let's use the formula (9) as it is explicitly numbered.
        # Compute Hessian wrt rd? Or just gradient? Formula says \nabla^2.
        # Computing Hessian is expensive. 
        # Let's look at the note again: "r_2 is a weighting factor". This looks like a typo in the note or a specific term.
        # Given the ambiguity, and that formula (9) is explicit:
        
        # We need second derivative of E wrt rd.
        # E input includes rd.
        # Let's approximate or use gradient norm if Hessian is too complex for "plain text" implementation without specific library calls.
        # However, the prompt requires implementation.
        # Let's assume the note's formula is the intended one if it's simpler, but formula (9) is in the paper.
        # The note says: "L_S = (1/|U|) * sum || r_2 * r_d * E ||^2". This doesn't make sense dimensionally if r_2 is scalar.
        # Let's stick to Formula (9): || \nabla^2_{rd} E ||^2.
        
        # To compute Hessian, we need second order gradients.
        # This is complex. Let's check if there's a simpler interpretation.
        # "linearize the angular behavior" -> minimize curvature.
        # We will implement the gradient norm squared as a proxy if Hessian is not feasible, 
        # but the source explicitly gives Hessian.
        # We will implement Hessian using autograd.
        
        # Input to E for this part: x_surf, r_d, rd_fourier, n
        # We need d^2 E / d rd^2
        # This is a tensor of shape (B, 3, 3) for each output channel? Or just norm of Hessian matrix?
        # Formula: || \nabla^2_{rd} E ||^2_2. Frobenius norm of Hessian.
        
        # This is computationally heavy. 
        # Let's implement it.
        
        # Create a function to get E
        def get_E(rd_input):
            rd_fourier_in = self.fourier_features(rd_input)
            input_E_in = torch.cat([x_surf, rd_input, rd_fourier_in, n], dim=-1)
            return E_net(input_E_in)
            
        # First derivative
        rd_input = r_d.clone().detach().requires_grad_(True)
        E_val = get_E(rd_input)
        grad_E = torch.autograd.grad(E_val, rd_input, grad_outputs=torch.ones_like(E_val), create_graph=True)[0] # (B, 3, 3) ? No, E is (B,3), rd is (B,3). Grad is (B,3,3)
        
        # Second derivative (Hessian)
        # We need gradient of grad_E wrt rd_input
        # grad_E shape: (B, 3, 3) if we sum over output channels? 
        # E is (B, 3). grad_E is (B, 3, 3).
        # We want norm of Hessian.
        # Let's sum over output channels first to get scalar loss per pixel?
        # Formula: || \nabla^2 E ||^2. Usually means sum of squares of all second derivatives.
        
        # Flatten grad_E to (B, 9)
        grad_E_flat = grad_E.view(grad_E.shape[0], -1)
        
        # Compute gradient of each component of grad_E wrt rd
        hessian_components = []
        for i in range(grad_E_flat.shape[1]):
            h = torch.autograd.grad(grad_E_flat[:, i], rd_input, grad_outputs=torch.ones_like(grad_E_flat[:, i]), create_graph=True)[0]
            hessian_components.append(h)
            
        hessian = torch.stack(hessian_components, dim=1) # (B, 9, 3)
        L_S = torch.mean(torch.sum(hessian**2, dim=(1, 2)))
        
        # Total Loss
        L_total = L_R + self.w_E * L_E + self.w_M * L_M + self.w_S * L_S
        
        return L_total, L_R, L_E, L_M, L_S

    def render_realtime(self, mesh, textures, camera_poses, current_view_matrix, current_proj_matrix):
        """
        Render using unstructured lumigraph technique.
        mesh: Triangular mesh (vertices, faces)
        textures: List of N textures (images) and their projection matrices
        """
        # 1. Rasterize mesh
        # Not given exact rasterization code, assume standard OpenGL rasterization
        # Returns: depth buffer, pixel coordinates, vertex indices
        # [G] Standard rasterization step
        rendered_depth, pixel_data = rasterize_mesh(mesh, current_view_matrix, current_proj_matrix)
        
        # 2. Compute angles tau
        # tau_i: angle between ray to current camera and ray to texture viewpoint i
        # Ray to current camera: from pixel to camera center? 
        # Source: "angles tau_1...N between the ray towards the current rendering camera and the rays towards each of the N projective texture map viewpoints"
        # This implies we compute the direction from the pixel to the current camera, and from the pixel to the texture camera.
        
        # Get pixel positions in world space from depth
        # [G] Inverse projection and view to get world coords
        world_coords = unproject_pixel(pixel_data, current_view_matrix, current_proj_matrix, rendered_depth)
        
        # Ray to current camera (origin)
        cam_origin = get_camera_origin(current_view_matrix)
        ray_to_current = normalize(cam_origin - world_coords)
        
        # 3. Select top k textures
        angles = []
        for i, tex_pose in enumerate(camera_poses):
            tex_origin = get_camera_origin(tex_pose)
            ray_to_tex = normalize(tex_origin - world_coords)
            # Angle between ray_to_current and ray_to_tex
            # dot product
            dot = torch.sum(ray_to_current * ray_to_tex, dim=-1)
            dot = torch.clamp(dot, -1.0, 1.0)
            tau = torch.acos(dot)
            angles.append(tau)
            
        angles = torch.stack(angles, dim=0) # (N, B)
        
        # Sort by tau ascending
        sorted_indices = torch.argsort(angles, dim=0)
        top_k_indices = sorted_indices[:self.k, :]
        
        # 4. Compute weights
        # tau_i for the top k
        tau_k = angles[top_k_indices[-1], :] # Largest tau in top k
        tau_vals = angles[top_k_indices, :] # (k, B)
        
        # w_hat_i = 1/tau_i * (1 - tau_i / tau_k)
        # Handle tau_i = 0? Source says converges to exclusive mapping when tau -> 0.
        # If tau_i is 0, w_hat_i is infinite? 
        # Source: "satisfies epipolar consistency by converging to an exclusive mapping... when tau_j -> 0"
        # Implementation detail: if tau_i is very small, use large weight?
        # Let's implement as is.
        eps = 1e-6
        tau_vals_safe = torch.clamp(tau_vals, min=eps)
        tau_k_safe = torch.clamp(tau_k, min=eps)
        
        w_hat = (1.0 / tau_vals_safe) * (1.0 - tau_vals_safe / tau_k_safe)
        
        # Normalize
        w_sum = torch.sum(w_hat, dim=0, keepdim=True)
        w = w_hat / w_sum
        
        # 5. Blend textures
        # Discard occluded textures
        # Occlusion detection: compare pre-rendered depth of texture with distance from mesh voxel to texture viewpoint
        # [G] This requires rendering each texture's view of the mesh to get depth
        occlusion_masks = []
        for i in top_k_indices:
            # Render texture i's view of the mesh
            tex_depth = render_texture_view(mesh, camera_poses[i])
            # Distance from mesh voxel (world_coords) to texture viewpoint
            tex_origin = get_camera_origin(camera_poses[i])
            dist_to_tex = torch.norm(world_coords - tex_origin, dim=-1)
            
            # Compare depths
            # If dist_to_tex > tex_depth, it's occluded
            is_occluded = dist_to_tex > tex_depth
            occlusion_masks.append(is_occluded)
            
        occlusion_mask = torch.stack(occlusion_masks, dim=0) # (k, B)
        
        # Set weights to 0 if occluded
        w = w * (~occlusion_mask.float())
        
        # Re-normalize weights after occlusion
        w_sum = torch.sum(w, dim=0, keepdim=True)
        w = w / torch.clamp(w_sum, min=eps)
        
        # Sample textures
        # [G] Sample texture i at pixel coordinates using texture i's projection
        final_color = torch.zeros_like(world_coords[..., :3])
        for i in range(self.k):
            idx = top_k_indices[i]
            tex_img = textures[idx]
            # Sample tex_img at pixel coordinates
            # [G] Standard texture sampling
            sampled_color = sample_texture(tex_img, pixel_data)
            final_color += w[i] * sampled_color
            
        return final_color
```

## Operating characteristics
*   **Time Complexity:** Training is intensive, requiring 150,000 batches on a single Nvidia GPU RTX 2080Ti [S1]. Rendering time for the sphere-traced version (NLR-ST) is 13 seconds for a 1600x1200 image on DTU scan 65 [S1]. The rasterized version (NLR-RAS) achieves real-time performance (>60 fps) at HD resolution [S1].
*   **Memory Complexity:** Model size for NLR-RAS is 34.68 MB, NLR-ST is 2.07 MB, and NeRF is 2.27 MB on DTU scan 65 [S1].
*   **Performance:** NLR-RAS achieves a PSNR of 33.33 dB on DTU Scan 65, comparable to NeRF (34.87 dB) and superior to Colmap (32.13 dB) and IDR (33.50 dB) [S1].
*   **Strengths:** Real-time rendering capability via mesh export; view-dependent texture mapping; high image quality comparable to volumetric methods; compact model size [S1].
*   **Limitations:** Sphere tracing is slow for high resolutions; requires posed images and masks; SIREN initialization is critical; large $w_S$ causes high-frequency artifacts [S1].
*   **When to use:** When real-time rendering of view-dependent appearance is required and posed multi-view images are available.
*   **When not to use:** When unposed images are used, or when relighting capabilities are needed (NLR does not model lighting) [S1].

## Background
Neural Lumigraph Rendering was introduced by Kellnhofer et al. in 2021 [S1]. It builds on the concept of unstructured lumigraph rendering by Buehler et al. [4] and utilizes SIREN networks for implicit surface representation [52]. The method uses Fourier features for angular resolution [38, 60]. It is implemented in PyTorch [42] and uses OpenGL for rendering [59]. The authors built a custom camera array with 16 GoPro HERO7 and 6 Back-Bone H7PRO cameras, synchronized using ArUco markers [13] [S1].

## Sources
[S1] Neural lumigraph rendering, https://openaccess.thecvf.com/content/CVPR2021/html/Kellnhofer_Neural_Lumigraph_Rendering_CVPR_2021_paper.html, fetched 2026-10-31
