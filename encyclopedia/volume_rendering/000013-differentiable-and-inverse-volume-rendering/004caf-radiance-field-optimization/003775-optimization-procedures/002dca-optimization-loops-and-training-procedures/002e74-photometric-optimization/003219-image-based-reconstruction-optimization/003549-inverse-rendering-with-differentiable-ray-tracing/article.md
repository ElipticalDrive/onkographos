# Inverse Rendering with Differentiable Ray Tracing

Also called: Differentiable Rendering Optimization, Gradient-based Inverse Rendering, Robust Point Light Source Estimation Using Differentiable Rendering

## Overview
Inverse rendering recovers latent scene parameters—such as camera pose, light placement, and surface color—that produce a given target photograph by formulating the problem as an unconstrained optimization task using gradients from the rendering pipeline [S1]. This method specifically estimates the 3D position and intensity of a punctual light source by minimizing the photometric error between a rendered image and an observed RGBD image, using a completely differentiable renderer based on the Blinn-Phong model with cast shadows [S2, unofficial]. Unlike methods that use discrete candidate light sources or explicit shadow detection, this approach performs continuous optimization of light position and renders a shadow map at each iteration [S2, unofficial]. Unlike Neverova et al. [22], this method does not rely on explicit separation between specular and diffuse components, nor does it segment images or extract surface properties [S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   Scene geometry $G = (X, N, C)$, where $X$ is a 3D point cloud, $N$ is the surface normals, and $C$ is the camera pose [S2, unofficial].
    *   Material properties $M = (k_d, k_s, \alpha)$, where $k_d$ is diffuse reflectance, $k_s$ is specular reflectance, and $\alpha$ is shininess [S2, unofficial].
    *   Observed target image $I^*$ [S2, unofficial].
    *   Ambient illumination $L_a$ [S2, unofficial].
*   **Outputs:**
    *   Optimized light parameters $L = (L_i, I_{L,i})$, where $L_i$ is the 3D location of the light and $I_{L,i}$ is its scalar intensity [S2, unofficial].

**Data Structures**
*   **Point Cloud $X$:** Array of 3D coordinates $(x, y, z)$.
*   **Normals $N$:** Array of unit vectors corresponding to points in $X$.
*   **Light Parameters:** A vector containing the 3D position of the light source and its intensity.

**Rendering Model (Blinn-Phong with Shadows)**
The total image intensity $I$ at a point $x$ is the sum of ambient, diffuse, and specular terms [S2, unofficial]:
$$ I = I_a + I_d + I_s $$

1.  **Ambient Term:**
    $$ I_a = k_d \times L_a $$
    where $L_a$ is the ambient illumination and $k_d$ is the diffuse reflectance [S2, unofficial].

2.  **Diffuse Term:**
    $$ I_d = k_d \times S_i \times (L_i - X)^t \times N \times I_{L,i} $$
    where $S_i$ is the shadow term (1 if illuminated, 0 otherwise), $L_i$ is the light position, $X$ is the surface point, $N$ is the normal, and $I_{L,i}$ is the light intensity [S2, unofficial].

3.  **Specular Term:**
    $$ I_s = k_s \times S_i \times (H^t N)^\alpha \times I_{L,i} $$
    where $H$ is the halfway vector between the viewer direction $(C - X)$ and the light vector $(L_i - X)$, $k_s$ is the specular reflectance, and $\alpha$ is the shininess exponent [S2, unofficial].

4.  **Shadow Term $S_i$:**
    $S_i$ equals 1 when the light illuminates the point $X$ and 0 otherwise. This is determined by casting a ray from $X$ to $L_i$ and checking for occlusion [S2, unofficial].

**Optimization Objective**
The optimization problem minimizes the squared L2 norm of the difference between the rendered image $I(G, M, L)$ and the observed image $I^*$ with respect to the lighting parameters $L$ [S2, unofficial]:
$$ \hat{L} = \arg \min_L E = \arg \min_L \|I(G, M, L) - I^*\|_2^2 $$

The mean squared error (MSE) loss used is defined as [S1]:
$$ L(\theta) = \frac{1}{N} \sum_{i=1}^{N} (I_i(\theta) - T_i)^2 $$
where $I(\theta) \in [0, 1]^{H \times W \times C}$ is the rendered image normalized to $[0, 1]$, $T \in [0, 1]^{H \times W \times C}$ is the target image, and $N = H \cdot W \cdot C$ is the total number of pixel-channel values [S1].

**Optimization Procedure**
The problem is solved using the Adam optimizer [S1]. The update step for parameter $\theta$ at iteration $t$ is [S1]:
$$ \hat{m}_t = \frac{m_t}{1 - \beta_1^t}, \quad \hat{v}_t = \frac{v_t}{1 - \beta_2^t} $$
$$ \theta_{t+1} = \theta_t - \eta \frac{\hat{m}_t}{\sqrt{\hat{v}_t + \epsilon}} $$
where $m_t, v_t$ are biased first and second moment estimates, and $\eta, \beta_1, \beta_2, \epsilon$ are optimizer hyperparameters [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_rendering_optimization(G, M, I_star, L_a, theta_0, N_max, epsilon_loss, optimizer_config):
    """
    G: Scene geometry (X, N, C)
    M: Material properties (k_d, k_s, alpha)
    I_star: Observed target image
    L_a: Ambient illumination
    theta_0: Initial scene parameters (including light position and intensity)
    N_max: Maximum iterations
    epsilon_loss: Loss tolerance for convergence
    optimizer_config: Hyperparameters for Adam (eta, beta_1, beta_2, epsilon)
    """
    theta = theta_0
    converged = False
    iter_count = 0
    
    # Initialize Adam state
    m = zeros_like(theta)
    v = zeros_like(theta)
    t = 0
    
    while not converged and iter_count < N_max:
        # 1. Forward Pass: Render image
        I_rendered = render(G, M, L_a, theta)
        
        # 2. Compute Loss
        # N = H * W * C
        loss = mean((I_rendered - I_star) ** 2)
        
        # 3. Check Convergence
        if loss < epsilon_loss:
            converged = True
            break
            
        # 4. Compute Gradients via Automatic Differentiation
        # The renderer must support backpropagation through the Blinn-Phong model
        # and shadow casting logic.
        grad = compute_gradients(loss, theta)
        
        # 5. Update Parameters using Adam
        t += 1
        m = beta_1 * m + (1 - beta_1) * grad
        v = beta_2 * v + (1 - beta_2) * (grad ** 2)
        
        m_hat = m / (1 - beta_1 ** t)
        v_hat = v / (1 - beta_2 ** t)
        
        theta = theta - eta * (m_hat / (sqrt(v_hat) + epsilon))
        
        # 6. Project material parameters if optimizing colors
        # If optimizing RGB albedo, project onto [0, 1]^3
        if optimizing_material_colors:
            theta.materials = clamp(theta.materials, 0, 1)
            
        iter_count += 1
        
    return theta
```

**Parameters**
*   **Adam Hyperparameters:** Used consistently across camera calibration, light estimation, and material recovery tasks. Specific values are not listed in the sources but were selected for stable, monotone convergence [S1].
*   **Finite Difference Perturbation:** For gradient verification, a perturbation size of $h = 1 \times 10^{-5}$ relative to the parameter value is used ($\Delta \theta_i = h \cdot |\theta_i|$ for $\theta_i \neq 0$, and $\Delta \theta_i = h$ for $\theta_i = 0$) [S1].
*   **Gradient Error Tolerance:** $\epsilon = 10^{-8}$ prevents division by zero in gradient error calculations [S1].

**Edge Cases and Pitfalls**
*   **Imperfect Reflectance:** The method handles imperfect reflectance estimation by including cast shadows in the optimization, which increases robustness compared to methods that do not use shadows or explicitly separate diffuse/specular components [S2, unofficial].
*   **Gradient Verification:** Gradient approximation error is quantified using relative $\ell_2$ error: $E_{grad} = \frac{\|\nabla_{AD} L - \nabla_{FD} L\|_2}{\|\nabla_{FD} L\|_2 + \epsilon}$, where $\nabla_{AD}$ is the AD gradient and $\nabla_{FD}$ is the finite-difference gradient [S1]. This error remains below $10^{-4}$ for AD across tested scene complexities [S1].

## Operating characteristics
*   **Time Complexity:** The computational cost scales with the number of operations in the forward pass rather than with the number of parameters, making it substantially more efficient for high-dimensional optimization problems [S1].
*   **Measured Performance:**
    *   **Camera Calibration:** MSE loss fell from >5700 to <100 within 150 iterations on a 512x512 image. Final position error was $0.008 \pm 0.002$ units (0.20 ± 0.05% relative error). Converged in $147 \pm 12$ iterations over 10 runs [S1].
    *   **Light Estimation:** Converged in <500 iterations. Position error <2% of ground truth. Intensity error <5% of true value [S1].
    *   **Material Estimation:** Final MSE < $1 \times 10^{-4}$ in RGB space. Initial RGB error $0.43 \pm 0.12$ reduced to $0.004 \pm 0.001$ at convergence (within 50 iterations) [S1].
*   **Strengths:** Robust to imperfect reflectance estimation due to shadow modeling; does not require explicit separation of specular/diffuse components or image segmentation [S2, unofficial].
*   **Limitations:** Requires a differentiable renderer capable of computing gradients through shadow casting and the Blinn-Phong model [S2, unofficial].

## Background
Inverse rendering was introduced as a method to recover latent scene parameters by formulating the problem as an unconstrained optimization task using gradients from the rendering pipeline [S1]. The specific approach of robust point light source estimation using differentiable rendering was proposed to address the need for consistent lighting in mixed reality applications, such as inserting virtual objects into real scenes [S2, unofficial]. It differs from prior optimization-based methods like Neverova et al. [22] by incorporating cast shadows and avoiding explicit component separation [S2, unofficial].

## Sources
[S1] Optimizing 3D Scene Reconstruction with Differentiable Ray Tracing, https://doi.org/10.31224/5014, fetched 2026-10-29
[S2] Robust Point Light Source Estimation Using Differentiable Rendering, https://doi.org/10.48550/arxiv.1812.04857, fetched 2026-10-29 (unofficial)
