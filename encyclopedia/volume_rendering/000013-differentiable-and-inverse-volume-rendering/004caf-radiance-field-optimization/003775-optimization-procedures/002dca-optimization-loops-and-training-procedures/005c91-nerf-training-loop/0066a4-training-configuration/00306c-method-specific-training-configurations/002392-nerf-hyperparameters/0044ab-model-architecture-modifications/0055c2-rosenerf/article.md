# RoseNeRF

Also called: Refracting Once is Enough, Refracting once is enough for NeRF

## Overview
RoseNeRF is a method for high-fidelity novel view synthesis of real refractive objects that simplifies complex ray interactions inside objects to a single refraction event [S1]. It models refraction by predicting a single refracted ray using a 4D light field representation and a refraction network, avoiding the need for Index of Refraction (IoR) fields or detailed surface reconstructions [S1]. Unlike deformation field methods that predict position offsets for each sampling point, RoseNeRF predicts the refracted ray and uses hierarchical sampling, avoiding the need for point-by-point deformation and additional priors like IoR [S1]. It is designed for seamless integration into most NeRF-like frameworks, specifically implemented on top of MS-NeRF [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera rays defined by origin $\mathbf{o}$ and direction $\mathbf{d}$. Ground truth images $\mathbf{C}(\mathbf{r})$.
*   **Output:** Synthesized pixel color $\hat{\mathbf{C}}(\mathbf{r})$.

**Data Structures and Layout:**
The method builds on the standard NeRF architecture [Parent](11313-model-architecture-modifications.md) but introduces three specific components:
1.  **RefNet (Refraction Network):** An MLP that maps incident rays to refracted rays.
    *   **Student RefNet ($\Theta_s$):** 2-layer MLP, 64 hidden units. Input: 4D light field coordinates $(u, v, s, t)$. Output: 4D refracted ray coordinates $(u_{nr}, v_{nr}, s_{nr}, t_{nr})$.
    *   **Teacher RefNet ($\Theta_t$):** 3-layer MLP, 128 hidden units. Input: 5D vector $(u, v, s, t, \mathbf{N})$, where $\mathbf{N}$ is the surface normal. Output: 4D refracted ray coordinates.
2.  **Decoder MLP ($\Theta_D$):** Fuses density features from original and refracted rays.
    *   **Architecture:** 2-layer MLP, 256 hidden units, ReLU activations.
    *   **Input:** Densities $\sigma_i$ (from original ray) and $\sigma_{ir}$ (from refracted ray).
    *   **Output:** Complete density $\sigma_{ic}$.
3.  **Base NeRF:**
    *   **Spatial MLP:** 256 hidden units. Input: Positional encoding $\gamma(\mathbf{x})$. Output: Density $\sigma$ and intermediate features.
    *   **Directional MLP:** 256 hidden units. Input: Decoded density $\sigma_{ic}$ and direction $\mathbf{d}$ (direction input last). Output: View-dependent color $\mathbf{c}$.

**Ray Parameterization (4D Light Field):**
Rays are parameterized by their intersections with two planes at $Z_0=0$ and $Z_1=1$. A ray is represented by 4D coordinates $(u, v, s, t)$, where $(u, v)$ is the intersection with $Z_0$ and $(s, t)$ is the intersection with $Z_1$ [S1]. The Cartesian direction $\mathbf{d}$ is derived as:
$$ \mathbf{d} = (Z_1 - Z_0, u - s, v - t) = (1, u - s, v - t) $$
[S1]

**Numbered Steps:**

1.  **Initial Sampling (Warm-up Phase):**
    *   For the first 5,000 iterations, only the base NeRF density $\sigma$ is queried along the original ray. The Decoder is not used [S1].
    *   Sample points $\mathbf{x}$ along the original ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$.
    *   Estimate the surface normal $\mathbf{N}$ at these points using the gradient of the predicted density (standard NeRF normal estimation) [S1].

2.  **Refracted Ray Prediction:**
    *   Convert the original ray to 4D light field coordinates $(u, v, s, t)$ [S1].
    *   **Teacher Network (Distillation Target):** Compute refracted coordinates using the teacher network:
        $$ (u_{nr}, v_{nr}, s_{nr}, t_{nr}) = \Theta_t(u, v, s, t, \mathbf{N}) $$
        [S1]
    *   **Student Network (Inference):** Compute refracted coordinates using the student network (without normal input):
        $$ (u'_{nr}, v'_{nr}, s'_{nr}, t'_{nr}) = \Theta_s(u, v, s, t) $$
        [S1]
    *   Convert the predicted 4D coordinates back to Cartesian direction $\mathbf{d}_r$ and origin $\mathbf{o}_r$. The origin $\mathbf{o}_r$ is set at the intersection of the refracted ray with the camera plane $Z = z_c$ (where $z_c$ is the camera's Z-coordinate) [S1]. The specific choice of starting point does not affect the final outcome [S1].

3.  **Hierarchical Sampling:**
    *   **Phase 1 (Stratified):** Sample along the original ray. Use the results to estimate normals $\mathbf{N}$ [S1].
    *   **Phase 2 (Importance/Uniform):**
        *   Predict the refracted ray $\mathbf{r}_r = \mathbf{o}_r + t\mathbf{d}_r$ using $\Theta_s$ [S1].
        *   Query density $\sigma_i$ at points along the original ray using the Spatial MLP.
        *   Query density $\sigma_{ir}$ at points along the refracted ray using the Spatial MLP.
        *   Apply importance sampling based on Phase 1 results to the original ray.
        *   Uniformly sample the refracted ray [S1].
    *   **Subsequent Stages:** Both original and refracted rays undergo independent importance sampling [S1].

4.  **Feature Decoding:**
    *   Feed the densities $\sigma_i$ and $\sigma_{ir}$ into the Decoder MLP $\Theta_D$:
        $$ \sigma_{ic} = \Theta_D(\sigma_i, \sigma_{ir}) $$
        [S1]
    *   Feed $\sigma_{ic}$ and the original viewing direction $\mathbf{d}$ into the Directional MLP to obtain color $\mathbf{c}_c$ [S1].

5.  **Rendering:**
    *   Compute the final pixel color $\hat{\mathbf{C}}(\mathbf{r})$ using volume rendering (alpha compositing):
        $$ \hat{\mathbf{C}}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_{ic,i} \delta_i)) \mathbf{c}_i $$
        where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_{ic,j} \delta_j)$ and $\delta_i = t_i - t_{i-1}$ [S1].

