# Surrogate Gradient for Discrete Material Assignment

Also called: Beer-Lambert Discrete CT, Discrete Computed Tomography (Absorptive), Inverse Rendering for Discrete X-Ray CT

## Overview
This method reconstructs discrete material assignments $X_i \in \{1, \dots, m\}$ for voxels in a volume using differentiable volume rendering based on the Beer-Lambert law for absorptive X-rays. It solves the inverse problem of Computed Tomography (CT) by directly minimizing the Reconstruction Residual Error (RRE) during optimization, rather than using it only for post-hoc evaluation. It distinguishes itself from continuous density reconstruction by enforcing a categorical distribution over a finite set of materials per voxel, using a two-pass DDA traversal to efficiently compute surrogate gradients without retracing rays.

## How it works
**Inputs and Outputs:**
*   **Input:** Measured projection intensities $\{I_{\text{ref}}(\mathbf{r})\}$ for a set of rays $\mathbf{r}$; source intensity $I_0$ (cancels out in log-space); material absorption coefficients $\mu^{(k)}$ for $k \in \{1, \dots, m\}$.
*   **Optimization Variables:** Categorical distributions $\theta_i \in \Delta_{m-1}$ for each voxel $i$, where $\theta_{i,k}$ is the probability of voxel $i$ being material $k$.
*   **Output:** Optimized $\theta_i$ (from which discrete assignments $X_i$ are derived, e.g., via argmax).

**Data Structures:**
*   **Voxel Grid:** Regular 3D grid. Each voxel $i$ stores a vector $\theta_i$ of size $m$.
*   **Ray Data:** For each ray $\mathbf{r}$, a list of intersected voxels $i \in \mathbf{r}$ and their corresponding path lengths $\Delta_i$. These are computed analytically via 3D-DDA (Digital Differential Analyzer) assuming homogeneous voxels [S1].
*   **Gradient Buffer:** A 2D array $g$ of shape $(N_{\text{voxels}}, m)$ to store per-voxel, per-material residuals.

**Formulas:**
1.  **Beer-Lambert Law:** The transmitted intensity for a ray $\mathbf{r}(t) = \mathbf{o} + t\boldsymbol{\omega}$ is:
    $$I(\mathbf{X}, \mathbf{r}) = I_0 \exp \left( - \sum_{i \in \mathbf{r}} \mu(X_i) \Delta_i \right)$$
    where $\Delta_i$ is the ray-voxel intersection length and $\mu(k)$ is the absorption coefficient of material $k$ [S1].

2.  **Optical Depth:** The logarithmic transform linearizes the law into additive optical depth:
    $$\tau = \sum_{i \in \mathbf{r}} \mu(X_i) \Delta_i$$
    The loss operates directly on $\tau$ because $I_0$ cancels out when taking $\log(I/I_0)$ [S1].

3.  **Reconstruction Loss:** The total loss is the sum of squared log-differences over all rays:
    $$\mathcal{L}(\mathbf{X}) = \sum_{\mathbf{r}} (\log I(\mathbf{X}, \mathbf{r}) - \log I_{\text{ref}}(\mathbf{r}))^2 = \sum_{\mathbf{r}} (\tau(\mathbf{X}, \mathbf{r}) - \tau_{\text{ref}}(\mathbf{r}))^2$$
    where $\tau_{\text{ref}}(\mathbf{r}) = -\log(I_{\text{ref}}(\mathbf{r})/I_0)$ [S1].

4.  **Surrogate Gradient:** To differentiate through the discrete assignment, a categorical distribution $\theta_i$ is used. The per-voxel expected loss is:
    $$\mathcal{L}_i(\theta, i) = \sum_{k=1}^m \theta_{i,k} \tilde{\mathcal{L}}_i(k)$$
    where $\tilde{\mathcal{L}}_i(k)$ is the loss if voxel $i$ is clamped to material $k$ [S1].

5.  **Approximated Optical Depth:** For a chosen surrogate strategy (e.g., expectation), the optical depth along a ray is:
    $$\tau(\theta, \mathbf{r}) = \sum_{i \in \mathbf{r}} \hat{\mu}_i \Delta_i$$
    where $\hat{\mu}_i$ is the sampled value $\mu^{(X_i)}$, the mode $\mu^{(\arg \max_k \theta_{i,k})}$, or the expectation $\bar{\mu}_i = \sum_k \theta_{i,k} \mu^{(k)}$ [S1].

6.  **Clamped Optical Depth:** To compute the gradient for material $k$ at voxel $i$, the optical depth is locally updated without retracing:
    $$\tau_i(\theta, \mathbf{r}; k) = \tau(\theta, \mathbf{r}) - \hat{\mu}_i \Delta_i + \mu^{(k)} \Delta_i$$
    The residual for this candidate material is $(\tau_i(\theta, \mathbf{r}; k) - \tau_{\text{ref}}(\mathbf{r}))^2$ [S1].

