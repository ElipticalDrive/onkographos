# Vanilla-NeRF Initialization Failure Analysis

Also called: ReLU initialization failure

## Overview
This analysis identifies the root causes of random training failures in standard (Vanilla) NeRF implementations, attributing them to poor initial weight distributions and the specific behavior of the ReLU activation function on density parameters [S1]. It contrasts this with robust initialization strategies that guarantee convergence [S1].

## How it works
**Inputs:**
*   MLP weights $W$ initialized from a standard distribution (e.g., Kaiming uniform or Xavier).
*   Scene data (camera poses, images).

**Outputs:**
*   Trained NeRF model (converged or failed).

**Data Structures:**
*   **MLP Layers:** Standard fully connected layers.
*   **Density Output:** The final layer output $\sigma$ before activation.

**Method Steps:**
1.  **Initialize MLP Weights:** Weights are typically drawn from a zero-mean distribution (e.g., Kaiming uniform). This results in an initial output distribution for the density $\sigma$ with 0 mean and 0 variance [S1].
2.  **Apply ReLU to Density:** The raw output $\sigma_{raw}$ is passed through a ReLU activation: $\sigma = \max(0, \sigma_{raw})$ [S1].
3.  **Compute Alpha:** Volume rendering computes transmittance $T$ and alpha $\alpha$ using $\sigma$.
4.  **Training:** Gradient descent updates weights.

**Failure Mechanism:**
*   **Zero Variance:** Because the initial weights have 0 mean and 0 variance, the initial $\sigma_{raw}$ is clustered near 0.
*   **ReLU Dead Zone:** ReLU maps negative values to 0. With $\sigma_{raw}$ centered at 0, a significant portion of the output is clamped to 0, preventing a smooth transition from low to high alpha values [S1].
*   **Random Failure:** This initialization sensitivity leads to random failure modes (non-convergence or poor quality) across different random seeds [S1].

**Robust Alternative (from [S1]):**
*   Use `exp` activation for density: $\sigma = \exp(\sigma_{raw})$.
*   Apply a high transmittance offset (bias) to the final layer to ensure initial $\sigma$ is small but non-zero, allowing smooth gradient flow [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Standard Vanilla NeRF Density Output (Prone to Failure)
def vanilla_density_output(raw_sigma):
    # raw_sigma comes from MLP with zero-mean initialization
    # ReLU clamps negative values to 0, causing issues with 0-variance init
    sigma = max(0.0, raw_sigma)
    return sigma

# Robust Density Output (Guaranteed Convergence per [S1])
def robust_density_output(raw_sigma, transmittance_offset=0.0):
    # exp ensures positive density and smooth gradients
    # transmittance_offset adjusts initial bias
    sigma = exp(raw_sigma + transmittance_offset)
    return sigma
```

**Edge Cases:**
*   **Low $k$ values:** Failure modes are more pronounced at lower scene complexity or specific $k$ values in Plenoxels comparisons [S1].
*   **Precision:** Standard float32 is assumed; no specific precision issues noted beyond standard numerical stability.

## Subtopics

### ReLU vs No-ReLU Variant
An ablation study removing the ReLU activation function to assess its impact on geometry capture and histogram consistency [S2]. Removing ReLU causes artifacts around sharp edges and drops all image metrics (PSNR, SSIM, LPIPS) due to inconsistent histograms and larger standard deviation in density values [S2]. Evaluated on the Mip-NeRF 360 dataset to verify ReLU's necessity for capturing fine geometry [S2].

### Low-Rank vs Random Initialization Ablation
A comparison between low-rank weight initialization and Kaiming uniform random initialization for distilled NeRF models [S3, unofficial]. Training a distilled NeRF with rank 40 using random weights (NeRF-rand) leads to poor reconstruction quality (severe blurriness) because the weight distribution differs significantly from low-rank weights (NeRF-lr) [S3, unofficial]. NeRF-lr has more values around 0 with long tails, while NeRF-rand has fewer values around 0 and weights mostly within $[-0.25, 0.25]$ without long tails [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Vanilla NeRF with default ReLU initialization shows random failure modes when trained 5 times on chair and ship scenes for each $k$-value [S1]. The proposed robust parametrization guarantees convergence on all runs across all tested $k$ values [S1].
*   **Strengths:** Identifies a critical failure mode in standard implementations.
*   **Limitations:** The analysis is specific to Vanilla NeRF and Plenoxels comparisons; results may not generalize to all neural rendering architectures.
*   **When to Use:** When debugging training instability in standard NeRF implementations.
*   **When Not to Use:** When using robust initialization strategies (exp activation, high transmittance offset) which mitigate these issues [S1].

## Background
Originated from analysis of Vanilla NeRF training instability [S1]. The failure modes were attributed to the interaction between zero-mean weight initialization and ReLU activation [S1]. The robust solution involves using `exp` activation and high transmittance initialization [S1].

## Sources
[S1] Alpha Invariance: On Inverse Scaling Between Distance and Volume Density in Neural Radiance Fields, https://www.youtube.com/watch?v=YQhBEXC464w, fetched 2026-10-31
[S2] Analyzing the Internals of Neural Radiance Fields, https://doi.org/10.1109/cvprw63382.2024.00288, fetched 2026-10-26
[S3] Distilled Low Rank Neural Radiance Field with Quantization for Light Field Compression, https://core.ac.uk/works/391353499, fetched 2026-10-30 (unofficial)
