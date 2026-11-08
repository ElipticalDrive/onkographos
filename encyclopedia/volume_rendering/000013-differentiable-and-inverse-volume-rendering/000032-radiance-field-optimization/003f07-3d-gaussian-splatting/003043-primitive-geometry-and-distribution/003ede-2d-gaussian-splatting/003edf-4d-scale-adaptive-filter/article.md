# 4D Scale-adaptive Filter

Also called: Adaptive Scale Filter for 4D Gaussians

## Overview
The 4D Scale-adaptive Filter is a post-processing and regularization technique for 4D Gaussian Splatting that prevents the distortion of Gaussian anisotropy and eliminates the rendering of invisible primitives during dynamic scene reconstruction [S1, unofficial]. It distinguishes itself from fixed-scale smoothing filters (e.g., [3D Smoothing Filter](link)) by adaptively adjusting the dilation coefficient based on the primitive's current scale and visibility, thereby preserving high-frequency details while avoiding aliasing artifacts caused by over-dilation [S1, unofficial].

## How it works
The method modifies the covariance matrix of 4D Gaussian primitives during the splatting pipeline. It operates by computing an adaptive dilation scale $\sigma_{\text{adapt}}$ that depends on the primitive's intrinsic scale, a minimum dilation ratio, and a visibility threshold.

### Inputs and Outputs
*   **Inputs:**
    *   $\mathbf{G}_k$: The 4D Gaussian primitive $k$, defined by position $\mathbf{p}_k$, covariance $\mathbf{\Sigma}_k(t)$, opacity, and color.
    *   $s_t$: The current scale of the Gaussian in a specific dimension at time $t$.
    *   $\hat{\nu}_k$: The estimated maximum sampling frequency for primitive $k$.
    *   $\sigma_s$: A base scale factor.
    *   $\rho_{\min}, \rho_{\max}$: Minimum and maximum dilation ratios.
    *   $\rho_{\text{thre}}$: Threshold for scale-based dilation gating.
    *   $\epsilon$: Minimal dilation scale for invisible Gaussians.
*   **Outputs:**
    *   Filtered covariance $\mathbf{\Sigma}'_k(t)$ and amplitude scaling factor.
    *   Visibility mask $M$ determining if the Gaussian contributes to the image.

### Data Structures
The 4D Gaussian primitive $k$ maintains:
*   $\mathbf{p}_k \in \mathbb{R}^3$: Position.
*   $\mathbf{\Sigma}_k(t) \in \mathbb{R}^{4 \times 4}$: Covariance matrix in 4D space (3D space + time).
*   $\hat{\nu}_k \in \mathbb{R}$: Estimated maximum sampling frequency.
*   $s_t \in \mathbb{R}$: Current scale magnitude (derived from $\mathbf{\Sigma}_k$).

### Parameters
*   $\rho_{\min}$: Minimum dilation ratio. Controls the lower bound of adaptive scaling. In ablation studies, a smaller $\rho_{\min}$ is used to mitigate negative impacts on full-resolution reconstruction [S1, unofficial].
*   $\sigma_s$: Base scale factor. The minimal adaptive scale is computed as $\rho_{\min} \cdot \sigma_s$ [S1, unofficial].
*   $\rho_{\max}$: Maximum dilation ratio. Constrains the expansion of enlarged dimensions to preserve anisotropy [S1, unofficial].
*   $\rho_{\text{thre}}$: Critical scale threshold. Determines when dilation is prevented and scale loss constraints are applied. Sensitivity to this hyperparameter affects lighting sensitivity [S1, unofficial].
*   $\epsilon$: Minimal dilation scale for invisible primitives. Prevents imperceptible Gaussians from becoming visible due to dilation [S1, unofficial].

