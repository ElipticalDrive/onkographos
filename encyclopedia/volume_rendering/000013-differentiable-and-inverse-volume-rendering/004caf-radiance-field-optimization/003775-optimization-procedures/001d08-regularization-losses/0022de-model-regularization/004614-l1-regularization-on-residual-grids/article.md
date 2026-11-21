# L1 Regularization on Residual Grids

Also called: JointRF Training Objective, Residual Grid L1 Regularization

## Overview
This method applies L1 regularization to residual grids ($R_t$) during the optimization of dynamic radiance fields to enforce temporal continuity and minimize the magnitude of temporal changes [S1]. It is a core component of the JointRF training objective, distinguishing itself from static regularization by explicitly targeting the temporal residual structure to facilitate compression and stability [S1]. It builds on [Model Regularization](2912-model-regularization.md).

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ground truth color $c(r)$, rendered color $\hat{c}(r)$, residual grid $R_t$, coefficient grid $C_t$, and hyperparameters $\lambda_1, \lambda_2$.
*   **Outputs:** Total loss value $L_{total}$ used for gradient backpropagation.

**Data Structures:**
*   $R_t$: Residual grid representing temporal differences.
*   $\hat{R}_t$: Quantized version of $R_t$.
*   $\hat{C}_t$: Quantized coefficient grid.
*   $P_1, P_2$: Probability Mass Functions (PMFs) for $\hat{R}_t$ and $\hat{C}_t$ respectively, used for rate estimation.

**Formulas:**
The total loss is defined as [S1]:
$$
L_{total} = L_{mse} + \lambda_1 L_{rate} + \lambda_2 ||R_t||_1
$$

Where:
1.  **Distortion Loss ($L_{mse}$):** Measures the difference between ground truth and rendered results [S1]:
    $$
    L_{mse} = \sum ||c(r) - \hat{c}(r)||_2
    $$
2.  **Rate Loss ($L_{rate}$):** Estimated rate derived from quantized grids [S1]:
    $$
    L_{rate} = -\frac{1}{N} \sum_{\hat{y} \in \hat{R}_t} \log_2(P_1(\hat{y})) - \frac{1}{M} \sum_{\hat{y} \in \hat{C}_t} \log_2(P_2(\hat{y}))
    $$
    Here, $N$ and $M$ are the number of elements in $\hat{R}_t$ and $\hat{C}_t$ respectively.
3.  **Residual Regularization ($||R_t||_1$):** The L1 norm of the residual grid [S1]:
    $$
    ||R_t||_1 = \sum |R_t|
    $$

**Parameters:**
*   $\lambda_1$: Balances rate and distortion, controlling model size and reconstruction quality. Initialized at $0.000001$ and decrements progressively [S1].
*   $\lambda_2$: Measures the extent of constraint on $R_t$. Initialized at $0.000001$ [S1].

**Steps:**
1.  Render the scene to obtain $\hat{c}(r)$ given current radiance field parameters.
2.  Compute $L_{mse}$ by comparing $\hat{c}(r)$ with ground truth $c(r)$.
3.  Compute the residual grid $R_t$ (implementation detail: typically $R_t$ is derived from temporal differences in the latent or feature space before quantization).
4.  Quantize $R_t$ to $\hat{R}_t$ and $C_t$ to $\hat{C}_t$ to estimate PMFs $P_1$ and $P_2$.
5.  Compute $L_{rate}$ using the PMFs and quantized values.
6.  Compute $||R_t||_1$ as the sum of absolute values of the residual grid.
7.  Combine losses: $L_{total} = L_{mse} + \lambda_1 L_{rate} + \lambda_2 ||R_t||_1$.
8.  Backpropagate $L_{total}$ to update model parameters.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_jointrf_loss(c_gt, c_rendered, R_t, C_t, lambda_1, lambda_2):
    # 1. MSE Loss
    # Note: ||.||_2 in source likely refers to L2 norm per pixel/vector, summed over all pixels
    mse_loss = sum(l2_norm(c_gt[r] - c_rendered[r]) for r in all_pixels)
    
    # 2. Rate Loss
    # Quantize R_t and C_t to get histograms/PMFs
    # P1 is PMF of quantized R_t, P2 is PMF of quantized C_t
    # N and M are total counts
    hat_R_t = quantize(R_t)
    hat_C_t = quantize(C_t)
    
    P1 = compute_pmf(hat_R_t)
    P2 = compute_pmf(hat_C_t)
    
    rate_loss = 0.0
    N = len(hat_R_t)
    M = len(hat_C_t)
    
    if N > 0:
        for y in hat_R_t:
            p = P1[y]
            if p > 0:
                rate_loss -= log2(p)
        rate_loss /= N
        
    if M > 0:
        for y in hat_C_t:
            p = P2[y]
            if p > 0:
                rate_loss -= log2(p)
        rate_loss /= M
        
    # 3. L1 Regularization on Residual Grid
    l1_reg = sum(abs(val) for val in R_t.flatten())
    
    # 4. Total Loss
    total_loss = mse_loss + lambda_1 * rate_loss + lambda_2 * l1_reg
    
    return total_loss
```

**Edge Cases and Pitfalls:**
*   **Quantization:** The rate loss depends on accurate PMF estimation. If quantization bins are too sparse or too dense, the rate estimate may be inaccurate [S1].
*   **Parameter Decay:** $\lambda_1$ is decremented progressively. Failure to decay may over-penalize rate, leading to poor reconstruction [S1].
*   **Residual Magnitude:** If $R_t$ is not properly normalized, the L1 term may dominate or be negligible relative to MSE [S1].

## Subtopics

### L1 Time Plane Regularization
A regularizer that encourages time planes to be close to 1 by penalizing the L1 distance from 1 [S2, unofficial]. It identifies spatiotemporal grids: if grid length is 3, skip; otherwise, spatiotemporal grids are indices [2, 4, 5] [S2, unofficial]. For each spatiotemporal grid, it computes the mean of $|1 - \text{grid}|$ and adds it to the total loss [S2, unofficial]. This is used in 4D Gaussian Splatting variants [S2, unofficial].

## Operating characteristics
*   **Complexity:** The L1 term is $O(N)$ where $N$ is the number of elements in $R_t$. The rate loss requires histogramming and log computation, also roughly $O(N)$.
*   **Performance:** In JointRF experiments on ReRF (1920x1080) and DNA-Rendering (2048x2448) datasets, this regularization aids in compressing the dynamic representation while maintaining visual quality [S1].
*   **Strengths:** Explicitly targets temporal redundancy, facilitating compression [S1].
*   **Limitations:** Requires quantization for rate estimation, which may introduce artifacts if not tuned [S1].
*   **When to use:** When optimizing dynamic radiance fields for compression or when temporal smoothness is critical [S1].

## Background
This method originates from the JointRF paper, which proposes end-to-end joint optimization for dynamic neural radiance field representation and compression [S1]. It was presented in 2024 [S1]. The approach combines distortion, rate, and temporal regularization to balance quality and model size.

## Sources
[S1] JOINTRF: End-To-End Joint Optimization for Dynamic Neural Radiance Field Representation and Compression, https://doi.org/10.1109/icip51287.2024.10647336, fetched 2026-10-26
[S2] ElenaViewSynthesis/4dgs, https://github.com/ElenaViewSynthesis/4dgs, fetched 2026-10-28 (unofficial)
