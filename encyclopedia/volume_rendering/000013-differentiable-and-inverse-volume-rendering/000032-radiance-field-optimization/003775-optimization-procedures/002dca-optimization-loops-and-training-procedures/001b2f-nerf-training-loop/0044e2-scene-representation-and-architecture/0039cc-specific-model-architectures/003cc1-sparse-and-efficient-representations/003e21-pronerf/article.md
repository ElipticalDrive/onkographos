# ProNeRF

Also called: ProNeRF-12, ProNeRF-32, ProNeRF-8, Projection-Aware Ray Sampling for Fine-Grained Implicit Neural Radiance Fields

## Overview
ProNeRF is a few-point volumetric rendering method that uses a Projection-Aware Sampling (PAS) network to predict sparse 3D sampling points and opacity adjustments, allowing high-quality rendering with significantly fewer samples than standard NeRF [S1]. It achieves an optimal trade-off between memory footprint, speed, and quality, being 15–23× faster than vanilla NeRF while achieving higher PSNR than both NeRF and HyperReel [S1]. Unlike generalizable radiance fields that store all training images for epipolar projection, ProNeRF uses a fixed, small subset of reference views, ensuring a consistent memory footprint regardless of the number of training views [S1].

## How it works
ProNeRF consists of three main neural components: a coarse sampling head ($F_{\theta_c}$), a refinement head ($F_{\theta_f}$), and a shading network ($F_{\theta_s}$) [S1]. The method operates by first predicting coarse sample locations, projecting colors from reference views onto these locations to gather geometric context, refining the sample locations, and finally rendering using the shading network with adjusted opacities [S1].

### Inputs and Outputs
*   **Inputs:** Ray origin $\mathbf{r}_o$, ray direction $\mathbf{r}_d$, near/far bounds $t_n, t_f$, and a fixed set of $N_n$ neighboring reference view colors $\{\mathbf{c}_k\}_{k=1}^{N_n}$ [S1].
*   **Outputs:** Rendered ray color $\hat{\mathbf{c}}(\mathbf{r})$ [S1].

### Data Structures and Layouts
1.  **Plücker Ray-Point Encoding:** The ray is encoded into a vector $\mathbf{r}_{pr}$ of size $3 \times N_{pr}$ [S1].
    *   $N_{pr} = 48$ points are evenly spaced between $t_n$ and $t_f$ [S1].
    *   For each point $t_{nf}$ in the spacing vector:
        $$ \mathbf{r}_{pr} = [\mathbf{r}_d, \mathbf{r}_o + \mathbf{r}_d \odot t_{nf}, (\mathbf{r}_o + \mathbf{r}_d \odot t_{nf}) \odot \mathbf{r}_d] $$
        where $\odot$ is the Hadamard product [S1].
2.  **Network Architectures:**
    *   $F_{\theta_c}$ (Coarse Head): 6 fully connected layers, 256 neurons, ELU activations [S1].
    *   $F_{\theta_f}$ (Refinement Head): 6 fully connected layers, 256 neurons, ELU activations [S1].
    *   $F_{\theta_s}$ (Shading Network): 8 fully connected layers, 256 neurons (based on DoNeRF) [S1].
    *   $F_{\theta_m}$ (Pruning Head, optional): Same complexity as $F_{\theta_c}$, used for ProNeRF-8 prune [S1].

### Numbered Steps

1.  **Ray Encoding:** Compute $\mathbf{r}_{pr}$ from $\mathbf{r}_o, \mathbf{r}_d, t_n, t_f$ [S1].
2.  **Coarse Sampling:** Pass $\mathbf{r}_{pr}$ through $F_{\theta_c}$ to obtain:
    *   Coarse distances $\mathbf{T}' = \{t'_1, \dots, t'_{N_s}\}$ [S1].
    *   Opacity shifts $\Delta_t$ and scales $B_t$ [S1].
    *   Auxiliary coarse color $\hat{\mathbf{c}}_c$ [S1].
