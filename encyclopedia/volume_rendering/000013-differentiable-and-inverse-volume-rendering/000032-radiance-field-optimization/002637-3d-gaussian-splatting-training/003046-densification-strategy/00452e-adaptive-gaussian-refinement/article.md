# Adaptive Gaussian Refinement

Also called: Residual Correction Module, 7DGS Refinement

## Overview
Adaptive Gaussian Refinement is a technique that dynamically adjusts Gaussian parameters using neural network-predicted residuals to improve the modeling of complex non-rigid deformations and time-varying appearance [S1]. It addresses the limitation of static covariance matrices in standard 3D Gaussian Splatting, which cannot adequately represent dynamic behaviors such as motion-induced shape changes [S1]. Unlike sibling methods that rely on geometric heuristics or gradient norms for densification, this method uses lightweight MLPs to predict continuous corrections to position, time, direction, and covariance based on a contextual feature vector [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** The original 7D Gaussian parameters: spatial mean $\mu_p$, temporal coordinate $\mu_t$, directional mean $\mu_d$, and the vectorized lower-triangular elements of the 7D covariance matrix $l$. The observed time $t$ and view direction $d$ are also required for the feature construction and subsequent slicing, though the refinement itself primarily consumes the Gaussian state and time.
*   **Outputs:** Refined parameters $\hat{\mu}_p, \hat{\mu}_t, \hat{\mu}_d, \hat{l}$.

**Data Structures:**
*   **Feature Vector $f$:** A concatenated vector formed by $\mu_p$, $\mu_t$, $\mu_d$, and the temporal encoding $\gamma(t)$.
*   **MLPs:** Four separate two-layer multilayer perceptrons, one for each parameter group ($\phi_p, \phi_t, \phi_d, \phi_l$). Each MLP has the architecture $C_{in} \times 64 \times C_{out}$, where $C_{in}$ is the dimension of $f$ and $C_{out}$ is the dimension of the respective parameter (3 for means, 28 for the lower-triangular covariance vector) [S1].

**Steps:**
1.  **Construct Feature Vector:** Concatenate the spatial mean, temporal coordinate, directional mean, and high-frequency temporal encoding:
    $$f = \mu_p \oplus \mu_t \oplus \mu_d \oplus \gamma(t)$$
    where $\oplus$ denotes vector concatenation [S1].
2.  **Compute Temporal Encoding:** Generate $\gamma(t)$ using positional encoding with $K=10$ frequencies:
    $$\gamma(t) = [\sin(2^0 \pi t), \cos(2^0 \pi t), \dots, \sin(2^{K-1} \pi t), \cos(2^{K-1} \pi t)]$$
    This provides a rich representation capturing both low-frequency trends and high-frequency details [S1].
3.  **Predict Residuals:** Pass $f$ through the respective MLPs to predict residuals for position, time, direction, and covariance:
    $$r_p = \phi_p(f), \quad r_t = \phi_t(f), \quad r_d = \phi_d(f), \quad r_l = \phi_l(f)$$
4.  **Apply Refinement:** Add the residuals to the original parameters to obtain the refined estimates:
    $$\hat{\mu}_p = \mu_p + r_p$$
    $$\hat{\mu}_t = \mu_t + r_t$$
    $$\hat{\mu}_d = \mu_d + r_d$$
    $$\hat{l} = l + r_l$$
    Here, $l$ represents the vectorized lower-triangular elements of the 7D covariance matrix [S1].
5.  **Reconstruct Covariance:** Compute the refined covariance matrix $\hat{\Sigma}$ from $\hat{l}$:
    $$\hat{\Sigma} = \hat{L}\hat{L}^\top$$
    where $\hat{L}$ is the matrix reconstructed from $\hat{l}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class AdaptiveGaussianRefinement:
    def __init__(self, dim_mu=3, dim_cov=28, hidden_dim=64, num_freqs=10):
        """
        Initialize the refinement module.
        dim_mu: Dimension of mean vectors (3 for spatial, 1 for time, 3 for direction).
        dim_cov: Dimension of vectorized lower-triangular covariance (28 for 7x7 matrix).
        hidden_dim: Hidden layer size for MLPs.
        num_freqs: Number of frequency bands for positional encoding.
        """
        self.num_freqs = num_freqs
        # Input dimension: 3 (mu_p) + 1 (mu_t) + 3 (mu_d) + 2 * num_freqs (gamma)
        self.C_in = 7 + 2 * num_freqs
        self.dim_cov = dim_cov
        
        # Define MLPs for each parameter group
        # Architecture: C_in -> hidden_dim -> C_out
        self.mlp_p = nn.Sequential(
            nn.Linear(self.C_in, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, 3) # Output dim for mu_p
        )
        self.mlp_t = nn.Sequential(
            nn.Linear(self.C_in, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, 1) # Output dim for mu_t
        )
        self.mlp_d = nn.Sequential(
            nn.Linear(self.C_in, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, 3) # Output dim for mu_d
        )
        self.mlp_l = nn.Sequential(
            nn.Linear(self.C_in, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, dim_cov) # Output dim for l
        )

    def temporal_encoding(self, t):
        """
        Compute high-frequency temporal encoding gamma(t).
        t: Tensor of shape (..., 1) containing time values.
        Returns: Tensor of shape (..., 2 * num_freqs)
        """
        # t is expected to be normalized or raw time; formula uses 2^k * pi * t
        # We assume t is a scalar or batched tensor.
        # Reshape t to broadcast correctly
        t = t.unsqueeze(-1) # (..., 1)
        
        # Generate frequencies: 2^0, 2^1, ..., 2^(K-1)
        frequencies = torch.pow(2.0, torch.arange(self.num_freqs, device=t.device).float())
        # Shape: (num_freqs,)
        
        # Compute arguments: frequencies * pi * t
        # Shape: (..., num_freqs)
        args = frequencies * torch.pi * t
        
        # Compute sin and cos
        # args shape: (..., num_freqs)
        sin_vals = torch.sin(args)
        cos_vals = torch.cos(args)
        
        # Concatenate along the last dimension
        # Result shape: (..., 2 * num_freqs)
        gamma = torch.cat([sin_vals, cos_vals], dim=-1)
        return gamma

    def forward(self, mu_p, mu_t, mu_d, l, t):
        """
        Perform adaptive Gaussian refinement.
        
        Args:
            mu_p: Spatial mean, shape (..., 3)
            mu_t: Temporal coordinate, shape (..., 1)
            mu_d: Directional mean, shape (..., 3)
            l: Vectorized lower-triangular covariance elements, shape (..., 28)
            t: Observed time, shape (..., 1)
            
        Returns:
            hat_mu_p, hat_mu_t, hat_mu_d, hat_l: Refined parameters
        """
        # 1. Construct feature vector f
        gamma_t = self.temporal_encoding(t)
        f = torch.cat([mu_p, mu_t, mu_d, gamma_t], dim=-1)
        
        # 2. Predict residuals
        r_p = self.mlp_p(f)
        r_t = self.mlp_t(f)
        r_d = self.mlp_d(f)
        r_l = self.mlp_l(f)
        
        # 3. Apply residuals
        hat_mu_p = mu_p + r_p
        hat_mu_t = mu_t + r_t
        hat_mu_d = mu_d + r_d
        hat_l = l + r_l
        
        return hat_mu_p, hat_mu_t, hat_mu_d, hat_l
```

**Parameters:**
*   $K=10$: Number of frequency bands in temporal encoding [S1].
*   Hidden dimension: 64 [S1].
*   MLP Architecture: Two layers ($C_{in} \to 64 \to C_{out}$) [S1].

**Edge Cases and Pitfalls:**
*   **Covariance Validity:** The refinement adds a residual $r_l$ to the lower-triangular elements. The resulting matrix $\hat{L}$ must still yield a positive-definite covariance matrix $\hat{\Sigma} = \hat{L}\hat{L}^\top$. If the residuals are large, numerical instability may occur. The sources do not specify a projection step to enforce positive definiteness, implying the MLPs are trained to produce small, stable residuals [S1].
*   **Temporal Continuity:** The use of sinusoidal positional encoding assumes periodicity or bounded time ranges. If $t$ is unbounded, the encoding may not generalize well without normalization [S1].

## Operating characteristics
*   **Computational Cost:** Adds the cost of four small MLP evaluations per Gaussian per frame. The MLPs are lightweight ($C_{in} \times 64 \times C_{out}$), so the overhead is minimal compared to the rendering cost [S1].
*   **Memory:** Requires storing the weights of the four MLPs. No additional per-Gaussian storage is needed beyond the parameters themselves [S1].
*   **Strengths:** Enables accurate modeling of complex non-rigid deformations and time-varying appearance that static Gaussians cannot capture [S1].
*   **Limitations:** Relies on the capacity of the MLPs to learn the residual functions. May overfit if the network is too complex or underfit if too simple [S1].

## Background
Adaptive Gaussian Refinement was introduced in the context of 7D Gaussian Splatting (7DGS) [S1]. 7DGS extends 3D Gaussian Splatting to jointly encode spatial structure, temporal evolution, and angular view dependence [S1]. The method was developed to address the limitation that the intrinsic shape of each Gaussian (determined by its covariance) remains static over time, which hinders representing complex dynamic behaviors [S1]. It was evaluated on synthetic monocular videos (D-NeRF), in-the-wild multi-view videos (Technicolor), and a custom dataset (7DGS-PBR) [S1].

## Sources
[S1] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
