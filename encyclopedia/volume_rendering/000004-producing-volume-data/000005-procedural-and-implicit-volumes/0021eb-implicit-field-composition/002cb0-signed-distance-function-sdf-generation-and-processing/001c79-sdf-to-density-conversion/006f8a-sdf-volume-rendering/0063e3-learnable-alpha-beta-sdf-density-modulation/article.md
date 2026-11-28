# Learnable Alpha/Beta SDF Density Modulation

Also called: SDF Density Modulation, learnable alpha/beta SDF

## Overview
This method modulates volume density in a Signed Distance Function (SDF) representation using two separate learnable parameters, $\alpha$ and $\beta$, rather than tying them equal as in prior work [S1]. It solves the convergence instability issues found in methods that use a single tied parameter by allowing independent optimization of the density scale and shape [S1]. It differs from [VolSDF Volume Rendering](2551-volsdf-volume-rendering.md) by decoupling the $\alpha$ and $\beta$ parameters [S1].

## How it works
**Inputs:**
- 3D point coordinates $x \in \mathbb{R}^3$.
- SDF network predicting $d(x)$.
- Learnable parameters $\alpha, \beta$.

**Outputs:**
- Volume density $\sigma$ derived from $d(x)$, $\alpha$, and $\beta$.

**Data Structures:**
- $\alpha$: Scalar float, initialized to 1.0.
- $\beta$: Scalar float, initialized to 0.1.
- Lower bound clamp: $10^{-3}$ for both parameters.

**Steps:**
1. **SDF Initialization (Pre-training):** Before training the unconditional model, initialize the SDF to a unit sphere. Optimize the network for 1000 iterations using the loss:
   $$ L_{SDF} = \mathbb{E}_x [d(x) - (\|x\| - 1)^2] $$
   where $d(x)$ is the predicted SDF value at point $x$ [S1].

2. **Density Modulation:** During training, compute the volume density $\sigma$ using the learned $\alpha$ and $\beta$. The specific functional form mapping $d(x)$ to $\sigma$ using $\alpha$ and $\beta$ is not explicitly detailed in the source, but the parameters are used to modulate the density [S1].

3. **Parameter Clamping:** Clamp $\alpha$ and $\beta$ to a lower bound of $10^{-3}$ during training to ensure stability [S1].

4. **Training:** Train the model end-to-end for 120,000 iterations with a batch size of 32 samples using the Adam optimizer. Use an initial learning rate of $6 \times 10^{-5}$, decaying to $6 \times 10^{-6}$ after 60,000 iterations [S1].

5. **Losses:** Use Mean Squared Error (MSE) for the latent code ($L_{latent}$), L1 loss for the segmentation mask ($L_{mask}$), and a masked L2 loss (with square root) for the canonical map ($L_{map}$) [S1]. Apply an Eikonal loss with a weight of 0.1 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class SDFDensityModulation:
    def __init__(self, sdf_network):
        self.sdf_network = sdf_network
        # Initialize parameters as per [S1]
        self.alpha = nn.Parameter(torch.tensor(1.0))
        self.beta = nn.Parameter(torch.tensor(0.1))
        self.lower_bound = 1e-3
        
    def pretrain_sdf(self, optimizer, num_iterations=1000):
        """Initialize SDF to a unit sphere [S1]."""
        for _ in range(num_iterations):
            # Sample points x
            x = sample_points() 
            d_x = self.sdf_network(x)
            # Loss: E_x [d(x) - (||x|| - 1)^2]
            loss = torch.mean((d_x - (torch.norm(x, dim=1) - 1)**2)**2)
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()

    def compute_density(self, x):
        """Compute density using learnable alpha and beta [S1]."""
        d_x = self.sdf_network(x)
        # Clamp parameters for stability [S1]
        alpha_clamped = torch.clamp(self.alpha, min=self.lower_bound)
        beta_clamped = torch.clamp(self.beta, min=self.lower_bound)
        
        # Note: The exact formula for sigma(d, alpha, beta) is not provided in [S1].
        # This step represents the modulation process described.
        # [G] Assuming a standard modulation form like sigma = alpha * exp(-beta * d^2) 
        # or similar, but the source does not specify the exact equation.
        # The source only states that density is modulated by alpha, beta > 0.
        sigma = self._modulate_density(d_x, alpha_clamped, beta_clamped)
        return sigma

    def _modulate_density(self, d, alpha, beta):
        """Placeholder for the specific density modulation function.
        The source [S1] does not provide the explicit equation for sigma(d, alpha, beta).
        It only specifies the parameters and their initialization/clamping."""
        # [G] Implementation detail missing in source. 
        # A common approach in such contexts is sigma = alpha * exp(-beta * d^2) 
        # or sigma = alpha * sigmoid(-beta * d), but this is not confirmed by [S1].
        return alpha * torch.exp(-beta * d**2)

    def train_step(self, optimizer, batch):
        """Perform one training step [S1]."""
        x, y = batch
        sigma = self.compute_density(x)
        
        # Compute losses [S1]
        loss_latent = ... # MSE for latent code
        loss_mask = ...   # L1 for segmentation mask
        loss_map = ...    # Masked L2 for canonical map
        loss_eikonal = ... # Eikonal loss with weight 0.1
        
        total_loss = loss_latent + loss_mask + loss_map + 0.1 * loss_eikonal
        
        optimizer.zero_grad()
        total_loss.backward()
        optimizer.step()
```

## Subtopics

### Learnable Inverse Temperature (inv_s) Modulation
A network that outputs a single scalar inverse temperature ($inv\_s$) parameter, used to convert SDF values into volume rendering alphas via sigmoid functions [S2, unofficial]. The $inv\_s$ is computed as $\exp(\text{variance} \times 10.0)$. If modulation is enabled and active, it is clamped to $mod\_val$ [S2, unofficial]. During the update step, if modulation is enabled: if $\text{global\_step} > \text{mod\_start\_steps}$, set $\text{do\_mod} = \text{True}$. Calculate $mod\_val = \min((\text{global\_step} / \text{reach\_max\_steps}) \times (\text{max\_inv\_s} - \text{prev\_inv\_s}) + \text{prev\_inv\_s}, \text{max\_inv\_s})$ [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
- **Strengths:** Improves convergence compared to tying $\alpha = \beta$ [S1].
- **Limitations:** Requires pre-training for SDF initialization [S1].

**When to Use:**
Use when training SDF-based volume rendering models and experiencing convergence issues with tied density parameters [S1].

## Background
Originated in "Shape, Pose, and Appearance from a Single Image via Bootstrapped Radiance Field Inversion" [S1]. The method was introduced to address convergence issues in SDF representations by allowing separate learning of density modulation parameters.

## Sources
[S1] Shape, Pose, and Appearance from a Single Image via Bootstrapped Radiance Field Inversion, https://doi.org/10.1109/cvpr52729.2023.00427, fetched 2026-10-26
[S2] dehezhang2/RISE-SDF, https://github.com/dehezhang2/RISE-SDF, fetched 2026-10-27 (unofficial)
