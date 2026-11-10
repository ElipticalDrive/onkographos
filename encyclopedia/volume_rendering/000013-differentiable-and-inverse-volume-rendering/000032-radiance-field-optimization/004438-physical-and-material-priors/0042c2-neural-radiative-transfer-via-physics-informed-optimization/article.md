# Neural Radiative Transfer via Physics-Informed Optimization

Also called: Neural Inverse Rendering for Participating Media, Volumetric Inverse Rendering

## Overview
This method reconstructs spatially varying optical properties (absorption, scattering, phase function) and the light field of participating media from posed RGB images by enforcing the Radiative Transfer Equation (RTE) as a physics-informed constraint [S1]. Unlike surface-based inverse rendering which relies on stochastic ray sampling for nonlocal transport, this approach enforces light transport locally throughout the volume by imposing the RTE as a pointwise constraint on neural field representations [S1]. It is chosen when physically complete volumetric light transport is required without restrictive assumptions such as homogeneous media [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of posed RGB images $\{I_k\}$ providing multi-view observations. The environment illumination is assumed known and fixed [S1].
*   **Output:** A neural light field representation $L_\theta$ encoding the spatio-angular light field, and a neural medium field representation $M_\phi$ encoding absorption $\sigma_a$, scattering $\sigma_s$, and phase function parameters [S1].

**Data Structures:**
1.  **Neural Light Field Representation:** $L_\theta : (\mathbf{x}, \boldsymbol{\omega}) \rightarrow \mathbb{R}^3$.
    *   Inputs: Spatial position $\mathbf{x} \in \mathbb{R}^3$ and direction $\boldsymbol{\omega} \in S^2$.
    *   Output: Radiance $\mathbb{R}^3$.
    *   Parameters: $\theta$ [S1].
2.  **Neural Medium Field Representation:** $M_\phi : \mathbf{x} \rightarrow (\sigma_a, \sigma_s, g)$.
    *   Input: Spatial position $\mathbf{x} \in \mathbb{R}^3$.
    *   Outputs:
        *   Absorption coefficient $\sigma_a(\mathbf{x}) \in \mathbb{R}^3$.
        *   Scattering coefficient $\sigma_s(\mathbf{x}) \in \mathbb{R}^3$.
        *   Scalar anisotropy parameter $g$ defining the Henyey-Greenstein phase function [S1].
    *   Parameters: $\phi$ [S1].

**Formulas:**
*   **Extinction Coefficient:**
    $$ \sigma_t(\mathbf{x}) = \sigma_a(\mathbf{x}) + \sigma_s(\mathbf{x}) $$
    [S1]

*   **RTE Residual Loss ($L_{RTE}$):**
    Enforces the differential RTE at collocation points $(\mathbf{x}, \boldsymbol{\omega})$. The emission term is omitted as only environment illumination is considered [S1].
    $$ L_{RTE}(\theta, \phi) = \mathbb{E}_{\mathbf{x}, \boldsymbol{\omega}} \left[ \left( \boldsymbol{\omega} \cdot \nabla \right) L_\theta(\mathbf{x}, \boldsymbol{\omega}) + \sigma_t(\mathbf{x}) L_\theta(\mathbf{x}, \boldsymbol{\omega}) - \sigma_s(\mathbf{x}) \int_{S^2} f_p(\mathbf{x}, \boldsymbol{\omega}, \boldsymbol{\omega}') L_\theta(\mathbf{x}, \boldsymbol{\omega}') d\boldsymbol{\omega}' \right]^2 $$
    [S1]

*   **Inflow Boundary Condition Domain ($\Gamma$):**
    $$ \Gamma = \{ (\mathbf{x}, \boldsymbol{\omega}) \mid \mathbf{x} \in \partial B, \boldsymbol{\omega} \in S^2, \boldsymbol{\omega} \cdot \mathbf{n}_B(\mathbf{x}) < 0 \} $$
    where $\partial B$ is the scene boundary and $\mathbf{n}_B$ is the outward unit normal [S1].

*   **Boundary Condition Loss ($L_{BC}$):**
    $$ L_{BC}(\theta) = \mathbb{E}_{(\mathbf{x}, \boldsymbol{\omega}) \in \Gamma} \left[ L_{in}(\mathbf{x}, \boldsymbol{\omega}) - L_\theta(\mathbf{x}, \boldsymbol{\omega}) \right]^2 $$
    where $L_{in}$ is the incoming radiance from environment illumination [S1].

*   **Volumetric Rendering Equation (VRE) Loss ($L_{VRE}$):**
    Introduced to counteract oversmoothing by adding nonlocal constraints along viewing rays [S1].
    $$ L_{VRE}(\theta) = \mathbb{E}_{\mathbf{r}} \left[ \left\| \hat{C}(\mathbf{r}) - C(\mathbf{r}) \right\|^2 \right] $$
    where $\hat{C}(\mathbf{r})$ is the rendered color from $L_\theta$ and $C(\mathbf{r})$ is the observed color [S1].

**Steps:**
1.  **Sample Collocation Points:** Sample spatial positions $\mathbf{x}$ and directions $\boldsymbol{\omega}$ from the spatio-angular domain [S1].
2.  **Evaluate Directional Derivative:** Compute the term $(\boldsymbol{\omega} \cdot \nabla) L_\theta(\mathbf{x}, \boldsymbol{\omega})$ using automatic differentiation of the neural network $L_\theta$ with respect to $\mathbf{x}$ [S1].
3.  **Approximate In-Scattering Integral:** Approximate $\int_{S^2} f_p(\mathbf{x}, \boldsymbol{\omega}, \boldsymbol{\omega}') L_\theta(\mathbf{x}, \boldsymbol{\omega}') d\boldsymbol{\omega}'$ using Monte Carlo sampling over the angular domain $S^2$ at the spatial location $\mathbf{x}$ [S1]. No ray tracing is performed [S1].
4.  **Compute RTE Residual:** Calculate the residual of the RTE using the terms from steps 2 and 3, along with $\sigma_t$ and $\sigma_s$ from $M_\phi$ [S1].
5.  **Enforce Boundary Conditions:** Sample points on the boundary $\partial B$ with incoming directions $\boldsymbol{\omega}$ (where $\boldsymbol{\omega} \cdot \mathbf{n}_B < 0$) and compute $L_{BC}$ [S1].
6.  **Render and Compare:** For each input image $I_k$, render the color $\hat{C}(\mathbf{r})$ by integrating $L_\theta$ along the viewing rays $\mathbf{r}$ corresponding to the pixels. Compute $L_{VRE}$ against the observed image pixels [S1].
7.  **Optimize:** Minimize the total loss $L_{total} = L_{RTE} + L_{BC} + L_{VRE}$ with respect to parameters $\theta$ and $\phi$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_neural_radiative_transfer(images, poses, environment_radiance, num_iterations):
    # Initialize neural networks
    L_theta = NeuralNetwork() # L_theta: (x, w) -> R3
    M_phi = NeuralNetwork()   # M_phi: x -> (sigma_a, sigma_s, g)
    
    optimizer = Optimizer([L_theta.parameters(), M_phi.parameters()])
    
    for iteration in range(num_iterations):
        optimizer.zero_grad()
        
        # 1. Sample collocation points for RTE
        x_colloc = sample_spatial_points()
        w_colloc = sample_directions()
        
        # 2. Evaluate directional derivative (omega . grad) L_theta
        # Requires automatic differentiation of L_theta w.r.t x
        L_val = L_theta(x_colloc, w_colloc)
        grad_L = gradient(L_val, x_colloc)
        directional_derivative = dot(w_colloc, grad_L)
        
        # 3. Get medium properties
        sigma_a, sigma_s, g = M_phi(x_colloc)
        sigma_t = sigma_a + sigma_s
        phase_func = HenyeyGreenstein(g)
        
        # 4. Approximate in-scattering integral via Monte Carlo
        # Sample directions w_prime for each x_colloc
        w_prime = sample_directions()
        L_prime = L_theta(x_colloc, w_prime)
        phase_values = phase_func(w_colloc, w_prime)
        
        # Monte Carlo estimate: (1/N) * sum(phase * L_prime) * 4*pi
        in_scattering = mean(phase_values * L_prime) * 4 * pi
        
        # 5. Compute RTE Residual
        rte_residual = directional_derivative + sigma_t * L_val - sigma_s * in_scattering
        loss_rte = mean(rte_residual ** 2)
        
        # 6. Enforce Boundary Conditions
        x_boundary, w_boundary = sample_boundary_points()
        L_boundary = L_theta(x_boundary, w_boundary)
        L_in = environment_radiance(x_boundary, w_boundary)
        loss_bc = mean((L_in - L_boundary) ** 2)
        
        # 7. Render and Compute VRE Loss
        loss_vre = 0
        for image, pose in zip(images, poses):
            rays = generate_rays(pose, image.shape)
            # Integrate L_theta along rays to get rendered color C_hat
            C_hat = render_volumetric(L_theta, rays)
            C_obs = image
            loss_vre += mean((C_hat - C_obs) ** 2)
            
        # 8. Total Loss and Backprop
        total_loss = loss_rte + loss_bc + loss_vre
        total_loss.backward()
        optimizer.step()
```

**Parameters:**
*   **Neural Network Architecture:** Not given in the sources.
*   **Monte Carlo Samples:** Not given in the sources.
*   **Collocation Points:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Oversmoothing:** Solutions may be overly smooth and lack high-frequency detail due to the low-frequency bias of neural networks [S1]. This is mitigated by adding the VRE loss [S1].
*   **Optimization Pathologies:** Related optimization pathologies have been reported for physics-informed formulations relying on differential constraints [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Naturally supports the optimization of soft geometry (e.g., continuous volumetric media such as clouds or fog) [S1]. Does not rely on restrictive assumptions such as homogeneous media [S1]. Enforces light transport locally throughout the volume [S1].
*   **Limitations:** Solutions may be overly smooth [S1]. Hard geometry optimization remains challenging for PINN-based inverse rendering in general [S1].
*   **When to use:** When physically complete volumetric light transport is required for participating media with known environment illumination [S1].
*   **When not to use:** When hard geometry with sharp discontinuities is the primary target [S1].

## Background
The method is based on Physics-Informed Neural Networks (PINNs) which integrate physical laws into neural network training by penalizing violations of equations in the loss function [S1]. While PINNs have been explored for light transport in surface-based scenes, this method extends the concept to volumetric scenes by enforcing the RTE as a pointwise constraint [S1]. The approach was introduced in "Volumetric Inverse Rendering via Neural Radiative Transfer" [S1].

## Sources
[S1] Volumetric Inverse Rendering via Neural Radiative Transfer, https://arxiv.org/abs/2607.13695, fetched 2026-10-29
