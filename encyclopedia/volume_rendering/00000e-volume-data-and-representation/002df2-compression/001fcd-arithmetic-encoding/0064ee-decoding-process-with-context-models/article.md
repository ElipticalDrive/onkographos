# Decoding Process with Context Models

Also called: FCGS Decoding, GMM-based Decoding

## Overview
This method describes the decoding phase of a neural compression pipeline for 3D Gaussian Splatting attributes, specifically reconstructing quantized feature values $\hat{y}$ using a Gaussian Mixture Model (GMM) estimated from contextual information. It solves the problem of efficiently recovering continuous-valued data from a binary bitstream by adapting probability distributions to local spatial and channel dependencies. It is distinguished from sibling methods like [MLZC 3-D Encoding/2-D Decoding](2501-mlzc-3-d-encoding-2-d-decoding.md) by its specific use of inter-Gaussian, intra-Gaussian, and hyperprior contexts to parameterize a GMM for arithmetic decoding, rather than using fixed or simpler statistical models.

## How it works
**Inputs and Outputs:**
*   **Input:** A binary bitstream containing encoded data; previously decoded Gaussian coordinates $\mu_g$, masks $m$, and hyperpriors $\hat{z}$.
*   **Output:** Reconstructed feature values $\hat{y}_{i, [nc-c, nc)}$ for the current Gaussian $i$ and channel slice.

**Data Structures:**
*   $\mu_g$: 16-bit quantized Gaussian coordinates.
*   $m$: Binary masks.
*   $\hat{z}$: Hyperprior features.
*   $\hat{y}$: Reconstructed feature values.
*   GMM Parameters: Mean $\mu$, standard deviation $\sigma$, and mixture weight $\pi$ for each component.

**Parameters:**
*   $N_s$: Number of source Gaussians for context. Typical value: 4 [S1, unofficial].
*   $c$: Channel slice width.
*   $\epsilon_m$: Mask probability noise parameter. Typical value: 0.01 [S1, unofficial].
*   Grid resolutions: Vary by attribute (e.g., 70-90 for 3D geometry, 300-500 for 2D color) [S1, unofficial].

**Steps:**
1.  **Decode Primitives:** Decode Gaussian coordinates $\mu_g$, masks $m$, and hyperpriors $\hat{z}$ from the bitstream. $\mu_g$ is decoded losslessly (e.g., via GPCC), and $m$ is decoded using Arithmetic Encoding (AE) based on the frequency of 1s [S1, unofficial].
2.  **Context Extraction:** For the target Gaussian $i$ and channel range $[nc-c, nc)$:
    *   **Inter-Gaussian Context:** Extract features $\hat{y}_{i, [0, N_s-1]}$ from previously decoded Gaussians. Apply channel-sliced inter-Gaussian context models to obtain parameters $\mu_{s,i,nc}, \sigma_{s,i,nc}, \pi_{s,i,nc}$ [S1, unofficial].
    *   **Intra-Gaussian Context:** Extract features $\hat{y}_{i, [0, nc-c)}$ from the current Gaussian's previously decoded channels. Apply intra-Gaussian context models to obtain parameters $\mu_{c,i,nc}, \sigma_{c,i,nc}, \pi_{c,i,nc}$ [S1, unofficial].
    *   **Hyperprior Context:** Extract hyperprior features $\hat{z}_i$. Apply channel-sliced hyperprior models to obtain parameters $\mu_{h,i,nc}, \sigma_{h,i,nc}, \pi_{h,i,nc}$ [S1, unofficial].
3.  **GMM Construction:** Combine the three sets of parameters to form the GMM probability distribution for the current value. The exact combination formula is not specified in the sources, but the GMM is defined by the union of these context-derived means, variances, and weights [S1, unofficial].
4.  **Arithmetic Decoding:** Perform Arithmetic Decoding (AD) using the constructed GMM to recover the value $\hat{y}_{i, [nc-c, nc)}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decode_gaussian_attribute(bitstream, mu_g, m, z_hat, i, nc, c, N_s):
    # Step 1: Primitives are assumed already decoded from bitstream
    # mu_g, m, z_hat are available
    
    # Step 2: Extract Contexts
    # Inter-Gaussian Context: Use previously decoded Gaussians
    # Assume get_inter_gaussian_context returns parameters based on y_hat from other Gaussians
    mu_s, sigma_s, pi_s = get_inter_gaussian_context(y_hat_prev_gaussians, i, nc, N_s)
    
    # Intra-Gaussian Context: Use previously decoded channels of current Gaussian
    # Assume get_intra_gaussian_context returns parameters based on y_hat from current Gaussian's earlier channels
    mu_c, sigma_c, pi_c = get_intra_gaussian_context(y_hat_current_partial, i, nc, c)
    
    # Hyperprior Context: Use hyperprior z_hat
    # Assume get_hyperprior_context returns parameters based on z_hat
    mu_h, sigma_h, pi_h = get_hyperprior_context(z_hat, i, nc)
    
    # Step 3: Construct GMM
    # The sources state the distribution is calculated using GMM based on these parameters.
    # The exact mathematical combination (e.g., product of densities, weighted sum) is not detailed.
    # We assume a standard GMM density evaluation where the total probability is a function
    # of the three component sets.
    gmm_params = combine_gmm_params(
        mu_s, sigma_s, pi_s,
        mu_c, sigma_c, pi_c,
        mu_h, sigma_h, pi_h
    )
    
    # Step 4: Arithmetic Decoding
    # Decode the value using the GMM probabilities
    y_value = arithmetic_decode(bitstream, gmm_params)
    
    return y_value
```

**Edge Cases and Pitfalls:**
*   **Precision:** $\mu_g$ is 16-bit quantized; ensure consistent quantization between encoder and decoder [S1, unofficial].
*   **Consistency:** A fixed random seed must be maintained during inference to guarantee consistency between encoding and decoding [S1, unofficial].
*   **Boundary Conditions:** The context models rely on previously decoded data. Ensure proper handling of the first channel or first Gaussian where context may be unavailable [G].

## Subtopics

### Context Modeler for Wavelet Compression
A subalgorithm for 3D wavelet-based compression of hyperspectral image data (ICER-3D). It provides probability-of-zero estimates for bits before encoding. It classifies a bit of a DWT coefficient into one of 19 contexts based on previously encoded bits. It uses a one-dimensional spectral context model relying on two neighbors in the spectral direction but no neighbors in the same spatial plane. DWT coefficients are assigned to one of four categories (0-3) before context assignment. Separate probability-of-zero statistics are gathered for each context class [S2, unofficial].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Implemented using PyTorch on NVIDIA L40s GPU [S1, unofficial].
*   **Strengths:** Adapts probability distributions to local spatial and channel dependencies, improving compression efficiency over static models.
*   **Limitations:** Requires decoding of prior Gaussians and channels before the current value can be decoded, imposing a sequential dependency.

## Background
The FCGS decoding method originates from the "Fast Feedforward 3D Gaussian Splatting Compression" paper (2024) [S1, unofficial]. The concept of context modeling for wavelet compression is rooted in the ICER-3D hyperspectral image compression software (2010) [S2, unofficial].

## Sources
[S1] Fast Feedforward 3D Gaussian Splatting Compression, https://doi.org/10.48550/arxiv.2410.08017, fetched 2026-10-26 (unofficial)
[S2] NASA Technical Reports Server (NTRS) 20100005261: ICER-3D Hyperspectral Image Compression Software, https://archive.org/details/NASA_NTRS_Archive_20100005261, fetched 2026-10-26 (unofficial)
