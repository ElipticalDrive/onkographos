# 4D Scale-adaptive Filter

Also called: Adaptive Scale Filter for 4D Gaussians

## Overview
The 4D Scale-adaptive Filter is a filtering technique for 4D Gaussian Splatting that prevents the distortion of Gaussian anisotropy and eliminates the rendering of invisible primitives during dynamic scene reconstruction [S1, unofficial]. It distinguishes itself from fixed-scale smoothing filters by adaptively adjusting the dilation coefficient based on the primitive's current scale and visibility, thereby preserving high-frequency details while avoiding aliasing artifacts caused by over-dilation [S1, unofficial].

## How it works
The method modifies the covariance matrix of 4D Gaussian primitives during the splatting pipeline [S1, unofficial]. It operates by computing an adaptive dilation scale $\sigma_{\text{adapt}}$ that depends on the primitive's intrinsic scale, a minimum dilation ratio, and a visibility threshold [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   $\mathbf{G}_k$: The 4D Gaussian primitive $k$, defined by position $\mathbf{p}_k$, covariance $\mathbf{\Sigma}_k(t)$, opacity, and color [S1, unofficial].
    *   $s_t$: The current scale of the Gaussian in a specific dimension at time $t$ [S1, unofficial].
    *   $\hat{\nu}_k$: The estimated maximum sampling frequency for primitive $k$ [S1, unofficial].
    *   $\sigma_s$: A base scale factor [S1, unofficial].
    *   $\rho_{\min}, \rho_{\max}$: Minimum and maximum dilation ratios [S1, unofficial].
    *   $\rho_{\text{thre}}$: Threshold for scale-based dilation gating [S1, unofficial].
    *   $\epsilon$: Minimal dilation scale for invisible Gaussians [S1, unofficial].
*   **Outputs:**
    *   Filtered covariance $\mathbf{\Sigma}'_k(t)$ and amplitude scaling factor [S1, unofficial].
    *   Visibility mask $M$ determining if the Gaussian contributes to the image [S1, unofficial].

### Data Structures
The 4D Gaussian primitive $k$ maintains:
*   $\mathbf{p}_k \in \mathbb{R}^3$: Position [S1, unofficial].
*   $\mathbf{\Sigma}_k(t) \in \mathbb{R}^{4 \times 4}$: Covariance matrix in 4D space (3D space + time) [S1, unofficial].
*   $\hat{\nu}_k \in \mathbb{R}$: Estimated maximum sampling frequency [S1, unofficial].
*   $s_t \in \mathbb{R}$: Current scale magnitude (derived from $\mathbf{\Sigma}_k$) [S1, unofficial].

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
    Calculate the proportional change in the dilation coefficient $\rho_{\text{adapt}}$ based on the scale variation ratio [S1, unofficial]:
    $$ \rho_{\text{adapt}} = \text{clip}\left( \frac{s_t^2}{\hat{\nu}_k^2}, \rho_{\min}, \rho_{\max} \right) $$
    [S1, unofficial].

3.  **Determine Adaptive Scale $\sigma_{\text{adapt}}$:**
    Apply the masking operation $M$ to determine the final adaptive dilation scale [S1, unofficial]. If the current scale $s_t$ is above the threshold $\rho_{\text{thre}} \cdot \sigma_s$, use the adaptive ratio; otherwise, use the minimal scale $\epsilon$ to prevent invisible Gaussians from becoming visible [S1, unofficial]:
    $$ \sigma_{\text{adapt}} = \begin{cases} \rho_{\text{adapt}} \cdot \sigma_s & \text{if } s_t \ge \rho_{\text{thre}} \cdot \sigma_s \\ \epsilon \cdot \sigma_s & \text{otherwise} \end{cases} $$
    This defines the masking operation $M$ (Eq. 12) [S1, unofficial].

4.  **Apply Filter to Covariance:**
    Update the covariance matrix $\mathbf{\Sigma}_k(t)$ by adding the adaptive dilation term $\sigma_{\text{adapt}}^2 \mathbf{I}$ to the exponent [S1, unofficial]:
    $$ \mathbf{\Sigma}'_k(t) = \mathbf{\Sigma}_k(t) + \sigma_{\text{adapt}}^2 \mathbf{I} $$
    Scale the amplitude by $\sqrt{|\mathbf{\Sigma}_k(t)| / |\mathbf{\Sigma}_k(t) + \sigma_{\text{adapt}}^2 \mathbf{I}|}$ to prevent distortion of the scale ratio between dimensions and avoid dilation when the primitive is invisible [S1, unofficial].