3.  **Color-to-Ray Projection:** For each coarse point $\mathbf{p}_i = \mathbf{r}(t'_i)$:
    *   Trace lines from $\mathbf{p}_i$ to the camera centers of $N_n$ neighboring reference views [S1].
    *   Collect the colors $\mathbf{c}^k_{p_i}$ from these views [S1].
    *   Form the projection vector $\mathbf{f}_{p_i} = [\mathbf{c}^1_{p_i}, \dots, \mathbf{c}^{N_n}_{p_i}]$ [S1].
4.  **Refinement:** Concatenate $\mathbf{r}_{pr}$ with all projection vectors $\mathbf{f}_{p_1}, \dots, \mathbf{f}_{p_{N_s}}$ and pass through $F_{\theta_f}$ to obtain:
    *   Refinement weights $\Delta_{\mathbf{T}}$ [S1].
    *   Auxiliary weights $\mathbf{W}$ (softmax) and $\mathbf{M}$ (sigmoid) [S1].
5.  **Fine Point Calculation:** Compute refined distances $\mathbf{T}$ by linear interpolation between consecutive elements of the expanded set $\dot{\mathbf{T}} = \{t_n, t'_1, \dots, t'_{N_s}, t_f\}$ using $\Delta_{\mathbf{T}}$ [S1]:
    $$ \mathbf{T} = \left\{ \frac{1}{2} \left( (\dot{\mathbf{T}}_i + \dot{\mathbf{T}}_{i+1}) + \Delta_{\mathbf{T}_i}(\dot{\mathbf{T}}_{i+2} - \dot{\mathbf{T}}_i) \right) \right\}_{i=1}^{N_s} $$
    Note: The source formula indices imply a specific alignment; implementation should ensure $\Delta_{\mathbf{T}}$ has length $N_s$ and maps to the intervals defined by $\dot{\mathbf{T}}$ [S1].
6.  **Auxiliary Light-Field Rendering (AVR):** Compute the auxiliary fine color $\hat{\mathbf{c}}_f$ using Approximated Volumetric Rendering (AVR) [S1]:
    *   For each view $k \in \{1, \dots, N_n\}$:
        $$ \hat{\mathbf{c}}_{avr}^k = \sum_{i=1}^{N_s} W_i \mathbf{c}^k_{p_i} $$
    *   Aggregate across views:
        $$ \hat{\mathbf{c}}_f = \sum_{k=1}^{N_n} M_k \hat{\mathbf{c}}_{avr}^k $$
    [S1].
7.  **Shading Network Query:** Query $F_{\theta_s}$ at the refined points $\mathbf{T}$ to get densities $\sigma_i$ and colors $\mathbf{c}_i$ for $i=1 \dots N_s$ [S1].
8.  **Opacity Adjustment:** Adjust opacities $\alpha_i$ to account for subsampling. The standard opacity is $\alpha_i = 1 - e^{-\sigma_i(t_{i+1}-t_i)}$. ProNeRF uses $\Delta_t$ and $B_t$ from step 2 to adjust $\alpha_i$ (denoted as $\hat{\alpha}_i$ in the final rendering equation) to compensate for the transmittance accumulated in the full ray distribution learned during exploration [S1].
9.  **Final Rendering:** Render the ray color using alpha compositing with adjusted opacities [S1]:
    $$ \hat{\mathbf{c}}(\mathbf{r}) = \sum_{i=1}^{N_s} \left( \prod_{j=1}^{i-1} 1 - \hat{\alpha}_j \right) \hat{\alpha}_i \mathbf{c}_i $$
    [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

class ProNeRF:
    def __init__(self, N_s, N_pr=48, N_n=8, t_n=0.2, t_f=6.0):
        """
        Initialize ProNeRF networks.
        :param N_s: Number of fine samples (e.g., 8, 12, 32).
        :param N_pr: Number of points for Plucker encoding.
        :param N_n: Number of neighboring reference views.
        :param t_n: Near bound.
        :param t_f: Far bound.
        """
        self.N_s = N_s
        self.N_pr = N_pr
        self.N_n = N_n
        self.t_n = t_n
        self.t_f = t_f
        
        # Network definitions (simplified structure)
        self.F_c = CoarseHead()
        self.F_f = RefinementHead()
        self.F_s = ShadingNetwork()
        self.F_m = PruningHead() # Optional, for ProNeRF-8 prune
        
        # Reference view colors are provided at inference time
        # In training, these are sampled from the training set
        
    def encode_ray(self, r_o, r_d):
        """
        Compute Plucker ray-point encoding.
        :param r_o: Ray origin [3]
        :param r_d: Ray direction [3]
        :return: Encoded ray vector [3 * N_pr]
        """
        # N_pr points evenly spaced between t_n and t_f
        t_nf = torch.linspace(self.t_n, self.t_f, self.N_pr, device=r_o.device)
        # Expand dimensions for broadcasting
        r_o = r_o.unsqueeze(1) # [1, 3]
        r_d = r_d.unsqueeze(1) # [1, 3]
        t_nf = t_nf.unsqueeze(0).unsqueeze(0) # [1, 1, N_pr]
        
        # r_o + r_d * t_nf
        p = r_o + r_d * t_nf # [1, 3, N_pr]
        
        # Hadamard product
        r_d_exp = r_d.unsqueeze(2) # [1, 3, 1]
        p_r_d = p * r_d_exp # [1, 3, N_pr]
        
        # Concatenate [r_d, p, p * r_d]
        # r_d needs to be broadcasted to [1, 3, N_pr]
        r_d_full = r_d.unsqueeze(2).expand(-1, -1, self.N_pr)
        
        r_pr = torch.cat([r_d_full, p, p_r_d], dim=1) # [1, 3*N_pr]
        return r_pr.squeeze(0)

    def forward(self, r_o, r_d, ref_colors):
        """
        Forward pass for rendering.
        :param r_o: Ray origin [3]
        :param r_d: Ray direction [3]
        :param ref_colors: Colors from N_n reference views at projected points [N_s, N_n, 3]
        :return: Rendered color [3]
        """
        r_pr = self.encode_ray(r_o, r_d)
        
        # 1. Coarse Sampling
        # F_c outputs: T_prime, Delta_t, B_t, c_hat_c
        # Note: The exact output tensor shapes depend on implementation details not fully specified
        # Assuming F_c returns tensors of appropriate shapes
        coarse_out = self.F_c(r_pr)
        T_prime = coarse_out['T_prime'] # [N_s]
        Delta_t = coarse_out['Delta_t'] # [N_s]
        B_t = coarse_out['B_t'] # [N_s]
        c_hat_c = coarse_out['c_hat_c'] # [3]
        
        # 2. Refinement
        # Concatenate r_pr with ref_colors
        # ref_colors shape: [N_s, N_n, 3] -> flatten to [N_s, N_n*3]
        ref_flat = ref_colors.view(ref_colors.size(0), -1)
        input_refinement = torch.cat([r_pr, ref_flat], dim=-1)
        
        ref_out = self.F_f(input_refinement)
        Delta_T = ref_out['Delta_T'] # [N_s]
        W = ref_out['W'] # [N_s] (softmax)
        M = ref_out['M'] # [N_n] (sigmoid)
        
        # 3. Fine Point Calculation
        # Expanded set T_dot = [t_n, t'_1, ..., t'_Ns, t_f]
        # Size: N_s + 2
        T_dot = torch.cat([torch.tensor([self.t_n], device=r_o.device), T_prime, torch.tensor([self.t_f], device=r_o.device)])
        
        # Refined T calculation
        # Formula: T_i = 0.5 * ( (T_dot_i + T_dot_{i+1}) + Delta_T_i * (T_dot_{i+2} - T_dot_i) )
        # Indices: i from 0 to N_s-1
        # T_dot indices: i, i+1, i+2
        T_i = T_dot[:-2]
        T_i_plus_1 = T_dot[1:-1]
        T_i_plus_2 = T_dot[2:]
        
        T = 0.5 * ( (T_i + T_i_plus_1) + Delta_T * (T_i_plus_2 - T_i) )
        
        # 4. Auxiliary Light-Field Rendering (AVR)
        # c_avr^k = sum_i W_i * c_p_i^k
        # ref_colors shape: [N_s, N_n, 3]
        # W shape: [N_s]
        # c_avr_k shape: [N_n, 3]
        c_avr_k = torch.sum(W.unsqueeze(1).unsqueeze(2) * ref_colors, dim=0)
        
        # c_hat_f = sum_k M_k * c_avr_k
        # M shape: [N_n]
        # c_hat_f shape: [3]
        c_hat_f = torch.sum(M.unsqueeze(1) * c_avr_k, dim=0)
        
        # 5. Shading Network Query
        # Query F_s at T
        # T shape: [N_s]
        # Points: r_o + r_d * T
        points = r_o.unsqueeze(0) + r_d.unsqueeze(0) * T.unsqueeze(1) # [N_s, 3]
        sigma, c = self.F_s(points) # sigma: [N_s], c: [N_s, 3]
        
        # 6. Opacity Adjustment
        # Standard alpha
        dt = T[1:] - T[:-1]
        # Handle boundary if necessary, assuming uniform or last point handled
        # Standard NeRF alpha: alpha_i = 1 - exp(-sigma_i * (t_{i+1} - t_i))
        alpha = 1.0 - torch.exp(-sigma * dt)
        
        # Adjusted alpha using Delta_t and B_t from coarse head
        # The source mentions adjusting alpha_i using Delta_t and B_t to compensate for subsampled transmittance
        # Exact formula for adjustment is not explicitly given as a simple algebraic expression in the notes,
        # but implies: alpha_hat_i = function(alpha_i, Delta_t_i, B_t_i)
        # Common adjustment in such methods is scaling or shifting.
        # Given the note "adjusting alpha_i in Eq. 4 is needed... learned for the full ray distribution",
        # and Eq 4 is standard alpha, we apply the learned parameters.
        # A plausible implementation based on "shifts and scales":
        # alpha_hat = B_t * alpha + Delta_t ? Or similar.
        # However, without explicit formula, we use the standard alpha for rendering 
        # if the adjustment mechanism is not fully detailed in the provided text snippets.
        # [G] Assuming the adjustment is applied to the opacity values before compositing.
        # For implementation completeness, we use the standard alpha if adjustment is ambiguous,
        # but note that ProNeRF explicitly predicts Delta_t and B_t for this purpose.
        # Let's assume a linear adjustment: alpha_hat = B_t * alpha + Delta_t, clamped to [0,1]
        alpha_hat = torch.clamp(B_t * alpha + Delta_t, 0.0, 1.0)
        
        # 7. Final Rendering
        # Transmittance T_i = product_{j=1}^{i-1} (1 - alpha_hat_j)
        # Using cumulative product
        one_minus_alpha = 1.0 - alpha_hat
        transmittance = torch.cumprod(one_minus_alpha, dim=0)
        # Shift transmittance: T_0 = 1, T_i = product_{j=0}^{i-1} (1 - alpha_j)
        # Standard compositing: c = sum alpha_i * c_i * T_{i-1}
        # T_shifted[0] = 1, T_shifted[i] = T[i-1]
        T_shifted = torch.cat([torch.tensor([1.0], device=r_o.device), transmittance[:-1]])
        
        c_rendered = torch.sum(alpha_hat.unsqueeze(1) * c * T_shifted.unsqueeze(1), dim=0)
        
        return c_rendered, c_hat_c, c_hat_f

    def train_step(self, rays, gt_colors, ref_colors, iteration):
        """
        Training step with Exploration/Exploitation strategy.
        """
        # PAS outputs
        r_o, r_d = rays['origin'], rays['direction']
        c_gt = gt_colors
        
        # Forward PAS
        # This calls the forward method but we need intermediate outputs
        # Re-implementing intermediate steps for loss calculation
        r_pr = self.encode_ray(r_o, r_d)
        coarse_out = self.F_c(r_pr)
        T_prime = coarse_out['T_prime']
        Delta_t = coarse_out['Delta_t']
        B_t = coarse_out['B_t']
        c_hat_c = coarse_out['c_hat_c']
        
        ref_flat = ref_colors.view(ref_colors.size(0), -1)
        input_refinement = torch.cat([r_pr, ref_flat], dim=-1)
        ref_out = self.F_f(input_refinement)
        Delta_T = ref_out['Delta_T']
        W = ref_out['W']
        M = ref_out['M']
        
        T_dot = torch.cat([torch.tensor([self.t_n], device=r_o.device), T_prime, torch.tensor([self.t_f], device=r_o.device)])
        T_i = T_dot[:-2]
        T_i_plus_1 = T_dot[1:-1]
        T_i_plus_2 = T_dot[2:]
        T = 0.5 * ( (T_i + T_i_plus_1) + Delta_T * (T_i_plus_2 - T_i) )
        
        c_avr_k = torch.sum(W.unsqueeze(1).unsqueeze(2) * ref_colors, dim=0)
        c_hat_f = torch.sum(M.unsqueeze(1) * c_avr_k, dim=0)
        
        # Exploration vs Exploitation
        if iteration % 2 == 0 and iteration < 400000:
            # Exploration Pass
            # Randomly interpolate N_s distances in T into N_s+ evenly-spaced distances T+
            # Add Gaussian noise
            N_s_plus = torch.randint(self.N_s, 64, (1,)).item() # N is 64
            # Sample N_s+ points around T
            # [G] Implementation detail: "Randomly interpolate... into evenly-spaced distances"
            # This implies generating a new set of samples T+ based on T, with noise.
            # For simplicity, we assume T+ is a perturbed version of T with different count.
            # Since exact sampling logic is not fully detailed, we use T for shading query 
            # but note that exploration uses dense sampling.
            # The source says: "draw a random number of samples... around the estimated T"
            # We will use T for the shading network query in this simplified pseudocode,
            # but in reality, T+ would be used.
            points = r_o.unsqueeze(0) + r_d.unsqueeze(0) * T.unsqueeze(1)
            sigma, c = self.F_s(points)
            
            dt = T[1:] - T[:-1]
            alpha = 1.0 - torch.exp(-sigma * dt)
            
            # Standard VRE
            one_minus_alpha = 1.0 - alpha
            transmittance = torch.cumprod(one_minus_alpha, dim=0)
            T_shifted = torch.cat([torch.tensor([1.0], device=r_o.device), transmittance[:-1]])
            c_rendered = torch.sum(alpha.unsqueeze(1) * c * T_shifted.unsqueeze(1), dim=0)
            
            loss = torch.mean((c_rendered - c_gt) ** 2)
            
            # Update only F_s (Opts)
            # [G] Optimizer logic omitted for brevity, assume Opts.step() updates F_s
            loss.backward()
            Opts.step()
            Opts.zero_grad()
            
        else:
            # Exploitation Pass
            points = r_o.unsqueeze(0) + r_d.unsqueeze(0) * T.unsqueeze(1)
            sigma, c = self.F_s(points)
            
            dt = T[1:] - T[:-1]
            alpha = 1.0 - torch.exp(-sigma * dt)
            alpha_hat = torch.clamp(B_t * alpha + Delta_t, 0.0, 1.0)
            
            one_minus_alpha = 1.0 - alpha_hat
            transmittance = torch.cumprod(one_minus_alpha, dim=0)
            T_shifted = torch.cat([torch.tensor([1.0], device=r_o.device), transmittance[:-1]])
            c_rendered = torch.sum(alpha_hat.unsqueeze(1) * c * T_shifted.unsqueeze(1), dim=0)
            
            loss = torch.mean((c_rendered - c_gt) ** 2)
            
            if iteration < 400000:
                # Auxiliary losses
                loss_c = torch.mean((c_hat_c - c_gt) ** 2)
                loss_f = torch.mean((c_hat_f - c_gt) ** 2)
                loss = loss + loss_c + loss_f
                
            # Update all networks (Opt_cfs)
            loss.backward()
            Opt_cfs.step()
            Opt_cfs.zero_grad()

class CoarseHead(nn.Module):
    def __init__(self):
        super().__init__()
        self.layers = nn.Sequential(
            nn.Linear(144, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256) # Output layer
        )
        # Outputs: T_prime (N_s), Delta_t (N_s), B_t (N_s), c_hat_c (3)
        self.out_layer = nn.Linear(256, 3 * self.N_s + 3 * self.N_s + 3 * self.N_s + 3)
        
    def forward(self, x):
        out = self.layers(x)
        out = self.out_layer(out)
        # Split output
        T_prime = out[:, :self.N_s]
        Delta_t = out[:, self.N_s:self.N_s*2]
        B_t = out[:, self.N_s*2:self.N_s*3]
        c_hat_c = out[:, self.N_s*3:]
        return {'T_prime': T_prime, 'Delta_t': Delta_t, 'B_t': B_t, 'c_hat_c': c_hat_c}

class RefinementHead(nn.Module):
    def __init__(self):
        super().__init__()
        self.layers = nn.Sequential(
            nn.Linear(144 + self.N_s * self.N_n * 3, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256)
        )
        self.out_layer = nn.Linear(256, self.N_s + self.N_s + self.N_n)
        
    def forward(self, x):
        out = self.layers(x)
        out = self.out_layer(out)
        Delta_T = out[:, :self.N_s]
        W = out[:, self.N_s:self.N_s*2]
        M = out[:, self.N_s*2:]
        return {'Delta_T': Delta_T, 'W': torch.softmax(W, dim=-1), 'M': torch.sigmoid(M)}

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
            nn.Linear(256, 4) # sigma (1), color (3)
        )
        
    def forward(self, x):
        out = self.layers(x)
        sigma = out[:, :1]
        c = out[:, 1:]
        return sigma, c

class PruningHead(nn.Module):
    def __init__(self):
        super().__init__()
        self.layers = nn.Sequential(
            nn.Linear(144, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256), nn.ELU(),
            nn.Linear(256, 256)
        )
        self.out_layer = nn.Linear(256, 1)
        
    def forward(self, x):
        out = self.layers(x)
        error = self.out_layer(out)
        return error
```

### Parameters
*   **Reference Views ($N_n$):** A small, fixed subset of reference views from the training set. The total number of training views $N_t$ can be large, but only $N_n$ are used for projection [S1].
*   **Training Iterations:** $7 \times 10^5$ [S1].
*   **Auxiliary Loss Duration:** Active for the first $4 \times 10^5$ iterations (approx. 57%) [S1].
*   **Lambda ($\lambda$):** Weight for auxiliary loss. Set to 1 for the first 60% of training, then 0 [S1].
*   **Batch Size ($N_r$):** 4,096 rays [S1].
*   **Learning Rate:** Initial $5 \times 10^{-4}$, exponentially decayed over 700K iterations [S1].
*   **Plücker Points ($N_{pr}$):** 48 [S1].
*   **Max Exploration Samples ($N$):** 64 [S1].
*   **Pruning Memory Overhead:** 3.3 MB [S1].

### Edge Cases and Pitfalls
*   **Precision:** The method uses FP16 for testing on RTX 3090, but training likely uses FP32 or mixed precision. Precision issues may arise in the cumulative product for transmittance if not handled carefully [S1].
*   **Boundary Conditions:** The expanded set $\dot{\mathbf{T}}$ includes $t_n$ and $t_f$. The refinement formula uses indices $i, i+1, i+2$. Care must be taken to ensure $\Delta_{\mathbf{T}}$ aligns correctly with the intervals [S1].
*   **Degenerate Input:** If reference views are not available or $N_n=0$, the projection step fails. The method assumes a calibrated multi-view setup [S1].

## Operating characteristics
*   **Speed:** ProNeRF is 15–23× faster than vanilla NeRF [S1]. ProNeRF-8 is >20× faster, ProNeRF-12 is ~15× faster [S1]. ProNeRF-8 prune is 23% faster than standard ProNeRF-8 [S1].
*   **Quality:** ProNeRF-12 achieves 0.65 dB higher PSNR than NeRF and 0.95 dB higher than HyperReel on LLFF [S1]. ProNeRF-8 achieves 26.78 dB PSNR on LLFF, outperforming NeRF (26.50 dB) by 0.28 dB [S1].
*   **Memory:** Similar to NeRF (3.8 MB for NeRF, ProNeRF-8 prune adds 3.3 MB overhead) [S1].
*   **Strengths:** High quality with few samples, consistent memory footprint regardless of training view count, fast inference [S1].
*   **Limitations:** Requires a fixed subset of reference views, training is complex due to alternating exploration/exploitation [S1].
*   **When to use:** When rendering speed is critical but quality must remain close to or better than vanilla NeRF, and memory is constrained [S1].
*   **When not to use:** When a large number of training views are available and can be efficiently utilized (generalizable methods might be better), or when extreme sparsity is not required [S1].

## Background
ProNeRF was proposed in 2024 [S1]. It builds on the principles of sparse and efficient representations, specifically addressing the limitations of sampler-based methods like HyperReel which suffer from quality degradation [S1]. It differs from generalizable radiance fields like IBRNet and NeuRay by using a fixed subset of reference views instead of storing all training images [S1]. It is evaluated on the LLFF and Blender datasets [S1].

## Sources
[S1] ProNeRF: Learning Efficient Projection-Aware Ray Sampling for Fine-Grained Implicit Neural Radiance Fields, https://doi.org/10.1109/access.2024.3390753, fetched 2026-10-30
