# Semi-Gradient Method

Also called: Ours, Ours w/o VR, Partial derivative-based optimization, SG method, Semi-Gradient Estimator

## Overview
The Semi-Gradient Method is a training optimization technique for neural radiance field caching that modifies the baseline loss function by applying a stop-gradient operation to the right-hand side (RHS) of the rendering equation [S1, unofficial]. It solves the problem of gradient bias and high variance inherent in standard differentiable rendering optimization, where the correlation between the network gradient and the radiance estimate impedes convergence [S1, unofficial]. It sets itself apart from the baseline by intentionally ignoring updates to the RHS, thereby removing the partial derivatives with respect to the RHS to achieve unbiased and low-variance gradient estimates [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** Network parameters $\theta$, learning rate $\eta$, location vectors $\{x_j\}$, outgoing directions $\{\omega_j\}$, incident directions $\{\omega_{in,j,k}\}$, and reference radiance values.
*   **Outputs:** Updated network parameters $\theta$.

**Data Structures**
*   **Radiance Prediction Network:** A multi-layer perceptron (MLP) consisting of seven linear layers, each with 512 units. Six ReLU activations are used; the final output layer does not pass through an activation function [S1, unofficial].
*   **Encoding:** Multi-resolution hash encoding (from InstantNGP) with base resolution 2, 14 hierarchy levels, 2 features per level, and a hashmap size of $2^{18}$ [S1, unofficial].
*   **Precision:** Mixed precision is used. Inputs $(x, \omega)$ are processed in 32-bit full precision, converted to 16-bit half-precision for the network, and outputs are converted back to full precision before the rendering engine [S1, unofficial].

**Parameters**
*   **Learning Rate ($\eta$):** $5 \times 10^{-4}$, reduced by a factor of three every 12,000 steps [S1, unofficial].
*   **Incident Sample Count ($M$):** 32 incident directions sampled per $(x, \omega)$ [S1, unofficial].
*   **Batch Size ($N$):** 214 samples of $(x, \omega)$ [S1, unofficial].
*   **Training Steps:** 36,000 steps per scene [S1, unofficial].
*   **Optimizer:** Adam [S1, unofficial].
*   **Initialization:** Xavier uniform distribution [S1, unofficial].

**Algorithm**
The method replaces the standard L2 loss gradient with a semi-gradient estimator. The baseline minimizes the L2 distance between the left-hand side (LHS, outgoing radiance $L_\theta$) and the right-hand side (RHS, sum of emitted and reflected radiance $\langle R_\theta \rangle$). The semi-gradient loss $\langle L(\theta) \rangle_{SG}$ is defined as:

$$
\langle L(\theta) \rangle_{SG} = \frac{\| L_\theta - \text{sg}(\langle R_\theta \rangle) \|^2}{\| \text{sg}(L_\theta) \|^2 + \epsilon}
$$

where $\text{sg}(\cdot)$ is the stop-gradient operator, which blocks the flow of gradients through its argument [S1, unofficial]. The term $\epsilon$ is a small constant for numerical stability [G].

The gradient estimator for this loss eliminates the partial derivative with respect to the RHS:

$$
\frac{\partial \langle R_\theta \rangle}{\partial \langle L(\theta) \rangle_{SG}} = 0
$$

This prevents the network from pulling the RHS estimates towards the LHS, a behavior identified as the source of bias and high variance in the baseline [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class RadianceNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        # Seven linear layers, 512 units each
        self.layers = nn.ModuleList([
            nn.Linear(512, 512) for _ in range(6)
        ] + [nn.Linear(512, 512)]) # Last layer output
        self.relu = nn.ReLU()
        
    def forward(self, x):
        h = x
        for i, layer in enumerate(self.layers[:-1]):
            h = self.relu(layer(h))
        # Last layer: no activation
        h = self.layers[-1](h)
        return h

def semi_gradient_step(network, optimizer, positions, outgoing_dirs, incident_dirs, reference_radiance):
    """
    Performs one optimization step using the semi-gradient method.
    
    Args:
        network: RadianceNetwork instance.
        optimizer: Adam optimizer instance.
        positions: Tensor of shape (N, 3) location vectors.
        outgoing_dirs: Tensor of shape (N, 3) outgoing directions.
        incident_dirs: Tensor of shape (N, M, 3) incident directions.
        reference_radiance: Tensor of shape (N,) reference radiance values.
        
    Returns:
        loss: Scalar loss value.
    """
    optimizer.zero_grad()
    
    # 1. Evaluate LHS (Outgoing Radiance)
    # In a full implementation, this involves rendering the scene given the network's predictions
    # For the purpose of the gradient definition, we assume L_theta is computed.
    # Note: The actual L_theta computation involves the rendering engine (e.g., Mitsuba 3).
    # Here we simulate the forward pass of the network to get the radiance estimate.
    # The network takes encoded inputs.
    
    # Encode inputs (simplified)
    # encoded_pos = hash_encode(positions)
    # encoded_dir = hash_encode(outgoing_dirs)
    # input_vec = torch.cat([encoded_pos, encoded_dir], dim=-1)
    
    # L_theta: The predicted outgoing radiance (LHS)
    # In the paper, L_theta is the result of the rendering equation using the network's BSDF/radiance cache.
    # For the semi-gradient loss, we need the value of L_theta.
    L_theta = network(input_vec) # Placeholder for actual rendering output
    
    # 2. Evaluate RHS (Emitted + Reflected Radiance)
    # RHS is the sum of emitted radiance and reflected radiance integrated over incident directions.
    # R_theta = E_theta + integral(L_theta * f * cos_theta d_omega)
    # The paper uses Monte Carlo integration with M incident directions.
    
    # Sample incident directions (already provided in incident_dirs)
    # Evaluate radiance at incident directions using the network
    # This requires querying the network for the radiance coming from those directions.
    # Let's assume a helper function `get_radiance_at_direction` exists which queries the network.
    
    # Reflected radiance estimate <R_theta>
    # <R_theta> = sum( L_theta(omega_in) * f(omega_out, omega_in) * cos_theta ) / M
    # The paper simplifies this to a neural cache prediction.
    
    # To implement the semi-gradient, we compute <R_theta> but stop gradients through it.
    with torch.no_grad():
        # Compute RHS estimate without tracking gradients
        # This involves querying the network for incident radiance and integrating.
        # For pseudocode clarity, we assume `compute_rhs_estimate` performs the MC integration
        # using the network's predictions for incident directions.
        R_theta = compute_rhs_estimate(network, positions, outgoing_dirs, incident_dirs)
        
    # 3. Compute Semi-Gradient Loss
    # Loss = || L_theta - sg(<R_theta>) ||^2 / (|| sg(L_theta) ||^2 + epsilon)
    epsilon = 1e-8
    
    # Stop gradient on RHS
    R_theta_sg = R_theta.detach()
    
    # Stop gradient on LHS in the denominator (as per Eq 7 in paper)
    L_theta_sg = L_theta.detach()
    
    numerator = (L_theta - R_theta_sg) ** 2
    denominator = L_theta_sg ** 2 + epsilon
    
    loss = torch.mean(numerator / denominator)
    
    # 4. Backpropagate
    loss.backward()
    
    # 5. Update parameters
    optimizer.step()
    
    return loss.item()

def compute_rhs_estimate(network, positions, outgoing_dirs, incident_dirs):
    """
    Computes the RHS estimate using Monte Carlo integration over incident directions.
    This function is executed within a no_grad context in the main loop.
    """
    # This is a simplified representation.
    # In practice, this involves:
    # 1. Querying the network for radiance at incident directions.
    # 2. Applying the BSDF (Bidirectional Scattering Distribution Function).
    # 3. Integrating over the hemisphere.
    
    # Assuming the network predicts the radiance cache value directly.
    # The incident directions are sampled uniformly over a hemisphere.
    
    # Query network for incident radiance
    # encoded_incident = hash_encode(incident_dirs)
    # incident_radiance = network(encoded_incident)
    
    # Integrate (simplified sum)
    # R_theta = torch.mean(incident_radiance, dim=1)
    
    # Return placeholder
    return torch.zeros_like(positions[:, 0])
```

**Edge Cases and Pitfalls**
*   **Bias in Baseline:** The baseline method is a biased estimator because of the correlation between $\nabla_\theta \langle R_\theta \rangle$ and $\langle R_\theta \rangle$ in the partial derivative with respect to the RHS. This bias impedes convergence to the minimum, causing the reconstruction to appear darker than the reference [S1, unofficial].
*   **Convergence Guarantee:** Due to mathematical shortcomings in the definition of the stop-gradient operation, convergence of the L2-form loss cannot be prematurely guaranteed. Convergence is theoretically guaranteed only for scenes satisfying the energy absorption condition [S1, unofficial].
*   **Implementation Detail:** To realize the speedup, unnecessary intermediate activations should not be stored for the backward pass, e.g., using PyTorch's `no_grad` context manager [S1, unofficial].

## Subtopics

### Variance Reduced Gradient Estimator
A technique to expedite initial convergence by integrating neural path guiding. It uses additional neural caches to sample incident directions, reducing the variance of the Right Hand Side (RHS) estimates and subsequently the variance of the semi-gradient estimator [S1, unofficial]. The path guiding model consists of four linear layers of 256 units with three activations [S1, unofficial]. It is trained alongside radiance caches for 1,000 to 5,000 iterations, then held fixed for the remaining iterations [S1, unofficial]. It operates with next event estimation for multiple importance sampling to reduce variance [S1, unofficial]. This reduction mitigates visual artifacts during early training and accelerates error reduction, but does not impact the error at final convergence [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time per iteration is reduced by 25-30% compared to the baseline [S1, unofficial]. This efficiency is primarily due to the elimination of partial derivatives for the RHS, which obviates the need to store intermediate activations in memory [S1, unofficial].
*   **Memory Complexity:** Lower memory usage than reverse-mode AD because intermediate activations for the RHS are not stored [S1, unofficial].
*   **Performance:** On average, the semi-gradient method decreases the error to the reference by 8.8 times across various scenes compared to the baseline [S1, unofficial]. In free-viewpoint video rendering, MAPE errors decrease by a factor of 2 to 35 times [S1, unofficial]. On the arXiv cover figure, the Baseline MAPE is 0.318 (1.0x) and the Semi-Gradient MAPE is 0.069 (4.6x) [S1, unofficial].
*   **Strengths:** Removes gradient bias and high variance associated with the RHS derivative. Faster convergence and lower final error.
*   **Limitations:** Convergence is not guaranteed for all scenes (only those satisfying energy absorption condition). Non-L2 losses (RMSE, Huber, MAE) still benefit (2-15 times lower error) but the theoretical justification is less robust than for L2 [S1, unofficial].
*   **When to use:** When training neural radiance caches for global illumination rendering and fast convergence is desired.
*   **When not to use:** When strict theoretical convergence guarantees are required for all scene types, or when the scene is simple enough that stochasticity is low (where the baseline may perform comparably) [S1, unofficial].

## Background
The Semi-Gradient Method was proposed by In-Young Cho and JaeWoong Cho from Krafton, Republic of Korea, in the paper "Fast and Accurate Neural Rendering Using Semi-Gradients" (arXiv:2410.10149, October 14, 2024) [S1, unofficial]. It addresses issues in neural radiosity methods, specifically the baseline method by Hadadan et al. [2021], which minimizes the residual between the LHS and RHS of the rendering equation [S1, unofficial]. The method builds on the observation that the stochasticity of Monte Carlo integration causes gradient bias and high variance in the RHS derivative [S1, unofficial]. It is implemented using PyTorch, Mitsuba 3, and the tiny-cuda-nn library [S1, unofficial]. Training is performed on a single NVIDIA A100 GPU, using 7GB or less of VRAM [S1, unofficial].

## Sources
[S1] Fast and Accurate Neural Rendering Using Semi-Gradients, https://doi.org/10.48550/arXiv.2410.10149, fetched 2026-10-29 (unofficial)