**Loss Functions:**
The total loss $\mathcal{L}$ is a weighted sum:
$$ \mathcal{L} = \lambda_1 \mathcal{L}_{dist} + \lambda_2 \mathcal{L}_{ref} + \mathcal{L}_{NeRF} $$
[S1]

1.  **Photometric Loss ($\mathcal{L}_{NeRF}$):** Standard MSE between predicted color $\hat{\mathbf{C}}(\mathbf{r})$ and ground truth $\mathbf{C}(\mathbf{r})$ [S1].
2.  **Refraction Loss ($\mathcal{L}_{ref}$):** Ensures reversibility and physical consistency. It penalizes deviations from the teacher's prediction (using normal) and enforces reversibility (incident direction vs. predicted refracted direction):
    $$ \mathcal{L}_{ref} = \sum_i (1 - \mathbf{d}_i \cdot \hat{\mathbf{d}}_{ri}) + \sum_i (1 - \mathbf{d}_i \cdot \hat{\mathbf{d}}_{rni}) $$
    where $\hat{\mathbf{d}}_{ri}$ is the direction from the student network, and $\hat{\mathbf{d}}_{rni}$ is the direction from the teacher network [S1].
3.  **Distillation Loss ($\mathcal{L}_{dist}$):** Encourages the student network (no normal) to match the teacher network (with normal):
    $$ \mathcal{L}_{dist} = \sum_i (1 - \hat{\mathbf{d}}_{ri} \cdot \hat{\mathbf{d}}_{rni}) $$
    [S1]

