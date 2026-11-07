# ProNeRF

Also called: ProNeRF-12, ProNeRF-32, ProNeRF-8, Projection-Aware Ray Sampling for Fine-Grained Implicit Neural Radiance Fields

## Overview
ProNeRF is a few-point volumetric rendering method that uses a Projection-Aware Sampling (PAS) network to predict sparse 3D sampling points and opacity adjustments, allowing high-quality rendering with significantly fewer samples than standard NeRF [S1]. It achieves faster inference than vanilla NeRF with higher quality than HyperReel by combining a coarse-to-fine sampling strategy with an alternating exploration-exploitation training loop [S1]. Unlike generalizable radiance fields that store all training input images for epipolar feature projection, ProNeRF uses a fixed subset of reference views, ensuring a consistent memory footprint [S1].

## How it works
ProNeRF computes a radiance field function $f_\theta(\mathbf{x}, \mathbf{d})$ that outputs volume density $\sigma$ and color $\mathbf{c}$ [S1]. The final pixel color $\mathbf{C}(\mathbf{r})$ for a ray $\mathbf{r}$ is computed via the standard volume rendering integral [Parent]. ProNeRF differs by using a PAS network to generate a sparse set of sampling points $T$ and adjusting opacities to compensate for the subsampled transmittance [S1].

### Inputs and Outputs
*   **Input:** Ray $\mathbf{r}$ defined by origin $\mathbf{r}_o$ and direction $\mathbf{r}_d$, near/far bounds $t_n, t_f$.
*   **Output:** Rendered color $\hat{\mathbf{c}}(\mathbf{r})$.

### Data Structures
1.  **Plücker Ray-Point Encoding ($\mathbf{r}_{pr}$):**
    *   A vector of size $N_{pr} \times 9$ (or concatenated components).
    *   Calculated as $\mathbf{r}_{pr} = [\mathbf{r}_d, \mathbf{r}_o + \mathbf{r}_d \odot \mathbf{t}_{nf}, (\mathbf{r}_o + \mathbf{r}_d \odot \mathbf{t}_{nf}) \odot \mathbf{r}_d]$, where $\mathbf{t}_{nf}$ is a vector of $N_{pr}$ elements evenly spaced between $t_n$ and $t_f$, and $\odot$ is the Hadamard product [S1].
    *   Typical value: $N_{pr} = 48$ [S1].

