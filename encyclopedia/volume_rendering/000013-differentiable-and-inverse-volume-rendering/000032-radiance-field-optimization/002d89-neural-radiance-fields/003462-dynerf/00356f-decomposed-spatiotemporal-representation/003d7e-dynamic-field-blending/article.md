# Dynamic Field Blending


## Overview
Dynamic Field Blending is a technique for modeling dynamic scenes by blending static and dynamic neural network predictions for opacity and radiance, evaluated via quadrature to compute the final RGB radiance [S1, unofficial]. It addresses the problem of rendering time-varying scenes by decomposing the scene representation into static and dynamic components, allowing the renderer to adaptively switch between or mix these components based on a learned blending weight [S1, unofficial]. It sets itself apart from sibling methods in the [Decomposed Spatiotemporal Representation](7525-decomposed-spatiotemporal-representation.md) family by explicitly linearly interpolating the opacity and premultiplied radiance terms rather than using separate volume rendering integrals or hard segmentation masks [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera position $\mathbf{x}$, view direction $\boldsymbol{\omega}_0$, time $\tau$, number of samples $N$, sample positions $\mathbf{x}_k$, step size $\Delta \mathbf{x}_k$, static network outputs ($\sigma^{\text{stat}}, L_s^{\text{stat}}$), dynamic network outputs ($\sigma^{\text{dyn}}, L_s^{\text{dyn}}$), and blending weights $b(\mathbf{x}_k, \tau)$.
*   **Output:** Final RGB radiance $I_{\text{RGB}}(\mathbf{x}, \boldsymbol{\omega}_0, \tau)$.

**Data Structures:**
The method relies on the outputs of two neural networks (static and dynamic) and a blending weight field.
*   Static Network Output: Volume density $\sigma^{\text{stat}}(\mathbf{x})$ and scattered radiance $L_s^{\text{stat}}(\mathbf{x}, \boldsymbol{\omega}_0)$.
*   Dynamic Network Output: Volume density $\sigma^{\text{dyn}}(\mathbf{x}, \tau)$ and scattered radiance $L_s^{\text{dyn}}(\mathbf{x}, \boldsymbol{\omega}_0, \tau)$.
*   Blending Weight Field: $b(\mathbf{x}, \tau)$, a scalar value in $[0, 1]$ indicating the contribution of the dynamic component.

**Numbered Steps:**
1.  For each sample point $\mathbf{x}_k$ along the ray (where $\mathbf{x}_k = \mathbf{x} - \boldsymbol{\omega}_0 k$ for $k=0 \dots N$):
    a. Query the static network for $\sigma^{\text{stat}}(\mathbf{x}_k)$ and $L_s^{\text{stat}}(\mathbf{x}_k, \boldsymbol{\omega}_0)$.
    b. Query the dynamic network for $\sigma^{\text{dyn}}(\mathbf{x}_k, \tau)$ and $L_s^{\text{dyn}}(\mathbf{x}_k, \boldsymbol{\omega}_0, \tau)$.
    c. Query the blending weight field for $b(\mathbf{x}_k, \tau)$.
    d. Compute static opacity $\alpha^{\text{stat}}(\mathbf{x}_k)$ using Equation (13).
    e. Compute dynamic opacity $\alpha^{\text{dyn}}(\mathbf{x}_k, \tau)$ using Equation (14).
    f. Compute blended opacity $\alpha^{\text{blend}}(\mathbf{x}_k, \tau)$ using Equation (15).
    g. Compute blended premultiplied radiance $\alpha^{\text{blend}}(\mathbf{x}_k, \tau) L_s^{\text{blend}}(\mathbf{x}_k, \boldsymbol{\omega}_0, \tau)$ using Equations (16) and (17).
2.  Compute blended transmittance $\hat{T}^{\text{blend}}(\mathbf{x}, \mathbf{x}_k, \tau)$ for each sample using Equation (12).
3.  Sum the contributions of all samples to compute the final radiance using Equation (11).

**Formulas:**
*   **Blended Transmittance:**
    $$ \hat{T}^{\text{blend}}(\mathbf{x}, \mathbf{x}_k, \tau) = \prod_{j=0}^{k-1} \left(1 - \alpha^{\text{blend}}(\mathbf{x}_j, \tau)\right) \quad (12) $$
    where $\hat{T}^{\text{blend}}(\mathbf{x}, \mathbf{x}_0, \tau) = 1$.

*   **Static Opacity:**
    $$ \alpha^{\text{stat}}(\mathbf{x}_k) = 1 - \exp(-\sigma^{\text{stat}}(\mathbf{x}_k) \Delta \mathbf{x}_k) \quad (13) $$

*   **Dynamic Opacity:**
    $$ \alpha^{\text{dyn}}(\mathbf{x}_k, \tau) = 1 - \exp(-\sigma^{\text{dyn}}(\mathbf{x}_k, \tau) \Delta \mathbf{x}_k) \quad (14) $$

*   **Blended Opacity:**
    $$ \alpha^{\text{blend}}(\mathbf{x}_k, \tau) = (1 - b(\mathbf{x}_k, \tau)) \cdot \alpha^{\text{stat}}(\mathbf{x}_k) + b(\mathbf{x}_k, \tau) \cdot \alpha^{\text{dyn}}(\mathbf{x}_k, \tau) \quad (15) $$

*   **Blended Premultiplied Radiance:**
    $$ \alpha^{\text{blend}}(\mathbf{x}_k, \tau) L_s^{\text{blend}}(\mathbf{x}_k, \boldsymbol{\omega}_0, \tau) = (1 - b(\mathbf{x}_k, \tau)) \cdot \alpha^{\text{stat}}(\mathbf{x}_k, \tau) L_s^{\text{stat}}(\mathbf{x}_k, \boldsymbol{\omega}_0) + b(\mathbf{x}_k, \tau) \cdot \alpha^{\text{dyn}}(\mathbf{x}_k, \tau) L_s^{\text{dyn}}(\mathbf{x}_k, \boldsymbol{\omega}_0, \tau) \quad (16, 17) $$
    Note: The source text lists Equation (16) and (17) as a single combined equation for the premultiplied term. The term $\alpha^{\text{stat}}(\mathbf{x}_k, \tau)$ in the source text for the static radiance part appears to be a typo or specific notation, as Equation (13) defines $\alpha^{\text{stat}}$ without time dependence. However, the source explicitly writes $\alpha^{\text{stat}}(\mathbf{x}_k, \tau)$ in Eq (16). We retain the source's notation.

*   **Final Radiance:**
    $$ I_{\text{RGB}}(\mathbf{x}, \boldsymbol{\omega}_0, \tau) = \sum_{k=0}^N \hat{T}^{\text{blend}}(\mathbf{x}, \mathbf{x}_k, \tau) \alpha^{\text{blend}}(\mathbf{x}_k, \tau) L_s^{\text{blend}}(\mathbf{x}_k, \boldsymbol{\omega}_0, \tau) \quad (11) $$

**Parameters:**
*   $b(\mathbf{x}_k, \tau)$: The blending weight at position $\mathbf{x}_k$ and time $\tau$. It is a learned parameter or output of a network, typically constrained to $[0, 1]$. It linearly interpolates between static and dynamic opacity and radiance terms [S1, unofficial].
*   $N$: Number of samples along the ray. Not given in the sources.
*   $\Delta \mathbf{x}_k$: Step size between samples. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The product in the transmittance calculation (Eq 12) can underflow to zero if the scene is dense, leading to numerical instability. Standard volume rendering techniques (e.g., using log-space accumulation) should be applied [G].
*   **Blending Weight Boundaries:** If $b(\mathbf{x}_k, \tau)$ is not constrained to $[0, 1]$, the blending may produce invalid opacity values (e.g., $>1$ or $<0$). The blending weight network should use a sigmoid activation or similar constraint [G].
*   **Temporal Consistency:** The blending weight $b(\mathbf{x}_k, \tau)$ must vary smoothly with time to avoid flickering artifacts. This is typically enforced via temporal regularization during training [G].
*   **Static Radiance Time Dependence:** Equation (16) includes $\alpha^{\text{stat}}(\mathbf{x}_k, \tau)$, implying the static opacity might depend on time, which contradicts the "static" label. This may be a notation artifact in the source [S1, unofficial]. Implementers should verify if the static network is truly time-independent.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_field_blending(ray_origin, ray_direction, time, num_samples, step_size, 
                           static_net, dynamic_net, blending_net):
    """
    Renders a single ray using Dynamic Field Blending.
    
    Args:
        ray_origin: 3D vector, camera position x.
        ray_direction: 3D vector, view direction omega_0.
        time: float, time tau.
        num_samples: int, N.
        step_size: float, Delta x_k.
        static_net: Function returning (sigma_stat, L_stat) for a given position.
        dynamic_net: Function returning (sigma_dyn, L_dyn) for a given position and time.
        blending_net: Function returning blending weight b for a given position and time.
        
    Returns:
        RGB radiance value.
    """
    total_radiance = 0.0
    transmittance = 1.0
    
    for k in range(num_samples):
        # Sample position x_k = x - omega_0 * k (assuming uniform sampling for simplicity)
        # Note: The source uses x_k = x - omega_0 * k, which implies k is a scalar multiplier.
        # In practice, k might be an index and step_size is used.
        # We assume x_k = ray_origin + ray_direction * (k * step_size) for standard volume rendering.
        # However, the source explicitly states x_k = x - omega_0 * k. 
        # This suggests k might be the distance, not the index. 
        # Let's assume k is the index and the formula in the source is a simplification.
        # We will use standard ray marching: x_k = ray_origin + ray_direction * (k * step_size)
        x_k = ray_origin + ray_direction * (k * step_size)
        
        # Query networks
        sigma_stat, L_stat = static_net(x_k)
        sigma_dyn, L_dyn = dynamic_net(x_k, time)
        b = blending_net(x_k, time)
        
        # Compute opacities
        alpha_stat = 1.0 - math.exp(-sigma_stat * step_size)
        alpha_dyn = 1.0 - math.exp(-sigma_dyn * step_size)
        
        # Compute blended opacity
        alpha_blend = (1.0 - b) * alpha_stat + b * alpha_dyn
        
        # Compute blended premultiplied radiance
        # Note: Source uses alpha_stat(x_k, tau) in Eq 16. We use alpha_stat(x_k) as per Eq 13.
        # If the source implies time-dependent static opacity, adjust accordingly.
        # We follow the source's explicit formula structure.
        premultiplied_radiance = ((1.0 - b) * alpha_stat * L_stat + 
                                  b * alpha_dyn * L_dyn)
        
        # Accumulate radiance
        total_radiance += transmittance * premultiplied_radiance
        
        # Update transmittance
        transmittance *= (1.0 - alpha_blend)
        
        # Early termination if transmittance is very small
        if transmittance < 1e-5:
            break
            
    return total_radiance
```

## Subtopics

### Static-Dynamic Decomposition
Separates the scene into static background and dynamic foreground objects. It is chosen when the scene has significant static structure and limited dynamic elements. [Decomposed Spatiotemporal Representation](7525-decomposed-spatiotemporal-representation.md)

### Blending Weight Learning
The mechanism for determining the contribution of static vs. dynamic components. It is chosen when smooth transitions between static and dynamic regions are required. [Decomposed Spatiotemporal Representation](7525-decomposed-spatiotemporal-representation.md)

## Operating characteristics
*   **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples. Each sample requires queries to three networks (static, dynamic, blending).
*   **Memory Complexity:** Depends on the size of the static, dynamic, and blending networks. Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with the number of samples and the complexity of the neural networks.
*   **Strengths:** Allows for smooth blending between static and dynamic components, avoiding hard edges. Efficiently handles scenes with both static and dynamic elements.
*   **Limitations:** Requires training three networks (static, dynamic, blending). The blending weight must be learned, which can be challenging.
*   **When to Use:** When rendering dynamic scenes with significant static background.
*   **When Not to Use:** When the scene is entirely static or entirely dynamic, as the overhead of blending may not be justified.

## Background
Dynamic Field Blending was introduced in the TöRF (Time-of-Flight Radiance Fields) paper by [S1, unofficial]. It is part of the DyNeRF family of methods for dynamic scene view synthesis. The method builds on the idea of decomposing the scene into static and dynamic components and blending their contributions.

## Sources
[S1] TöRF: Time-of-Flight Radiance Fields for Dynamic Scene View Synthesis, https://doi.org/10.48550/arxiv.2109.15271, fetched 2026-10-29 (unofficial)