5.  **Integrate Masking into Splatting:**
    Integrate the masking operation $M$ into the splatting pipeline to avoid rendering invisible Gaussians [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def identity(n):
    """Returns an n x n identity matrix."""
    return [[1 if i == j else 0 for j in range(n)] for i in range(n)]

def det(M):
    """Computes the determinant of a square matrix M recursively."""
    n = len(M)
    if n == 1:
        return M[0][0]
    if n == 2:
        return M[0][0] * M[1][1] - M[0][1] * M[1][0]
    return sum((-1) ** i * M[0][i] * det([row[:i] + row[i+1:] for row in M[1:]]) for i in range(n))

def apply_4d_scale_adaptive_filter(gaussian_k, time_t, params):
    """
    Applies the 4D Scale-adaptive Filter to a single 4D Gaussian primitive.
    
    Args:
        gaussian_k: Object containing p_k, Sigma_k, s_t (current scale).
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
    s_t = gaussian_k.s_t
    Sigma_k = gaussian_k.Sigma_k
    
    # 2. Compute adaptive dilation ratio rho_adapt
    ratio = (s_t ** 2) / (nu_hat_k ** 2)
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
    I_4 = identity(4)
    Sigma_filtered = [[Sigma_k[i][j] + (sigma_adapt ** 2) * I_4[i][j] for j in range(4)] for i in range(4)]
    
    det_original = det(Sigma_k)
    det_filtered = det(Sigma_filtered)
    
    # Guard against numerical instability
    if det_original <= 0 or det_filtered <= 0:
        amplitude_scale = 1.0
    else:
        amplitude_scale = (det_original / det_filtered) ** 0.5
        
    return Sigma_filtered, amplitude_scale, is_visible
```

### Edge Cases and Pitfalls
*   **Precision:** Determinant calculations for 4D matrices can suffer from numerical instability if the Gaussian is highly anisotropic or degenerate. Guard against negative determinants due to floating-point errors [G].
*   **Threshold Sensitivity:** The parameter $\rho_{\text{thre}}$ is critical. A small value reduces sensitivity to lighting but may allow unwanted dilation; a large value may overly constrain the Gaussian [S1, unofficial].
*   **Invisible Gaussians:** Without the masking operation $M$, dilation could make imperceptible Gaussians visible, introducing noise. The $\epsilon$ term prevents this [S1, unofficial].
*   **Anisotropy Distortion:** Fixed-scale dilation distorts the scale ratio between dimensions. The adaptive scaling $\rho_{\text{adapt}}$ preserves the dimensional scale ratio [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The filter adds a constant-time operation per Gaussian per frame (matrix addition, determinant calculation, clipping) [G]. It does not increase the asymptotic complexity of the splatting pipeline, which is dominated by rasterization [G].
*   **Memory Complexity:** No additional memory is required beyond the existing Gaussian attributes. The frequency estimate $\hat{\nu}_k$ requires one scalar per primitive [S1, unofficial].
*   **Performance:** The filter improves anti-aliasing performance, particularly in monocular video reconstruction tasks, outperforming Mip-Splatting at higher resolutions when combined with scale regularization [S1, unofficial]. It reduces unnecessary computation by masking invisible Gaussians [S1, unofficial].
*   **Strengths:** Preserves Gaussian anisotropy; prevents dilation of invisible primitives; mitigates aliasing artifacts from scale variations [S1, unofficial].
*   **Limitations:** Sensitive to the hyperparameter $\rho_{\text{thre}}$. Requires accurate estimation of $\hat{\nu}_k$, which depends on camera parameters and deformation field stability [S1, unofficial].

## Background
The 4D Scale-adaptive Filter was introduced in the context of 4D Gaussian Splatting to address artifacts caused by fixed-scale dilation in dynamic scenes [S1, unofficial]. It builds on the principles of 2D Gaussian Splatting [6068-2d-gaussian-splatting.md] but extends them to 4D space [S1, unofficial]. The method was proposed to overcome limitations of 3D smoothing filters, which do not account for visibility or anisotropy preservation in 4D [S1, unofficial].

## Sources
[S1] Alias-free 4D Gaussian Splatting, https://arxiv.org/abs/2511.18367v1, fetched 2026-10-26 (unofficial)