**Algorithm Steps:**
1.  **Preprocessing:** Compute ray-voxel intersections and path lengths $\Delta_i$ for all rays using 3D-DDA. Compute reference optical depths $\tau_{\text{ref}}(\mathbf{r})$ from $I_{\text{ref}}$.
2.  **Forward Pass (Pass 1):** For each ray $\mathbf{r}$, traverse intersected voxels $i \in \mathbf{r}$. Accumulate the expected optical depth $\tau(\theta, \mathbf{r}) = \sum_{i \in \mathbf{r}} \hat{\mu}_i \Delta_i$ and store the base residual $r_{\text{base}} = \tau(\theta, \mathbf{r}) - \tau_{\text{ref}}(\mathbf{r})$.
3.  **Gradient Pass (Pass 2):** For each ray $\mathbf{r}$, traverse intersected voxels $i \in \mathbf{r}$ again. For each material $k \in \{1, \dots, m\}$:
    *   Compute clamped optical depth: $\tau_i(\theta, \mathbf{r}; k) = \tau(\theta, \mathbf{r}) - \hat{\mu}_i \Delta_i + \mu^{(k)} \Delta_i$.
    *   Compute residual: $residual = \tau_i(\theta, \mathbf{r}; k) - \tau_{\text{ref}}(\mathbf{r})$.
    *   Store squared residual in gradient buffer: $g[i, k] = residual^2$.
4.  **Backpropagation:** Compute gradients of $\mathcal{L}$ with respect to $\theta_{i,k}$ using the stored residuals $g[i, k]$. For expectation surrogate, $\frac{\partial \mathcal{L}}{\partial \theta_{i,k}} = \tilde{\mathcal{L}}_i(k) - \sum_j \theta_{i,j} \tilde{\mathcal{L}}_i(j)$ (standard categorical gradient) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_pass_surrogate_gradient(rays, voxels, theta, mu, delta):
    """
    rays: list of rays, each ray is a list of (voxel_index, path_length)
    voxels: number of voxels
    theta: (num_voxels, m) array of categorical distributions
    mu: (m,) array of material absorption coefficients
    delta: not needed as input, path lengths are in rays
    
    Returns:
    gradients: (num_voxels, m) array of per-voxel per-material losses
    """
    m = mu.shape[0]
    gradients = np.zeros((len(voxels), m)) # Assuming voxel indices are 0..N-1
    
    for ray in rays:
        # Pass 1: Compute expected optical depth tau
        tau = 0.0
        # Assume expectation surrogate: mu_hat_i = sum(theta[i,k] * mu[k])
        # Precompute expected mu for all voxels if not done
        mu_hat = np.dot(theta, mu) 
        
        for voxel_idx, d_i in ray:
            tau += mu_hat[voxel_idx] * d_i
            
        # Reference optical depth for this ray (precomputed)
        tau_ref = ray.reference_tau 
        
        # Base residual for the current configuration
        # Note: The loss is (tau - tau_ref)^2. 
        # We need to compute the loss for each material k at each voxel i.
        
        # Pass 2: Compute per-voxel per-material residuals
        for voxel_idx, d_i in ray:
            # Current contribution of this voxel to tau
            current_contrib = mu_hat[voxel_idx] * d_i
            
            # Base tau without this voxel's current expected contribution
            tau_without_i = tau - current_contrib
            
            for k in range(m):
                # Clamped optical depth if voxel_idx is material k
                tau_clamped = tau_without_i + mu[k] * d_i
                
                # Residual for this specific clamping
                residual = tau_clamped - tau_ref
                
                # Store squared residual (proxy for loss component)
                gradients[voxel_idx, k] = residual ** 2
                
    return gradients
```
*Note: The pseudocode above computes the loss components $\tilde{\mathcal{L}}_i(k)$. The actual gradient w.r.t $\theta$ requires backpropagating through the categorical distribution. The sources describe the evaluation of the surrogate loss components efficiently [S1].*

**Parameters:**
*   $m$: Number of discrete materials. Typical values depend on the sample (e.g., 2-10).
*   Surrogate type: Expectation, Mode, or Sampling. Expectation is smooth and commonly used [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Logarithmic transform requires $I_{\text{ref}} > 0$. Clamp intensities to a small epsilon if necessary [G].
*   **Boundaries:** DDA must correctly handle ray entry/exit and voxel boundaries. Regular tracking assumes homogeneous voxels [S1].
*   **Degenerate Input:** If a ray does not intersect any voxels, $\tau=0$. If $I_{\text{ref}}$ is zero, log is undefined.

## Subtopics

### Discrete CT with Scattering
Extends the absorptive model to include scattered photons, which are typically treated as artifacts. Uses anti-scatter grids or modeling to mitigate effects. [Link](21619-discrete-ct-scattering.md)

### Comparison
Unlike standard CT which minimizes continuous density errors, this method minimizes the discrete assignment error directly. It ensures local optimality with respect to individual variable changes [S1].

## Operating characteristics
*   **Time Complexity:** $O(N_{\text{rays}} \cdot N_{\text{voxels\_per\_ray}} \cdot m)$. The two-pass DDA traversal is efficient and can be fused into a single GPU kernel.
*   **Memory Complexity:** $O(N_{\text{voxels}} \cdot m)$ to store $\theta$ and gradients.
*   **Strengths:** Directly minimizes RRE; efficient two-pass evaluation avoids ray retracing; handles discrete materials naturally.
*   **Limitations:** NP-hard problem in general; relies on local optimality; assumes absorptive X-rays without scattering in the base formulation.
*   **When to Use:** When the material composition is known to be discrete (e.g., biological tissues, industrial parts) and continuous reconstruction leads to ambiguity.
*   **When Not to Use:** When scattering is significant and not modeled; when materials are continuous or unknown.

## Background
Originated in the context of inverse rendering for discrete X-Ray Computed Tomography. The method addresses the NP-hard nature of discrete reconstruction by using differentiable optimization with surrogate gradients [S1]. It builds on the Beer-Lambert law and standard CT reconstruction techniques, adapting them for discrete variable optimization.

## Sources
[S1] Inverse Rendering for Discrete X-Ray Computed Tomography, https://doi.org/10.1145/3811391, fetched 2026-11-03
