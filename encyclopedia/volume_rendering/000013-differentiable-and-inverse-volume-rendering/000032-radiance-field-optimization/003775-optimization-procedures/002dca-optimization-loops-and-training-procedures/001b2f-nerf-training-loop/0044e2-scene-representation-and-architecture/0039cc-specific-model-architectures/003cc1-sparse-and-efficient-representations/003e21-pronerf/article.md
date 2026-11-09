# ProNeRF

Also called: ProNeRF-12, ProNeRF-32, ProNeRF-8, Projection-Aware Ray Sampling for Fine-Grained Implicit Neural Radiance Fields

## Overview
ProNeRF is a sparse sampling method for implicit neural radiance fields that uses a Projection-Aware Sampling (PAS) network to predict a small number of high-density 3D points per ray, enabling high-quality rendering with significantly fewer samples than standard NeRF [S1]. It achieves an optimal trade-off between memory footprint, speed, and quality, being 15–23× faster than vanilla NeRF while achieving higher PSNR than both NeRF and HyperReel [S1]. Unlike generalizable radiance fields that require storing all training images for epipolar projection, ProNeRF uses a fixed, small subset of reference views, ensuring a consistent memory footprint regardless of the number of training views [S1].

## How it works
ProNeRF decomposes the rendering process into a coarse-to-fine sampling pipeline driven by a PAS network, followed by a shading network for volumetric rendering. The method employs an alternating exploration-exploitation training strategy to align the sparse sampler with the full scene density distribution.

### Inputs and Outputs
*   **Input:** A target ray $\mathbf{r} = (\mathbf{r}_o, \mathbf{r}_d)$ defined by origin $\mathbf{r}_o$ and direction $\mathbf{r}_d$, bounded by near $t_n$ and far $t_f$ distances.
*   **Output:** A synthesized pixel color $\hat{\mathbf{c}}(\mathbf{r})$.

### Data Structures and Network Layout
ProNeRF consists of three main neural components:

1.  **Coarse MLP Head ($F_{\theta_c}$):**
    *   **Architecture:** 6 fully connected layers, 256 neurons per layer, ELU activation.
    *   **Input:** Plücker ray-point encoding $\mathbf{r}_{pr}$ (48 points).
    *   **Output:** Coarse distances $\mathbf{T}'$, opacity shifts/scales $\mathbf{A}_t, \mathbf{B}_t$, coarse color $\hat{\mathbf{c}}_c$, and auxiliary weights $\mathbf{W}, \mathbf{M}$ (via softmax/sigmoid).

2.  **Feature MLP Head ($F_{\theta_f}$):**
    *   **Architecture:** 6 fully connected layers, 256 neurons per layer, ELU activation.
    *   **Input:** Refined ray encoding $\mathbf{r}'_{pr}$ and aggregated color-to-ray projections $\mathbf{f}_{p_i}$.
    *   **Output:** Refinement weights $\Delta_{\mathbf{T}}$, auxiliary weights $\mathbf{W}, \mathbf{M}$.

3.  **Shading Network ($F_{\theta_s}$):**
    *   **Architecture:** 8 fully connected layers, 256 neurons per layer (based on DoNeRF).
    *   **Input:** 3D points $\mathbf{x} = \mathbf{r}_o + \mathbf{r}_d \odot t$.
    *   **Output:** Density $\sigma$ and color $\mathbf{c}$.

4.  **Pruning Head ($F_{\theta_m}$) (Optional, ProNeRF-8 prune):**
    *   **Architecture:** Same complexity as $F_{\theta_c}$.
    *   **Output:** Predicted error between fast output $\hat{\mathbf{c}}_f$ and full output $\hat{\mathbf{c}}$.

### Plücker Ray-Point Encoding
To encode the ray and spatial context, ProNeRF uses a Plücker ray-point representation. Let $t_{nf}$ be a vector of $N_{pr}$ elements evenly spaced between $t_n$ and $t_f$. The encoding is:
$$
\mathbf{r}_{pr} = [\mathbf{r}_d, \mathbf{r}_o + \mathbf{r}_d \odot t_{nf}, (\mathbf{r}_o + \mathbf{r}_d \odot t_{nf}) \odot \mathbf{r}_d]
$$
where $\odot$ is the Hadamard product and $[\cdot, \cdot]$ is concatenation [S1].