2.  **Coarse Sampling Distances ($T'$):**
    *   Vector of $N_s$ distances $\{t'_1, \dots, t'_{N_s}\}$ along the ray.
    *   Predicted by $F_{\theta_c}$ [S1].

3.  **Refined Sampling Distances ($T$):**
    *   Vector of $N_s$ distances $\{t_1, \dots, t_{N_s}\}$.
    *   Obtained by linear interpolation between consecutive elements of the expanded set $\dot{T} = \{t_n, t'_1, \dots, t'_{N_s}, t_f\}$ using refinement weights $\Delta_T$ [S1]:
        $$ T_i = \frac{1}{2} \left( (\dot{T}_i + \dot{T}_{i+1}) + \Delta_{T_i}(\dot{T}_{i+2} - \dot{T}_i) \right) $$
        for $i=1 \dots N_s$ [S1].

4.  **Color-to-Ray Projections ($\mathbf{f}_{p}$):**
    *   For each coarse point $p_i = \mathbf{r}(t'_i)$, projections are computed by tracing lines from $p_i$ to the camera centers of $N_n$ neighboring reference views [S1].
    *   $\mathbf{f}_{p_i} = [c^1_{p_i}, c^2_{p_i}, \dots, c^{N_n}_{p_i}]$, where $c^k_{p_i}$ is the color from the $k$-th view [S1].

5.  **Network Outputs:**
    *   $F_{\theta_c}$ outputs: $T'$, opacity shifts/scales $A_t, B_t$, coarse color $\hat{\mathbf{c}}_c$, and auxiliary weights $W, M$ (via $F_{\theta_f}$) [S1].
    *   $F_{\theta_s}$ outputs: density $\sigma_i$ and color $\mathbf{c}_i$ for each point in $T$ [S1].

### Network Architecture
*   **Coarse MLP ($F_{\theta_c}$):** 6 fully connected layers, 256 neurons, ELU activations [S1].
*   **Feature MLP ($F_{\theta_f}$):** 6 fully connected layers, 256 neurons, ELU activations [S1].
*   **Shading Network ($F_{\theta_s}$):** 8 fully connected layers, 256 neurons [S1].
*   **Pruning MLP ($F_{\theta_m}$):** Same complexity as $F_{\theta_c}$, predicts error between $\hat{\mathbf{c}}_f$ and $\hat{\mathbf{c}}$ [S1].

### Parameters
*   $N_s$: Number of samples (e.g., 8, 12, 32) [S1].
*   $N$: Maximum number of exploration samples (default 64) [S1].
*   $N_n$: Number of neighboring reference views for projection (significantly smaller than total training views $N_t$) [S1].
*   $\lambda$: Auxiliary loss weight (1 for first 60% of training, 0 afterward) [S1].
*   Batch size $N_r$: 4,096 rays [S1].
*   Learning rate: $5 \times 10^{-4}$, exponentially decayed for 700,000 iterations [S1].

### Steps

1.  **Encode Ray:** Compute $\mathbf{r}_{pr}$ using $\mathbf{r}_o, \mathbf{r}_d, t_n, t_f$ and $N_{pr}=48$ [S1].
2.  **Coarse Sampling:** Pass $\mathbf{r}_{pr}$ to $F_{\theta_c}$ to obtain coarse distances $T'$, opacity adjustments $A_t, B_t$, and coarse color $\hat{\mathbf{c}}_c$ [S1].
3.  **Color-to-Ray Projection:** For each $t'_i \in T'$, compute point $p_i = \mathbf{r}(t'_i)$. Trace lines from $p_i$ to the camera centers of $N_n$ reference views. Collect colors to form $\mathbf{f}_{p_i}$ [S1].
4.  **Refine Sampling:** Pass $\mathbf{r}_{pr}$ and $\{\mathbf{f}_{p_i}\}_{i=1}^{N_s}$ to $F_{\theta_f}$ to obtain refinement weights $\Delta_T$, aggregation weights $W$, and mixture weights $M$ [S1]. Compute refined distances $T$ using the formula above [S1].
5.  **Approximate Volumetric Rendering (AVR) for Auxiliary Output:**
    *   Compute sub-light-field views: $c_{avr}^k = \sum_{i=1}^{N_s} W_i c_{p_i}^k$ for each view $k \in \{1, \dots, N_n\}$ [S1].
    *   Aggregate: $\hat{\mathbf{c}}_f = \sum_{k=1}^{N_n} M_k c_{avr}^k$ [S1].
6.  **Shading:** Query $F_{\theta_s}$ at points $\mathbf{r}(t_i)$ for $t_i \in T$ to get $\sigma_i, \mathbf{c}_i$ [S1].
7.  **Render Final Color:** Compute opacity $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ where $\delta_i = t_{i+1} - t_i$ (with $t_{N_s+1}$ handled appropriately, typically $t_f$ or adjusted by $B_t$) [S1]. Apply opacity adjustments $A_t, B_t$ to $\alpha_i$ to compensate for sparse sampling [S1]. Render $\hat{\mathbf{c}}(\mathbf{r}) = \sum_{i=1}^{N_s} (\prod_{j=1}^{i-1} (1 - \hat{\alpha}_j)) \hat{\alpha}_i \mathbf{c}_i$ [S1].
8.  **Pruning (Optional):** If $F_{\theta_m}$ predicts low error between $\hat{\mathbf{c}}_f$ and $\hat{\mathbf{c}}$, return $\hat{\mathbf{c}}_f$ (AVR mode); otherwise, return $\hat{\mathbf{c}}$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

class ProNeRF:
    def __init__(self, N_s=12, N_pr=48, N_n=8, N_explore=64):
        self.N_s = N_s
        self.N_pr = N_pr
        self.N_n = N_n
        self.N_explore = N_explore
        # Initialize networks F_theta_c, F_theta_f, F_theta_s, F_theta_m
        self.F_c = CoarseMLP()
        self.F_f = FeatureMLP()
        self.F_s = ShadingMLP()
        self.F_m = PruningMLP()
        
        # Reference views colors and camera matrices (fixed subset)
        self.ref_colors = None 
        self.ref_cameras = None

    def encode_ray(self, r_o, r_d, t_n, t_f):
        # r_o: (B, 3), r_d: (B, 3)
        t_nf = torch.linspace(t_n, t_f, self.N_pr).to(r_o.device)
        t_nf = t_nf.unsqueeze(0).expand(r_o.shape[0], -1) # (B, N_pr)
        
        # r_pr = [r_d, r_o + r_d * t_nf, (r_o + r_d * t_nf) * r_d]
        # Note: r_d is (B, 3), t_nf is (B, N_pr). Broadcasting needed.
        # r_d_expanded: (B, 1, 3) -> (B, N_pr, 3)
        r_d_exp = r_d.unsqueeze(1).expand(-1, self.N_pr, -1)
        t_nf_exp = t_nf.unsqueeze(2).expand(-1, -1, 3)
        
        term1 = r_d_exp
        term2 = r_o.unsqueeze(1) + r_d_exp * t_nf_exp
        term3 = term2 * r_d_exp
        
        r_pr = torch.cat([term1, term2, term3], dim=-1) # (B, N_pr, 9)
        return r_pr

    def get_projections(self, r_o, r_d, t_coarse, ref_cameras, ref_colors):
        # t_coarse: (B, N_s)
        # Points p_i = r_o + r_d * t_coarse
        pts = r_o.unsqueeze(1) + r_d.unsqueeze(1) * t_coarse.unsqueeze(2) # (B, N_s, 3)
        
        # For each point, project to N_n reference views
        # ref_cameras: (N_n, 4, 4) or similar, ref_colors: (N_n, H, W, 3)
        # This step requires ray-tracing into reference views to get colors
        # Simplified: assume function get_color_from_view(camera, point) exists
        # Returns (B, N_s, N_n, 3)
        projections = torch.zeros(pts.shape[0], pts.shape[1], self.N_n, 3).to(pts.device)
        
        for k in range(self.N_n):
            cam = ref_cameras[k]
            # Trace ray from cam center through pts to find intersection with reference image plane
            # Or simply sample color at projected pixel coordinates
            # Placeholder for actual projection logic
            colors_k = self.sample_reference_view(cam, pts) 
            projections[:, :, k, :] = colors_k
        return projections

    def forward(self, r_o, r_d, t_n, t_f, ref_cameras, ref_colors, is_training=False, iteration=0):
        r_pr = self.encode_ray(r_o, r_d, t_n, t_f)
        
        # 1. Coarse Sampling
        out_c = self.F_c(r_pr)
        t_coarse = out_c['t_coarse'] # (B, N_s)
        A_t, B_t = out_c['A_t'], out_c['B_t']
        c_hat_c = out_c['c_hat_c'] # (B, 3)
        
        # 2. Projections
        f_p = self.get_projections(r_o, r_d, t_coarse, ref_cameras, ref_colors)
        
        # 3. Refine Sampling
        # Concatenate r_pr and f_p for F_f input
        # f_p: (B, N_s, N_n, 3) -> reshape or aggregate? 
        # Source says F_f takes [r'_pr, f_p1, ..., f_pNs]
        # Assuming r'_pr is broadcasted or repeated
        r_pr_exp = r_pr.unsqueeze(1).expand(-1, self.N_s, -1, -1) # (B, N_s, N_pr, 9)
        f_p_flat = f_p.view(f_p.shape[0], f_p.shape[1], -1) # (B, N_s, N_n*3)
        
        input_f = torch.cat([r_pr_exp, f_p_flat], dim=-1)
        out_f = self.F_f(input_f)
        delta_T = out_f['delta_T'] # (B, N_s)
        W = out_f['W'] # (B, N_s)
        M = out_f['M'] # (B, N_n)
        
        # Compute refined T
        t_nf_all = torch.cat([torch.full((r_o.shape[0], 1), t_n).to(r_o.device), t_coarse, torch.full((r_o.shape[0], 1), t_f).to(r_o.device)], dim=1) # (B, N_s+2)
        # T_i = 0.5 * ( (t_nf_all[i] + t_nf_all[i+1]) + delta_T[i] * (t_nf_all[i+2] - t_nf_all[i]) )
        t_prev = t_nf_all[:, :-2]
        t_curr = t_nf_all[:, 1:-1]
        t_next = t_nf_all[:, 2:]
        t_refined = 0.5 * ((t_prev + t_curr) + delta_T * (t_next - t_prev))
        
        # 4. AVR Auxiliary Output
        # c_avr^k = sum_i W_i * c_p_i^k
        # W: (B, N_s), f_p: (B, N_s, N_n, 3)
        # c_avr: (B, N_n, 3)
        c_avr = torch.sum(W.unsqueeze(2) * f_p, dim=1) # (B, N_n, 3)
        # c_hat_f = sum_k M_k * c_avr^k
        c_hat_f = torch.sum(M.unsqueeze(1) * c_avr, dim=1) # (B, 3)
        
        # 5. Shading
        pts_refined = r_o.unsqueeze(1) + r_d.unsqueeze(1) * t_refined.unsqueeze(2) # (B, N_s, 3)
        sigma, c_shade = self.F_s(pts_refined) # (B, N_s), (B, N_s, 3)
        
        # 6. Render
        # Compute delta_t
        t_expanded = torch.cat([t_refined, torch.full((r_o.shape[0], 1), t_f).to(r_o.device)], dim=1)
        delta_t = t_expanded[:, 1:] - t_expanded[:, :-1]
        
        alpha = 1 - torch.exp(-sigma * delta_t)
        # Apply adjustments A_t, B_t? Source says adjust alpha_i.
        # Assuming A_t is shift, B_t is scale or similar. 
        # Standard NeRF opacity adjustment in ProNeRF: alpha_hat = alpha * B_t + A_t? 
        # Or alpha_hat = 1 - exp(-(sigma * delta_t + A_t) * B_t)?
        # Source Eq 5 mentions adjusted opacities. Let's assume simple scaling/shift for implementation gap [G]
        # Common practice: alpha_hat = alpha * B_t + A_t (clamped)
        alpha_hat = torch.clamp(alpha * B_t + A_t, min=0, max=1)
        
        T_trans = torch.cumprod(1 - alpha_hat, dim=1)
        T_trans = torch.cat([torch.ones_like(T_trans[:, :1]), T_trans[:, :-1]], dim=1)
        
        c_hat = torch.sum(T_trans * alpha_hat.unsqueeze(2) * c_shade, dim=1)
        
        return c_hat, c_hat_c, c_hat_f, alpha_hat

    def training_step(self, batch, iteration):
        r_o, r_d, c_gt = batch['rays'], batch['dirs'], batch['colors']
        t_n, t_f = batch['t_n'], batch['t_f']
        
        c_hat, c_hat_c, c_hat_f, _ = self.forward(r_o, r_d, t_n, t_f, ...)
        
        loss_primary = torch.mean((c_hat - c_gt)**2)
        
        if iteration < 400000:
            loss_aux = torch.mean((c_hat_c - c_gt)**2) + torch.mean((c_hat_f - c_gt)**2)
            loss = loss_primary + 1.0 * loss_aux
            # Update all networks
            loss.backward()
            self.optimizer_all.step()
        else:
            loss = loss_primary
            loss.backward()
            self.optimizer_all.step()
            
        # Exploration pass logic handled by alternating optimizer or sampling strategy
        # If exploration pass:
        # Sample N_s+ random points around T, add noise, query F_s, compute VRE loss, update F_s only
        pass
```

### Edge Cases and Pitfalls
*   **Precision:** Use FP16 for inference (TensorRT) but train in FP32 or mixed precision to avoid instability in the alternating training loop [S1].
*   **Boundary Conditions:** The expanded set $\dot{T}$ includes $t_n$ and $t_f$. The refinement formula uses $t_{i+2}$, so the last two elements of $\dot{T}$ are not used as the "next" term for the last two refinements, or boundary conditions must be handled carefully [S1].
*   **Auxiliary Loss:** Removing the auxiliary loss $l_a$ causes an almost 1 dB drop in PSNR [S1].
*   **Plücker Encoding:** Removing Plücker ray-point encoding causes an almost 0.5 dB PSNR drop [S1].

## Operating characteristics
*   **Speed:** 15-23× faster than vanilla NeRF [S1]. ProNeRF-8 is >20× faster, ProNeRF-12 is ~15× faster [S1]. ProNeRF-8 prune is 23% faster than standard ProNeRF-8 [S1].
*   **Quality:** 0.65 dB higher PSNR than NeRF, 0.95 dB higher than HyperReel on LLFF [S1]. ProNeRF-8 achieves 26.78 dB PSNR on LLFF (vs 26.50 dB for NeRF) [S1].
*   **Memory:** Similar to NeRF. Pruning adds 3.3 MB overhead [S1]. ProNeRF-32 uses 6.3 MB memory on Blender [S1].
*   **Scaling:** Performance scales with the number of samples $N_s$. More samples improve quality but reduce speed gains [S1].
*   **Strengths:** High quality with few samples, consistent memory footprint, no need for all training views at inference [S1].
*   **Limitations:** Requires careful tuning of the exploration-exploitation balance; pruning adds complexity [S1].

## Background
ProNeRF was proposed in 2024 by authors publishing in IEEE Access [S1]. It addresses the trade-off between memory, speed, and quality in neural radiance fields, specifically improving upon sampler-based methods like HyperReel and K-Planes [S1]. It is evaluated on LLFF and Blender datasets [S1].

## Sources
[S1] ProNeRF: Learning Efficient Projection-Aware Ray Sampling for Fine-Grained Implicit Neural Radiance Fields, https://doi.org/10.1109/access.2024.3390753, fetched 2026-10-30
