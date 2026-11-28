# Depth-guided ray sampling

Also called: Confidence and Depth-Guided Sampling, Depth Prior Sampling, Pd sampling

## Overview
Depth-guided ray sampling is a strategy that uses the standard deviation of depth values in pixel neighborhoods to guide ray sampling, targeting areas with abrupt depth changes [S2]. It complements pixel-guided sampling by focusing on geometric boundaries rather than color variation, ensuring sample points are concentrated near object surfaces where depth uncertainty is high [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Depth probability volume $P_i(u, v)$ from a coarse-to-fine estimator (e.g., ENeRF) [S2]; reference view ray origin $o$ and direction $g$ [S1]; near plane $e_n$ and far plane $e_f$ [S1]; number of samples $M$ [S1]; neighborhood size $n \times n$ [S2]; fusion weight $\beta$ [S2].
*   **Outputs:** A set of $M$ sample distances along the ray [S1].

**Data Structures:**
*   **Depth Probability Volume:** A 3D tensor or array where $P_i(u, v)$ represents the probability of depth at pixel $(u, v)$ for depth plane $L_i$.
*   **Depth Map:** A 2D array $D(u, v)$ storing the expected depth for each pixel.
*   **Sampling Probability Map:** A 2D array $P'_d(u, v)$ storing the normalized sampling probability derived from depth standard deviation.

**Steps:**

1.  **Compute Expected Depth:** For each pixel $(u, v)$, calculate the expected depth $D(u, v)$ using the depth probability distribution [S2]:
    $$D(u, v) = \sum_{i=1}^{N_d} P_i(u, v) \cdot L_i(u, v)$$
    where $N_d$ is the number of sampled depth planes, and $L_i$ are the depth planes sampled from $[L_{min}, L_{max}]$ [S2].

2.  **Compute Depth Standard Deviation:** Calculate the standard deviation of depth values within an $n \times n$ neighborhood centered at $(u, v)$ to determine the sampling probability $P_d(u, v)$ [S2]:
    $$P_d(u, v) = \text{std}(D(u, v)) = \sqrt{\frac{1}{n^2} \sum_{x', y'} [D(x', y') - \bar{D}]^2}$$
    where $\bar{D}$ is the mean depth of the $n \times n$ neighborhood, and $(x', y')$ iterates over the neighborhood pixels [S2].

3.  **Normalize Probability:** Apply the same normalization operation as pixel-guided sampling to $P_d(u, v)$ to obtain $P'_d(u, v)$, avoiding underfitting for certain rays [S2].

4.  **Calculate Confidence Interval (Optional/Alternative Formulation):** In some implementations, the standard deviation $\hat{S}(p)$ is calculated directly from the depth probability distribution $P_j(p)$ for pixel $p$ [S1]:
    $$\hat{S}(p) = \sqrt{\sum_{j=1}^{J} P_j(p) (\hat{D}_j(p) - \hat{D}(p))^2}$$
    where $\hat{D}(p)$ is the estimated depth and $\hat{D}_j(p)$ are depth values at discrete planes [S1]. The confidence range is defined as $\hat{U}(p) = [\hat{D}(p) - \hat{S}(p), \hat{D}(p) + \hat{S}(p)]$ [S1].

5.  **Sample Points:** Distribute the $M$ sample points along the ray $r_p = o + e g$ [S1]:
    *   Distribute half of the $M$ points uniformly between the near plane $e_n$ and far plane $e_f$ [S1].
    *   Extract the second half of the sampled points within the confidence and depth prior range $\hat{U}(p)$ (or guided by $P'_d(u, v)$) [S1, S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def depth_guided_sampling(depth_prob_volume, L_planes, n, M, e_n, e_f, o, g, beta=None, P_c=None):
    """
    depth_prob_volume: 3D array [H, W, N_d] representing P_i(u, v)
    L_planes: 1D array of length N_d representing depth planes L_i
    n: int, neighborhood size for std calculation
    M: int, total number of samples
    e_n: float, near plane distance
    e_f: float, far plane distance
    o: 3D vector, ray origin
    g: 3D vector, ray direction
    beta: float, weight for pixel-guided sampling (if None, only depth-guided)
    P_c: 2D array [H, W], normalized pixel-guided probability (if beta is provided)
    """
    H, W, N_d = depth_prob_volume.shape
    
    # Step 1: Compute Expected Depth D(u, v)
    # D(u, v) = sum(P_i(u, v) * L_i)
    # L_planes is broadcasted to [1, 1, N_d]
    D = np.sum(depth_prob_volume * L_planes, axis=2) # Shape: [H, W]
    
    # Step 2: Compute Depth Standard Deviation P_d(u, v)
    # Calculate std over n x n neighborhood
    # Using a sliding window or convolution with a uniform kernel
    mean_kernel = np.ones((n, n)) / (n * n)
    
    # Compute mean depth in neighborhood
    D_mean = convolve2d(D, mean_kernel, mode='same')
    
    # Compute variance in neighborhood
    # Variance = E[X^2] - (E[X])^2, but here we compute std of D values in neighborhood
    # P_d(u, v) = sqrt( (1/n^2) * sum( (D(x',y') - D_bar)^2 ) )
    # This is equivalent to the standard deviation of the local patch of D
    D_std = np.zeros_like(D)
    for u in range(H):
        for v in range(W):
            y_start = max(0, u - n // 2)
            y_end = min(H, u + n // 2 + 1)
            x_start = max(0, v - n // 2)
            x_end = min(W, v + n // 2 + 1)
            
            patch = D[y_start:y_end, x_start:x_end]
            # Adjust kernel size if patch is smaller than n x n at boundaries
            patch_size = patch.size
            D_std[u, v] = np.std(patch) # np.std computes sqrt(mean((x - x.mean())^2))
            
    P_d = D_std
    
    # Step 3: Normalize P_d to get P'_d
    # Normalization operation as in pixel-guided sampling (typically min-max or softmax-like)
    # Assuming min-max normalization to [0, 1] for simplicity, or as per source [S2]
    P_min = P_d.min()
    P_max = P_d.max()
    if P_max - P_min > 1e-8:
        P_prime_d = (P_d - P_min) / (P_max - P_min)
    else:
        P_prime_d = np.zeros_like(P_d)
        
    # Step 4: Determine Sampling Distribution
    if beta is not None and P_c is not None:
        # Weighted fusion with pixel-guided sampling
        P_samp = beta * P_c + (1 - beta) * P_prime_d
    else:
        P_samp = P_prime_d
        
    # Step 5: Sample Points
    # Ray definition: r_p = o + e * g
    # Half points between e_n and e_f
    half_M = M // 2
    e_uniform = np.linspace(e_n, e_f, half_M)
    
    # Second half points guided by P_samp
    # The source [S1] says "extracted within the range of the confidence and depth prior"
    # This implies sampling from a distribution defined by P_samp or within a specific depth range.
    # If using P_samp as a probability map, we sample indices (u, v) or distances e.
    # However, the context suggests sampling *distances* along the ray.
    # A common interpretation for "guided by probability" is to sample distances e
    # such that the probability of sampling at e is proportional to P_samp.
    # But P_samp is per-pixel. For a single ray, we need a 1D distribution.
    # The source [S1] mentions a range U(p). Let's assume we sample uniformly within U(p)
    # if a range is defined, or use P_samp to bias the uniform sampling.
    
    # Implementation detail: The source [S1] defines U(p) = [D_hat - S_hat, D_hat + S_hat]
    # and samples half points in [e_n, e_f] and half in U(p).
    # Let's implement the range-based sampling for the second half.
    
    # Calculate confidence range U(p) for the current pixel (u, v)
    # D_hat is D[u, v]
    # S_hat can be approximated by P_d[u, v] (the std computed above)
    D_hat = D[u, v]
    S_hat = P_d[u, v]
    
    e_near_bound = max(e_n, D_hat - S_hat)
    e_far_bound = min(e_f, D_hat + S_hat)
    
    # If the range is invalid or empty, fallback to uniform
    if e_near_bound >= e_far_bound:
        e_guided = np.linspace(e_n, e_f, M - half_M)
    else:
        e_guided = np.linspace(e_near_bound, e_far_bound, M - half_M)
        
    # Combine samples
    e_samples = np.concatenate([e_uniform, e_guided])
    
    # Return 3D points
    # r_p = o + e * g
    points = o + e_samples[:, np.newaxis] * g
    
    return points
```

**Parameters:**
*   $n$: Neighborhood size for standard deviation calculation. Typical value not given in sources.
*   $M$: Number of sample points. Typical value not given in sources.
*   $\beta$: Fusion weight for pixel-guided sampling. Typical value not given in sources.

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** When computing the $n \times n$ neighborhood at image boundaries, the patch size is smaller than $n^2$. The standard deviation calculation must account for this [S2].
*   **Invalid Confidence Range:** If the confidence range $\hat{U}(p)$ falls outside $[e_n, e_f]$ or is empty, fallback to uniform sampling or clamp the range [S1].
*   **Precision:** Depth probability volumes may suffer from quantization errors if $N_d$ is small [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the computation of the depth probability volume and the neighborhood standard deviation. The neighborhood operation is $O(H \cdot W \cdot n^2)$ or $O(H \cdot W)$ with separable filters.
*   **Memory Complexity:** Requires storing the depth probability volume $P_i(u, v)$ and the depth map $D(u, v)$.
*   **Strengths:** Targets areas with abrupt depth changes, improving reconstruction quality in geometrically complex regions [S2]. Ensures sample points are concentrated near object surfaces [S1].
*   **Limitations:** Relies on the accuracy of the depth estimation network. If depth estimation is poor, sampling may be misdirected [S1].
*   **When to Use:** When depth information is available and geometric boundaries are critical for rendering quality.
*   **When Not to Use:** When depth estimation is unreliable or when color variation is a better indicator of sampling importance.

## Background
Depth-guided ray sampling was introduced in the context of neural volume rendering and multi-view stereo. It builds on the ENeRF coarse-to-fine approach for depth prediction [S2]. The method was proposed to address the issue of uniform sampling not concentrating points on object surfaces, leading to decreased rendering quality [S1]. It is often used in conjunction with pixel-guided sampling via weighted fusion [S2].

## Sources
[S1] Multi-View Stereo Network Based on Attention Mechanism and Neural Volume Rendering, https://doi.org/10.3390/electronics12224603, fetched 2026-10-26
[S2] Efficient ray sampling for radiance fields reconstruction, https://doi.org/10.1016/j.cag.2023.11.005, fetched 2026-10-27