### Steps
1.  **Estimate Sampling Frequency:**
    Compute the rough minimum sampling interval $\hat{T}_k$ using initial positions $\mathbf{p}_k$ and camera parameters $f_n, d_n$ [S1, unofficial]:
    $$ \hat{T}_k = \min_{n \in \{1, \dots, N\}} \left( \frac{1}{f_n} \cdot \frac{d_n}{\mathbf{1}_n(\mathbf{p}_k)} \right) $$
    Once the deformation field stabilizes, update the sampling frequency estimate $\hat{\nu}_k$ using momentum [S1, unofficial]. The maximum sampling frequency is determined by minimizing the ratio of distance to frequency across all cameras $N$:
    $$ \hat{\nu}_k = \min_{n \in \{1, \dots, N\}} \left( \frac{1}{f_n(t)} \cdot \frac{d_n(t)}{\mathbf{1}_n(\mathbf{p}_k(t))} \right) $$
    where $\mathbf{1}_n(\mathbf{p}_k)$ is an indicator function (1 if visible, 0 otherwise) [S1, unofficial].

2.  **Compute Adaptive Dilation Ratio:**
    Calculate the proportional change in the dilation coefficient $\rho_{\text{adapt}}$ based on the scale variation ratio $s_t / \hat{\nu}_k$. Clip this ratio between $\rho_{\min}$ and $\rho_{\max}$ [S1, unofficial]:
    $$ \rho_{\text{adapt}} = \text{clip}\left( \frac{s_t}{\hat{\nu}_k}, \rho_{\min}, \rho_{\max} \right) $$
    *Note: The source notation $\text{clip}_t^2$ in Eq. 11 appears to be a typo for clipping the ratio $s_t/\hat{\nu}_k$ or similar scale-frequency metric. The text confirms $\rho_{\text{adapt}}$ is computed according to the scale variation ratio.*

3.  **Determine Adaptive Scale $\sigma_{\text{adapt}}$:**
    Apply the masking operation $M$ to determine the final adaptive dilation scale. If the current scale $s_t$ is above the threshold $\rho_{\text{thre}} \cdot \sigma_s$, use the adaptive ratio; otherwise, use the minimal scale $\epsilon$ to prevent invisible Gaussians from becoming visible [S1, unofficial]:
    $$ \sigma_{\text{adapt}} = \begin{cases} \rho_{\text{adapt}} \cdot \sigma_s & \text{if } s_t \ge \rho_{\text{thre}} \cdot \sigma_s \\ \epsilon \cdot \sigma_s & \text{otherwise} \end{cases} $$
    This defines the masking operation $M$ (Eq. 12) [S1, unofficial].

4.  **Apply Filter to Covariance:**
    Update the covariance matrix $\mathbf{\Sigma}_k(t)$ by adding the adaptive dilation term $\sigma_{\text{adapt}}^2 \mathbf{I}$ to the exponent. Simultaneously, scale the amplitude (opacity/color contribution) to preserve energy [S1, unofficial]:
    $$ \mathbf{\Sigma}'_k(t) = \mathbf{\Sigma}_k(t) + \sigma_{\text{adapt}}^2 \mathbf{I} $$
    $$ \text{Amplitude Scale} = \sqrt{\frac{|\mathbf{\Sigma}_k(t)|}{|\mathbf{\Sigma}_k(t) + \sigma_{\text{adapt}}^2 \mathbf{I}|}} $$
    Here, $|\cdot|$ denotes the determinant. This step prevents distortion of the scale ratio between dimensions [S1, unofficial].

