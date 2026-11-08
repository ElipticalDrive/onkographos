# Differentiable Neural Radiosity

Also called: DNR

## Overview
Differentiable Neural Radiosity (DNR) is a method for solving the differential rendering equation using a neural network to represent the gradient of the radiance field with respect to arbitrary scene parameters [S1, unofficial]. It differs from standard Neural Radiosity by optimizing a network for the *derivative* of radiance rather than radiance itself, enabling gradient-based inverse rendering without backpropagating through the Monte Carlo rendering graph [S1, unofficial]. Unlike Radiative Backpropagation (RB) [link], Path Replay Backpropagation (PRB) [link], and Automatic Differentiation (AD) [link], DNR provides view-independent solutions with constant time and memory complexity relative to path length, avoiding the quadratic or linear scaling of path-building methods [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene geometry, material properties (BRDFs), illumination, and camera parameters. A set of scene parameters $p$ with respect to which derivatives are desired.
*   **Outputs:** A neural network $\partial_p L_\theta(x, \omega_o)$ representing the differential radiance field, and a pre-trained primal radiance network $L_\phi(x, \omega_o)$.

**Data Structures:**
1.  **Primal Radiance Network ($L_\phi$):** A neural network with fixed parameters $\phi$. It takes position $x$ and outgoing direction $\omega_o$ as input and outputs radiance $L_\phi(x, \omega_o)$. Parameters $\phi$ are constant during the differential training phase.
2.  **Differential Radiance Network ($\partial_p L_\theta$):** A neural network with learnable parameters $\theta$. It takes position $x$, outgoing direction $\omega_o$, and potentially other scene descriptors as input, outputting the differential radiance $\partial_p L_\theta(x, \omega_o)$.

**Formulas:**
The differential rendering equation for a parameter $p$ is derived from the standard rendering equation [S1, unofficial]:
$$
\partial_p L(x, \omega_o) = \partial_p E(x, \omega_o) + \int_{H^2} f(x, \omega_i, \omega_o) \partial_p L(x'(x, \omega_i), -\omega_i) d\omega_i^\perp + \int_{H^2} \partial_p f(x, \omega_i, \omega_o) L(x'(x, \omega_i), -\omega_i) d\omega_i^\perp
$$
where:
*   $L(x, \omega_o)$ is the outgoing radiance.
*   $E(x, \omega_o)$ is the emitted radiance.
*   $f(x, \omega_i, \omega_o)$ is the BRDF.
*   $x'(x, \omega_i)$ is the next intersection point along direction $\omega_i$ from $x$.
*   $H^2$ is the hemisphere of incoming directions.
*   $\partial_p$ denotes the derivative with respect to scene parameter $p$.

The residual $r_\theta$ for the differential radiance network is defined as the difference between the left-hand side (LHS) and right-hand side (RHS) of the differential equation [S1, unofficial]:
$$
r_\theta(x, \omega_o) = \partial_p L_\theta(x, \omega_o) - \partial_p E(x, \omega_o) - \int_{H^2} f(x, \omega_i, \omega_o) \partial_p L_\theta(x'(x, \omega_i), -\omega_i) d\omega_i^\perp - \int_{H^2} \partial_p f(x, \omega_i, \omega_o) L_\phi(x'(x, \omega_i), -\omega_i) d\omega_i^\perp
$$
The loss function is the L2 norm of this residual [S1, unofficial]:
$$
\mathcal{L}(\theta) = \| r_\theta \|^2
$$

**Steps:**
1.  **Train Primal Network:** Train $L_\phi$ to approximate the primal radiance field $L(x, \omega_o)$ using standard Neural Radiosity techniques. Fix parameters $\phi$.
2.  **Sample Points:** Sample positions $x$ and directions $\omega_o$ from the scene.
3.  **Evaluate Residual:** For each sample, compute the residual $r_\theta(x, \omega_o)$ using the current differential network $\partial_p L_\theta$, the fixed primal network $L_\phi$, the scene BRDF $f$, and its derivative $\partial_p f$.
    *   The term $\partial_p L_\theta(x'(x, \omega_i), -\omega_i)$ is obtained by querying the differential network at the next intersection.
    *   The term $L_\phi(x'(x, \omega_i), -\omega_i)$ is obtained by querying the primal network at the next intersection.
    *   The integrals are approximated using Monte Carlo integration.
4.  **Optimize:** Update parameters $\theta$ to minimize $\mathcal{L}(\theta)$ using gradient descent.
5.  **Inference:** Use the trained $\partial_p L_\theta$ to compute gradients for inverse rendering tasks.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

class DifferentialNeuralRadiosity:
    def __init__(self, primal_network, differential_network, scene):
        """
        primal_network: nn.Module representing L_phi(x, w_o)
        differential_network: nn.Module representing partial_p L_theta(x, w_o)
        scene: Object providing ray intersection, BRDF evaluation, and derivative of BRDF
        """
        self.primal_network = primal_network
        self.differential_network = differential_network
        self.scene = scene
        # Parameters phi are frozen
        for param in self.primal_network.parameters():
            param.requires_grad = False

    def compute_residual(self, x, w_o, p):
        """
        Computes the residual r_theta for a given position, direction, and parameter p.
        """
        # LHS: Differential radiance from network
        dL_theta = self.differential_network(x, w_o, p)
        
        # RHS Term 1: Differential emission
        # Note: Source assumes static camera and ignores visibility discontinuities [S1, unofficial]
        dE = self.scene.compute_differential_emission(x, w_o, p)
        
        # RHS Term 2 & 3: Integrals over hemisphere
        # Approximated via Monte Carlo sampling
        num_samples = 10 # Example value, not specified in source
        integral_term_2 = 0.0
        integral_term_3 = 0.0
        
        for _ in range(num_samples):
            w_i, pdf = self.scene.sample_hemisphere(x, w_o)
            x_next = self.scene.ray_intersect(x, w_i)
            
            if x_next is None:
                continue
                
            # Term 2: Integral of f * dL_theta
            f = self.scene.brdf(x, w_i, w_o)
            dL_theta_next = self.differential_network(x_next, -w_i, p)
            integral_term_2 += f * dL_theta_next / pdf
            
            # Term 3: Integral of df/dp * L_phi
            df_dp = self.scene.brdf_derivative(x, w_i, w_o, p)
            L_phi_next = self.primal_network(x_next, -w_i)
            integral_term_3 += df_dp * L_phi_next / pdf
            
        # Average over samples
        integral_term_2 /= num_samples
        integral_term_3 /= num_samples
        
        residual = dL_theta - dE - integral_term_2 - integral_term_3
        return residual

    def train(self, num_iterations, optimizer, batch_size=100):
        """
        Trains the differential network by minimizing the L2 norm of the residual.
        """
        for i in range(num_iterations):
            optimizer.zero_grad()
            
            # Sample batch of positions, directions, and parameters
            x_batch, w_o_batch, p_batch = self.scene.sample_batch(batch_size)
            
            residuals = []
            for x, w_o, p in zip(x_batch, w_o_batch, p_batch):
                r = self.compute_residual(x, w_o, p)
                residuals.append(r)
            
            residuals = torch.stack(residuals)
            loss = torch.mean(residuals ** 2)
            
            loss.backward()
            optimizer.step()
            
    def get_gradient(self, x, w_o, p):
        """
        Returns the differential radiance for a given query.
        """
        return self.differential_network(x, w_o, p)
```

**Parameters:**
*   $\theta$: Learnable parameters of the differential radiance network.
*   $\phi$: Fixed parameters of the primal radiance network.
*   $p$: Scene parameters (e.g., BRDF roughness, albedo, geometry).
*   Typical values for learning rates and batch sizes are not specified in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Visibility Discontinuities:** The method ignores visibility-related discontinuities (occlusions) for simplicity, which can lead to incorrect gradients in scenes with sharp shadows or silhouettes [S1, unofficial].
*   **Primal Network Dependency:** The accuracy of the differential network depends on the quality of the pre-trained primal network $L_\phi$.

## Subtopics

### [Inverse Rendering with Neural Radiosity](7428-inverse-rendering-with-neural-radiosity.md)
Optimizes scene parameters by minimizing the distance between rendered and reference images using gradients from a differential radiance network. It differs from standard Neural Radiosity by solving the differential rendering equation to obtain $\partial_p L$, enabling gradient-based optimization of scene properties without backpropagating through the entire Monte Carlo rendering graph.

## Operating characteristics
*   **Time Complexity:** Constant in path length given pre-trained networks, as it queries the network at the first intersection rather than building complete light paths [S1, unofficial]. In contrast, PRB and AD are linear in path length, and RB is quadratic [S1, unofficial].
*   **Memory Complexity:** Constant in path length, similar to RB and PRB, as it does not store per-bounce data in an adjoint transcript [S1, unofficial].
*   **Performance:** Gradient computation time per iteration is 0.002s, compared to 0.009s for RB, 0.017s for PRB, and 0.088s for AD [S1, unofficial].
*   **View-Independence:** Provides view-independent solutions, meaning sensor parameter changes do not require recomputation unless sensor parameters are being optimized [S1, unofficial].
*   **Strengths:** Faster convergence, lower memory usage, and view-independence.
*   **Limitations:** Ignores visibility discontinuities, potentially leading to inaccurate gradients in complex occlusion scenarios [S1, unofficial].

## Background
Differentiable Neural Radiosity was introduced by Saeed Hadadan and Matthias Zwicker from the University of Maryland, College Park, in a paper published on arXiv in January 2022 [S1, unofficial]. It builds on the concept of Neural Radiosity, extending it to solve the differential rendering equation directly using neural networks.

## Sources
[S1] Differentiable Neural Radiosity, https://arxiv.org/abs/2201.13190, fetched 2026-10-29 (unofficial)