**Parameters:**
*   $\lambda_1 = 0.003$ (Distillation weight) [S1].
*   $\lambda_2 = 0.003$ (Refraction weight) [S1].
*   $L$: Highest frequency in positional encoding $\gamma(\mathbf{x}) = [\sin(\mathbf{x}), \cos(\mathbf{x}), \dots, \sin(2^{L-1}\mathbf{x}), \cos(2^{L-1}\mathbf{x})]$ [S1].
*   Batch size: 1024 rays [S1].
*   Iterations: 200,000 [S1].
*   Warm-up: First 5,000 iterations use only density query without Decoder [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class RoseNeRF:
    def __init__(self, config):
        # Base NeRF components
        self.spatial_mlp = SpatialMLP(hidden_dim=256)
        self.directional_mlp = DirectionalMLP(hidden_dim=256)
        
        # RoseNeRF specific components
        # Student RefNet: 4D -> 4D
        self.student_refnet = RefNetMLP(layers=2, hidden=64)
        # Teacher RefNet: 5D -> 4D (used for distillation target)
        self.teacher_refnet = RefNetMLP(layers=3, hidden=128)
        # Decoder: 2 densities -> 1 density
        self.decoder = DecoderMLP(layers=2, hidden=256, activation='relu')
        
        # Hyperparameters
        self.lambda_dist = 0.003
        self.lambda_ref = 0.003
        self.warmup_iters = 5000
        self.current_iter = 0
        
        # Positional Encoding parameters
        self.L = config.L # Highest frequency

    def positional_encoding(self, x, L):
        """Standard NeRF positional encoding."""
        # x shape: (..., 3)
        # Output shape: (..., 2*L*3)
        scales = torch.tensor([2**i for i in range(L)], device=x.device)
        xb = torch.reshape((x * scales[:, None]), (x.shape[0], -1))
        freq_bands = torch.cat([torch.sin(xb), torch.cos(xb)], dim=-1)
        return freq_bands

    def ray_to_lightfield(self, origin, direction):
        """
        Convert ray (o, d) to 4D light field (u, v, s, t).
        Planes at Z0=0, Z1=1.
        d = (1, u-s, v-t) => u-s = dx/dz, v-t = dy/dz
        Intersection with Z=0: x = o_x - o_z * (dx/dz)
        Intersection with Z=1: x = o_x + (1-o_z) * (dx/dz)
        """
        # direction is normalized or unnormalized? 
        # Formula d = (1, u-s, v-t) implies d_z = 1. 
        # We must normalize direction such that d_z = 1 for this specific parameterization 
        # or scale the coordinates. The paper defines d = (Z1-Z0, u-s, v-t).
        # Assuming d is the vector from origin to intersection.
        
        # Let's assume standard ray r(t) = o + t*d.
        # Intersection with Z=0: o_z + t0*d_z = 0 => t0 = -o_z/d_z
        # u = o_x + t0*d_x = o_x - o_z * (d_x/d_z)
        # v = o_y + t0*d_y = o_y - o_z * (d_y/d_z)
        
        # Intersection with Z=1: o_z + t1*d_z = 1 => t1 = (1-o_z)/d_z
        # s = o_x + t1*d_x = o_x + (1-o_z) * (d_x/d_z)
        # t = o_y + t1*d_y = o_y + (1-o_z) * (d_y/d_z)
        
        dz = direction[:, 2:3]
        # Avoid division by zero
        dz = torch.clamp(dz, min=1e-6)
        
        ratio_x = direction[:, 0:1] / dz
        ratio_y = direction[:, 1:2] / dz
        
        u = origin[:, 0:1] - origin[:, 2:3] * ratio_x
        v = origin[:, 1:2] - origin[:, 2:3] * ratio_y
        
        s = origin[:, 0:1] + (1.0 - origin[:, 2:3]) * ratio_x
        t = origin[:, 1:2] + (1.0 - origin[:, 2:3]) * ratio_y
        
        return torch.cat([u, v, s, t], dim=-1)

    def lightfield_to_ray(self, lightfield, camera_z):
        """
        Convert 4D light field (u, v, s, t) back to ray (origin, direction).
        Origin is set at intersection with Z = camera_z.
        """
        u, v, s, t = lightfield[:, 0:1], lightfield[:, 1:2], lightfield[:, 2:3], lightfield[:, 3:4]
        
        # Direction components
        # d_z = 1 (by definition of the parameterization space)
        # d_x = u - s
        # d_y = v - t
        direction = torch.cat([u - s, v - t, torch.ones_like(u)], dim=-1)
        direction = F.normalize(direction, dim=-1)
        
        # Origin calculation:
        # The ray passes through (u, v, 0) and (s, t, 1).
        # We want the point on this line where Z = camera_z.
        # Line equation: P(Z) = (u, v, 0) + Z * ((s-u), (t-v), 1)
        # x = u + Z*(s-u)
        # y = v + Z*(t-v)
        # z = Z
        
        origin_x = u + camera_z * (s - u)
        origin_y = v + camera_z * (t - v)
        origin_z = camera_z
        
        origin = torch.cat([origin_x, origin_y, origin_z], dim=-1)
        
        return origin, direction

    def get_normal(self, spatial_mlp, points):
        """Estimate normal from density gradient."""
        points.requires_grad_(True)
        with torch.enable_grad():
            sigma = spatial_mlp(points)
            grad = torch.autograd.grad(
                sigma, points, grad_outputs=torch.ones_like(sigma),
                create_graph=True, retain_graph=True, only_inputs=True
            )[0]
        normals = F.normalize(grad, dim=-1)
        return normals

    def forward(self, rays_o, rays_d, gt_images, spatial_mlp, directional_mlp, 
                student_refnet, teacher_refnet, decoder, optimizer):
        """
        Training step.
        """
        self.current_iter += 1
        
        # 1. Initial Sampling for Normal Estimation (Phase 1)
        # Sample along original ray
        t_vals = torch.linspace(0, 1, 64) # Example N_samples
        points_orig = rays_o[:, None, :] + rays_d[:, None, :] * t_vals[None, :, None]
        
        # Get normals from density gradient
        normals = self.get_normal(spatial_mlp, points_orig.reshape(-1, 3))
        normals = normals.reshape(rays_o.shape[0], 64, 3)
        
        # 2. Predict Refracted Rays
        # Convert original rays to 4D light field
        lightfield_orig = self.ray_to_lightfield(rays_o, rays_d)
        
        # Teacher prediction (with normal)
        # Concatenate lightfield (4) and normal (3) -> 5D input
        # Note: Normal needs to be associated with the ray. 
        # The paper implies using the estimated normal N.
        # We can use the mean normal or the normal at the first intersection.
        # Let's use the normal at the first sample point for simplicity, 
        # or average if needed. The paper says "local surface normal".
        # We'll assume N is passed as a tensor of shape (B, 3).
        # For this example, let's use the normal at the first sample point.
        N = normals[:, 0, :] 
        
        teacher_input = torch.cat([lightfield_orig, N], dim=-1)
        lightfield_refracted_teacher = teacher_refnet(teacher_input)
        
        # Student prediction (without normal)
        lightfield_refracted_student = student_refnet(lightfield_orig)
        
        # Convert refracted light fields to rays
        # Camera Z coordinate (assuming camera at Z=0 or similar, 
        # paper says Z_c is camera center Z. Let's assume Z_c = 0 for simplicity 
        # or pass it as a parameter. Standard NeRF often has camera at origin.
        # If camera is at origin, Z_c = 0.
        camera_z = 0.0 
        o_r, d_r = self.lightfield_to_ray(lightfield_refracted_student, camera_z)
        
        # 3. Hierarchical Sampling
        # Phase 2: Sample along both rays
        # Original ray: Importance sampling based on Phase 1 density
        # Refracted ray: Uniform sampling
        
        # Query densities
        # Points on original ray
        points_orig_sample = rays_o[:, None, :] + rays_d[:, None, :] * t_vals[None, :, None]
        sigma_orig = spatial_mlp(points_orig_sample.reshape(-1, 3)).reshape(rays_o.shape[0], 64, 1)
        
        # Points on refracted ray
        points_refr_sample = o_r[:, None, :] + d_r[:, None, :] * t_vals[None, :, None]
        sigma_refr = spatial_mlp(points_refr_sample.reshape(-1, 3)).reshape(rays_o.shape[0], 64, 1)
        
        # 4. Decoding
        # If warm-up, skip decoder
        if self.current_iter < self.warmup_iters:
            sigma_combined = sigma_orig
            color = directional_mlp(sigma_combined, rays_d)
        else:
            # Decoder takes sigma_orig and sigma_refr
            # Shape: (B, N, 1) -> (B, N, 2) for decoder input?
            # Decoder MLP usually takes concatenated features.
            decoder_input = torch.cat([sigma_orig, sigma_refr], dim=-1)
            sigma_combined = decoder(decoder_input)
            
            # Directional MLP takes sigma_combined and direction
            # Paper: "feed the decoded density and direction into the directional MLP"
            # "Inputting the direction into the network last"
            color = directional_mlp(sigma_combined, rays_d)
            
        # 5. Rendering
        # Alpha compositing
        # delta = t[i] - t[i-1]
        deltas = t_vals[1:] - t_vals[:-1]
        deltas = torch.cat([deltas, torch.tensor([1e10], device=deltas.device)], dim=0)
        
        # T_i = exp(-sum(sigma * delta))
        alpha = 1.0 - torch.exp(-sigma_combined * deltas[:, None])
        transmittance = torch.cumprod(torch.exp(-sigma_combined * deltas[:, None]), dim=1)
        transmittance = torch.cat([torch.ones_like(transmittance[:, 0:1]), transmittance[:, :-1]], dim=1)
        
        weights = alpha * transmittance
        rendered_color = torch.sum(weights * color, dim=1)
        
        # 6. Loss Calculation
        loss_neurf = F.mse_loss(rendered_color, gt_images)
        
        if self.current_iter >= self.warmup_iters:
            # Refraction Loss
            # Convert student and teacher lightfields to directions
            d_student = self.lightfield_to_ray(lightfield_refracted_student, camera_z)[1]
            d_teacher = self.lightfield_to_ray(lightfield_refracted_teacher, camera_z)[1]
            
            # L_ref = sum(1 - d_i . d_hat_ri) + sum(1 - d_i . d_hat_rni)
            # d_i is original direction
            l_ref = torch.mean(1 - torch.sum(rays_d * d_student, dim=-1)) + \
                    torch.mean(1 - torch.sum(rays_d * d_teacher, dim=-1))
            
            # Distillation Loss
            # L_dist = sum(1 - d_hat_ri . d_hat_rni)
            l_dist = torch.mean(1 - torch.sum(d_student * d_teacher, dim=-1))
            
            loss = self.lambda_dist * l_dist + self.lambda_ref * l_ref + loss_neurf
        else:
            loss = loss_neurf
            
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        return loss.item()
```

## Operating characteristics
*   **Time Complexity:** The method adds the cost of two additional MLP evaluations (Student and Teacher RefNet) and one Decoder MLP evaluation per ray per iteration. The RefNet is shallow (2-3 layers), so the overhead is low compared to the base NeRF [S1].
*   **Memory Complexity:** Requires storing parameters for Student RefNet, Teacher RefNet, and Decoder. Student: $2 \times 64 \times 4 + 64$ (bias) $\approx$ small. Teacher: $3 \times 128 \times 5 + 128$. Decoder: $2 \times 256 \times 2 + 256$. Total extra parameters are minimal compared to the base NeRF [S1].
*   **Measured Performance:** On the Eikonal Fields dataset (Ball, Glass, Pen, WineGlass) and Shiny dataset, RoseNeRF demonstrates state-of-the-art performance without additional information [S1]. It outperforms Ref-NeRF and NeX (which has 500x more parameters) in detail and competitive specular highlights [S1].
*   **Strengths:**
    *   No need for IoR fields or surface masks [S1].
    *   Simplifies complex refraction to a single event, reducing computational load [S1].
    *   Avoids floater artifacts common in standard NeRF on refractive objects [S1].
    *   Easy integration with other NeRF variants [S1].
*   **Limitations:**
    *   Assumes a single refraction event; may not handle complex internal reflections or multiple refractions accurately [S1].
    *   Performance depends on the accuracy of the initial normal estimation [S1].
    *   Removing the refraction loss disrupts training [S1].
    *   Using origin/direction instead of 4D light field worsens accuracy [S1].

## Background
RoseNeRF (Refracting Once is Enough for NeRF) was introduced in 2024 by Xiaoqian Liang et al. [S1]. It addresses the challenge of novel view synthesis for real-world refractive objects, where standard NeRF fails due to floater artifacts caused by incorrect sampling along bent ray paths [S1]. The method is inspired by Han et al. [20] (Eikonal Fields), who demonstrated the feasibility of reconstructing transparent objects by immersing them in liquid, but RoseNeRF simplifies this by not requiring detailed simulations of light interactions or assumptions about object shapes [S1]. It builds on MS-NeRF [59] for the decoding strategy [S1].

## Sources
[S1] Refracting Once is Enough: Neural Radiance Fields for Novel-View Synthesis of Real Refractive Objects, https://doi.org/10.1145/3652583.3658000, fetched 2026-11-06