5.  **Integrate Masking into Splatting:**
    Integrate the masking operation $M$ into the splatting pipeline. Gaussians with $\sigma_{\text{adapt}} = \epsilon \cdot \sigma_s$ (invisible) are excluded from rendering, avoiding unnecessary splatting [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_4d_scale_adaptive_filter(gaussian_k, time_t, params):
    """
    Applies the 4D Scale-adaptive Filter to a single 4D Gaussian primitive.
    
    Args:
        gaussian_k: Object containing p_k, Sigma_k, s_t (current scale), 
                    and visibility status.
        time_t: Current time step.
        params: Dictionary containing rho_min, rho_max, rho_thre, 
                sigma_s, epsilon, and estimated frequency nu_hat_k.
                
    Returns:
        filtered_sigma: Filtered covariance matrix.
        amplitude_scale: Scaling factor for opacity/color.
        is_visible: Boolean mask indicating if the Gaussian should be splatted.
    """
    # 1. Get parameters
    rho_min = params['rho_min']
    rho_max = params['rho_max']
    rho_thre = params['rho_thre']
    sigma_s = params['sigma_s']
    epsilon = params['epsilon']
    nu_hat_k = params['nu_hat_k']
    s_t = gaussian_k.s_t  # Current scale in specific dimension
    
    # 2. Compute adaptive dilation ratio rho_adapt
    # Clip the scale-to-frequency ratio
    ratio = s_t / nu_hat_k
    rho_adapt = max(rho_min, min(rho_max, ratio))
    
    # 3. Determine adaptive scale sigma_adapt based on threshold
    threshold_val = rho_thre * sigma_s
    if s_t >= threshold_val:
        sigma_adapt = rho_adapt * sigma_s
        is_visible = True
    else:
        sigma_adapt = epsilon * sigma_s
        is_visible = False
        
    # 4. Apply filter to covariance
    # Sigma_k is 4x4 matrix. I is 4x4 identity.
    Sigma_k = gaussian_k.Sigma_k
    determinant_original = det(Sigma_k)
    
    # Add dilation term to covariance
    Sigma_filtered = Sigma_k + (sigma_adapt ** 2) * identity(4)
    
    # Calculate amplitude scaling factor
    determinant_filtered = det(Sigma_filtered)
    
    # Avoid division by zero or invalid sqrt
    if determinant_original <= 0 or determinant_filtered <= 0:
        amplitude_scale = 1.0
    else:
        amplitude_scale = sqrt(determinant_original / determinant_filtered)
        
    return Sigma_filtered, amplitude_scale, is_visible
```

### Edge Cases and Pitfalls
*   **Precision:** Determinant calculations for 4D matrices can suffer from numerical instability if the Gaussian is highly anisotropic or degenerate. Guard against negative determinants due to floating-point errors [G].
*   **Threshold Sensitivity:** The parameter $\rho_{\text{thre}}$ is critical. A small value reduces sensitivity to lighting but may allow unwanted dilation; a large value may overly constrain the Gaussian [S1, unofficial].
*   **Invisible Gaussians:** Without the masking operation $M$, dilation could make imperceptible Gaussians visible, introducing noise. The $\epsilon$ term prevents this [S1, unofficial].
*   **Anisotropy Distortion:** Fixed-scale dilation distorts the scale ratio between dimensions. The adaptive scaling $\rho_{\text{adapt}}$ preserves the dimensional scale ratio [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The filter adds a constant-time operation per Gaussian per frame (matrix addition, determinant calculation, clipping). It does not increase the asymptotic complexity of the splatting pipeline, which is dominated by rasterization.
*   **Memory Complexity:** No additional memory is required beyond the existing Gaussian attributes. The frequency estimate $\hat{\nu}_k$ requires one scalar per primitive.
*   **Performance:** The filter improves anti-aliasing performance, particularly in monocular video reconstruction tasks, outperforming Mip-Splatting at higher resolutions when combined with scale regularization [S1, unofficial]. It reduces unnecessary computation by masking invisible Gaussians.
*   **Strengths:** Preserves Gaussian anisotropy; prevents dilation of invisible primitives; mitigates aliasing artifacts from scale variations.
*   **Limitations:** Sensitive to the hyperparameter $\rho_{\text{thre}}$. Requires accurate estimation of $\hat{\nu}_k$, which depends on camera parameters and deformation field stability [S1, unofficial].

## Background
The 4D Scale-adaptive Filter was introduced in the context of 4D Gaussian Splatting to address artifacts caused by fixed-scale dilation in dynamic scenes. It builds on the principles of 2D Gaussian Splatting [6068-2d-gaussian-splatting.md] but extends them to 4D space. The method was proposed to overcome limitations of 3D smoothing filters, which do not account for visibility or anisotropy preservation in 4D [S1, unofficial].

## Sources
[S1] Alias-free 4D Gaussian Splatting, https://arxiv.org/abs/2511.18367v1, fetched 2026-10-26 (unofficial)