### Rendering Pipeline

#### 1. Coarse Sampling
The ray $\mathbf{r}$ is encoded into $\mathbf{r}_{pr}$. $F_{\theta_c}$ processes $\mathbf{r}_{pr}$ to output coarse sampling distances $\mathbf{T}' = \{t'_1, \dots, t'_{N_s}\}$ along the ray, where $N_s$ is the target number of samples (e.g., 8, 12, or 32) [S1].

#### 2. Color-to-Ray Projection
For each coarse point $p_i = \mathbf{r}(t'_i)$, color-to-ray projections are computed by tracing lines from $p_i$ to the camera centers of $N_n$ neighboring reference views from a fixed pool [S1]. Let $\mathbf{c}^k_{p_i}$ be the color projected from the $k$-th view. The aggregated projection vector for point $p_i$ is:
$$
\mathbf{f}_{p_i} = [\mathbf{c}^1_{p_i}, \mathbf{c}^2_{p_i}, \dots, \mathbf{c}^{N_n}_{p_i}]
$$
This step makes the sampler "projection-aware," allowing it to learn geometric information implicitly contained in multi-view consistency [S1].

#### 3. Fine Refinement
The refined ray encoding $\mathbf{r}'_{pr}$ and the sequence of projections $\{\mathbf{f}_{p_1}, \dots, \mathbf{f}_{p_{N_s}}\}$ are fed into $F_{\theta_f}$. The network outputs refinement weights $\Delta_{\mathbf{T}}$ [S1]. The final refined distances $\mathbf{T} = \{t_1, \dots, t_{N_s}\}$ are obtained by linear interpolation between consecutive elements of the expanded set $\dot{\mathbf{T}} = \{t_n, t'_1, \dots, t'_{N_s}, t_f\}$:
$$
t_i = \frac{1}{2} \left( (\dot{T}_i + \dot{T}_{i+1}) + \Delta_{T_i}(\dot{T}_{i+2} - \dot{T}_i) \right)
$$
where indices are managed to handle boundaries [S1].

#### 4. Volumetric Rendering with Opacity Adjustment
The refined points $\mathbf{T}$ are queried from the shading network $F_{\theta_s}$ to obtain densities $\sigma_i$ and colors $\mathbf{c}_i$. ProNeRF adjusts the opacity $\alpha_i$ to compensate for the sparse sampling, using the shifts $\mathbf{A}_t$ and scales $\mathbf{B}_t$ predicted by $F_{\theta_c}$:
$$
\hat{\alpha}_i = 1 - \exp(-\sigma_i \delta_i \cdot \mathbf{B}_{t_i} + \mathbf{A}_{t_i})
$$
where $\delta_i = t_{i+1} - t_i$ [S1]. The final color is rendered via alpha compositing:
$$
\hat{\mathbf{c}}(\mathbf{r}) = \sum_{i=1}^{N_s} \left( \prod_{j=1}^{i-1} (1 - \hat{\alpha}_j) \right) \hat{\alpha}_i \mathbf{c}_i
$$

#### 5. Auxiliary Light-Field Outputs (AVR)
$F_{\theta_f}$ also predicts weights $\mathbf{W}$ and $\mathbf{M}$ to approximate the Volume Rendering Equation (VRE) using the projected colors, known as Approximated Volumetric Rendering (AVR). For each view $k$:
$$
\hat{\mathbf{c}}^k_{avr} = \sum_{i=1}^{N_s} W_i \mathbf{c}^k_{p_i}
$$
The final auxiliary color is aggregated:
$$
\hat{\mathbf{c}}_f = \sum_{k=1}^{N_n} M_k \hat{\mathbf{c}}^k_{avr}
$$
This output $\hat{\mathbf{c}}_f$ serves as a regularization term during training and can be used for fast inference in pruned modes [S1].

### Training Strategy: Exploration and Exploitation
ProNeRF uses two alternating passes to train the networks, ensuring the sparse sampler learns to focus on high-density regions while the shading network learns the full distribution [S1].

*   **Exploration Pass:**
    *   **Goal:** Train $F_{\theta_s}$ to learn the full scene density/color distribution.
    *   **Sampling:** Randomly interpolate $N_s$ distances in $\mathbf{T}$ into $N^+_s$ evenly-spaced distances $\mathbf{T}^+$, where $N^+_s \in [N_s, N]$ ($N=64$). Add Gaussian noise to $\mathbf{T}^+$.
    *   **Forward:** Query $F_{\theta_s}$ at $\mathbf{T}^+$ to get $\{\sigma_i, \mathbf{c}_i\}$. Render using standard VRE (Eq. 1).
    *   **Update:** Update only $F_{\theta_s}$ using the optimizer `Opts`.
    *   **Loss:** L2 loss between rendered color and ground truth.

*   **Exploitation Pass:**
    *   **Goal:** Train the PAS network ($F_{\theta_c}, F_{\theta_f}$) and refine $F_{\theta_s}$ for sparse rendering.
    *   **Sampling:** Use the sparse refined points $\mathbf{T}$ from the PAS network.
    *   **Forward:** Query $F_{\theta_s}$ at $\mathbf{T}$. Render using PAS-guided VRE with opacity adjustments (Eq. 5). Compute auxiliary outputs $\hat{\mathbf{c}}_c$ and $\hat{\mathbf{c}}_f$.
    *   **Update:** Update all networks ($F_{\theta_c}, F_{\theta_f}, F_{\theta_s}$) using optimizer `Opt_{cfs}`.
    *   **Loss:** Primary L2 loss plus auxiliary losses for $\hat{\mathbf{c}}_c$ and $\hat{\mathbf{c}}_f$ (if iteration $< 4 \times 10^5$).

### Pruning (ProNeRF-8 prune)
To further accelerate inference, a complementary head $F_{\theta_m}$ is trained to predict the error between the fast output $\hat{\mathbf{c}}_f$ and the full output $\hat{\mathbf{c}}$. During inference:
1.  Compute $\hat{\mathbf{c}}_f$ using AVR.
2.  If the predicted error by $F_{\theta_m}$ is below a threshold, output $\hat{\mathbf{c}}_f$ (fast path).
3.  Otherwise, run the full shading network $F_{\theta_s}$ and standard VRE (slow path) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

class ProNeRF:
    def __init__(self, N_s=8, N_pr=48, N_n=4, N_explore=64):
        self.N_s = N_s
        self.N_pr = N_pr
        self.N_n = N_n
        self.N_explore = N_explore
        # Initialize networks
        self.F_c = CoarseMLP()
        self.F_f = FeatureMLP()
        self.F_s = ShadingNetwork()
        self.F_m = PruningMLP() # Optional
        self.ref_views = [] # Fixed subset of reference view cameras and colors

    def encode_ray(self, r_o, r_d, t_n, t_f):
        # Plücker ray-point encoding
        t_nf = torch.linspace(t_n, t_f, self.N_pr)
        # r_pr = [r_d, r_o + r_d * t_nf, (r_o + r_d * t_nf) * r_d]
        term1 = r_o + r_d.unsqueeze(-1) * t_nf.unsqueeze(0)
        term2 = term1 * r_d.unsqueeze(-1)
        r_pr = torch.cat([r_d, term1, term2], dim=-1)
        return r_pr

    def color_to_ray_projection(self, points, ref_views):
        # Trace lines from points to camera centers of N_n neighbors
        # Returns f_p: [N_s, N_n, 3]
        projections = []
        for i, p in enumerate(points):
            view_colors = []
            for k, view in enumerate(ref_views[:self.N_n]):
                # Project point p to view k's image plane and sample color
                # This is a simplified representation; actual implementation 
                # involves ray casting from p to view center and sampling texture
                c_k = sample_view_color(p, view)
                view_colors.append(c_k)
            projections.append(torch.stack(view_colors))
        return torch.stack(projections) # [N_s, N_n, 3]

    def forward(self, r_o, r_d, t_n, t_f, ref_views):
        r_pr = self.encode_ray(r_o, r_d, t_n, t_f)
        
        # 1. Coarse Sampling
        out_c = self.F_c(r_pr)
        T_prime = out_c['T_prime'] # [N_s]
        A_t, B_t = out_c['A_t'], out_c['B_t']
        c_c = out_c['c_c']
        
        # 2. Projections
        coarse_points = r_o + r_d.unsqueeze(-1) * T_prime.unsqueeze(0)
        f_p = self.color_to_ray_projection(coarse_points, ref_views) # [N_s, N_n, 3]
        
        # 3. Refinement
        r_prime_pr = self.encode_ray(r_o, r_d, t_n, t_f) # Re-encode or use refined context
        out_f = self.F_f(torch.cat([r_prime_pr, f_p.flatten(1)], dim=-1))
        delta_T = out_f['delta_T']
        W, M = out_f['W'], out_f['M']
        
        # Expand T_prime with bounds
        T_expanded = torch.cat([torch.tensor([t_n]), T_prime, torch.tensor([t_f])])
        # Refine distances
        T = 0.5 * ((T_expanded[:-1] + T_expanded[1:]) + delta_T.unsqueeze(-1) * (T_expanded[2:] - T_expanded[:-2]))
        
        # 4. Shading
        points = r_o + r_d.unsqueeze(-1) * T.unsqueeze(0)
        sigma, c = self.F_s(points)
        
        # Opacity adjustment
        deltas = T[1:] - T[:-1]
        alpha = 1.0 - torch.exp(-sigma * deltas.unsqueeze(-1) * B_t.unsqueeze(-1) + A_t.unsqueeze(-1))
        
        # Alpha compositing
        transmittance = torch.cumprod(1.0 - alpha, dim=0)
        c_hat = torch.sum(transmittance * alpha * c, dim=0)
        
        # 5. Auxiliary AVR
        # c_avr_k = sum(W_i * c_p_i^k)
        c_avr = torch.sum(W.unsqueeze(-1) * f_p, dim=0) # [N_n, 3]
        c_f = torch.sum(M.unsqueeze(-1) * c_avr, dim=0) # [3]
        
        return c_hat, c_c, c_f, alpha, sigma

    def train_step(self, ray, gt_color, ref_views, iteration):
        c_hat, c_c, c_f, alpha, sigma = self.forward(ray.r_o, ray.r_d, ray.t_n, ray.t_f, ref_views)
        
        if iteration % 2 == 0 and iteration < 400000:
            # Exploration Pass
            # Sample N_s+ points around T
            N_s_plus = torch.randint(self.N_s, self.N_explore, (1,)).item()
            T_plus = sample_uniform(ray.t_n, ray.t_f, N_s_plus)
            T_plus += torch.randn_like(T_plus) * noise_scale
            
            points_plus = ray.r_o + ray.r_d.unsqueeze(-1) * T_plus.unsqueeze(0)
            sigma_plus, c_plus = self.F_s(points_plus)
            deltas = T_plus[1:] - T_plus[:-1]
            alpha_plus = 1.0 - torch.exp(-sigma_plus * deltas.unsqueeze(-1))
            trans_plus = torch.cumprod(1.0 - alpha_plus, dim=0)
            c_hat_exp = torch.sum(trans_plus * alpha_plus * c_plus, dim=0)
            
            loss = torch.mean((c_hat_exp - gt_color)**2)
            loss.backward()
            self.optimizer_s.step()
            self.optimizer_s.zero_grad()
        else:
            # Exploitation Pass
            loss = torch.mean((c_hat - gt_color)**2)
            
            if iteration < 400000:
                loss += torch.mean((c_c - gt_color)**2)
                loss += torch.mean((c_f - gt_color)**2)
            
            loss.backward()
            self.optimizer_cfs.step()
            self.optimizer_cfs.zero_grad()

class CoarseMLP(nn.Module):
    def __init__(self):
        super().__init__()
        self.layers = nn.Sequential(
            nn.Linear(48, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256)
        )
        self.head_T = nn.Linear(256, 8)
        self.head_A = nn.Linear(256, 8)
        self.head_B = nn.Linear(256, 8)
        self.head_c = nn.Linear(256, 3)

    def forward(self, x):
        feat = self.layers(x)
        return {
            'T_prime': self.head_T(feat),
            'A_t': self.head_A(feat),
            'B_t': self.head_B(feat),
            'c_c': self.head_c(feat)
        }

class FeatureMLP(nn.Module):
    def __init__(self):
        super().__init__()
        self.layers = nn.Sequential(
            nn.Linear(48 + 8*3, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256)
        )
        self.head_delta = nn.Linear(256, 8)
        self.head_W = nn.Linear(256, 8)
        self.head_M = nn.Linear(256, 4)

    def forward(self, x):
        feat = self.layers(x)
        return {
            'delta_T': self.head_delta(feat),
            'W': torch.softmax(self.head_W(feat), dim=-1),
            'M': torch.sigmoid(self.head_M(feat))
        }

class ShadingNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        self.layers = nn.Sequential(
            nn.Linear(3, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256)
        )
        self.head_sigma = nn.Linear(256, 1)
        self.head_c = nn.Linear(256, 3)

    def forward(self, x):
        feat = self.layers(x)
        return torch.relu(self.head_sigma(feat)), torch.sigmoid(self.head_c(feat))

class PruningMLP(nn.Module):
    def __init__(self):
        super().__init__()
        self.layers = nn.Sequential(
            nn.Linear(48, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256)
        )
        self.head_err = nn.Linear(256, 1)

    def forward(self, x):
        feat = self.layers(x)
        return self.head_err(feat)
```

### Parameters and Settings
*   **Training Iterations:** $7 \times 10^5$.
*   **Auxiliary Loss Weight ($\lambda$):** 1 for the first $4 \times 10^5$ iterations (approx. 57%), then 0 [S1].
*   **Batch Size:** 4,096 rays.
*   **Learning Rate:** $5 \times 10^{-4}$, exponentially decayed over 700K iterations.
*   **Plücker Points ($N_{pr}$):** 48.
*   **Max Exploration Samples ($N$):** 64.
*   **Reference Views:** A fixed small subset ($N_n$) is used for projections. $N_n \ll N_t$ (total training views) [S1].
*   **Pruning Overhead:** 3.3 MB memory [S1].

### Edge Cases and Pitfalls
*   **Precision:** The method relies on accurate color-to-ray projections. Incorrect camera calibration or mismatched reference views can degrade quality [S1].
*   **Boundary Conditions:** The refinement formula uses $\dot{T}_{i+2}$, requiring careful handling of the last few samples near $t_f$ to avoid index out-of-bounds [S1].
*   **Auxiliary Loss Timing:** Removing the auxiliary loss ($l_a$) causes an almost 1 dB drop in PSNR, indicating its critical role in stabilizing the sparse sampling [S1].
*   **Encoding Sensitivity:** Removing the Plücker ray-point encoding causes an almost 0.5 dB PSNR drop [S1].

## Operating characteristics
*   **Speed:** ProNeRF is 15–23× faster than vanilla NeRF. ProNeRF-8 is >20× faster, and ProNeRF-8 prune is 23% faster than standard ProNeRF-8 [S1].
*   **Quality:** ProNeRF-12 achieves 27.15 dB PSNR on LLFF, 0.65 dB higher than vanilla NeRF (26.50 dB). ProNeRF-8 achieves 26.78 dB, outperforming NeRF by 0.28 dB [S1].
*   **Memory:** Memory footprint is similar to NeRF. ProNeRF-32 uses 6.3 MB. Pruning adds 3.3 MB overhead [S1].
*   **Scaling:** Performance is consistent across forward-facing (LLFF) and 360-degree (Blender) datasets. It scales well with the number of reference views used for projection, though a fixed small subset is sufficient for inference [S1].
*   **Strengths:** High quality with very few samples; consistent memory footprint; fast inference; no need for depth GT or pre-trained NeRF [S1].
*   **Limitations:** Requires careful tuning of the exploration-exploitation balance; pruning adds complexity; relies on accurate multi-view projections [S1].

## Background
ProNeRF was introduced in 2024 by researchers addressing the trade-off between rendering speed and quality in neural radiance fields [S1]. It builds upon the principles of sparse sampling methods like HyperReel and TermiNeRF but introduces the Projection-Aware Sampling (PAS) network to leverage multi-view consistency without storing all training images, unlike generalizable methods such as IBRNet or NeuRay [S1]. The method was evaluated on the LLFF and Blender datasets, demonstrating state-of-the-art performance among sampler-based methods at the time of publication [S1].

## Sources
[S1] ProNeRF: Learning Efficient Projection-Aware Ray Sampling for Fine-Grained Implicit Neural Radiance Fields, https://doi.org/10.1109/access.2024.3390753, fetched 2026-10-30
