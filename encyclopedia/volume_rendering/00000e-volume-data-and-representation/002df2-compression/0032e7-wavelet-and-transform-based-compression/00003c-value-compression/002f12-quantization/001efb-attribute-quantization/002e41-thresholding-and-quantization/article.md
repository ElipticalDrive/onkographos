# Thresholding and Quantization

Also called: Coefficient Thresholding and Quantization

## Overview
Thresholding and Quantization is a lossy compression technique applied to the coefficients of a compact volume model (such as wavelets or tensors) to further reduce data size by neglecting insignificant values [S1][S2]. It distinguishes itself from sibling methods like [Quantization with Noise Injection](2090-quantization-with-noise-injection.md) by focusing on coefficient sparsity and precision reduction rather than adding stochastic noise to preserve statistical properties. It allows the user to define the maximum allowed error, making it a controllable trade-off between compression ratio and fidelity [S1][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of coefficients $\{c_i\}$ from a compact model (e.g., wavelet coefficients, PCA components, or tensor factors).
*   **Output:** A reduced set of coefficients $\{c'_i\}$ and metadata defining the reconstruction error or compression ratio.

**Data Structures:**
The input is typically a dense array of floating-point numbers. The output is a sparse structure (e.g., list of non-zero coefficients with indices) or a quantized integer array, depending on the specific technique applied.

**Steps:**
1.  **Thresholding (Zeroing):** Identify coefficients with magnitude below a threshold $\tau$. Set these coefficients to zero. This is particularly effective for wavelet transforms where energy is concentrated in few large coefficients [S1][S2].
    $$ c'_i = \begin{cases} 0 & \text{if } |c_i| < \tau \\ c_i & \text{otherwise} \end{cases} $$
2.  **Truncation (Sorting):** For models like PCA, sort coefficients by magnitude (or importance) and truncate the least significant ones. Keep only the top $k$ coefficients [S1][S2].
3.  **Quantization (Integer Representation):** Map floating-point coefficients to a limited set of integer values. This can be done via uniform quantization or by using a pre-learned dictionary (Vector Quantization) where each coefficient is replaced by the index of the best-matching codeword [S1][S2].
    $$ c'_i = \text{round}\left(\frac{c_i}{\Delta}\right) \cdot \Delta $$
    Where $\Delta$ is the quantization step size.
4.  **Error Control:** The user defines the allowed error $\epsilon$. The threshold $\tau$ or step size $\Delta$ is adjusted to ensure the reconstruction error (e.g., MSE) remains below $\epsilon$ [S1][S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def threshold_and_quantize(coefficients, method='threshold', threshold=0.0, max_error=None):
    """
    Apply thresholding or quantization to volume model coefficients.
    
    Args:
        coefficients: List or array of float coefficients.
        method: 'threshold' or 'quantize'.
        threshold: Magnitude threshold for zeroing.
        max_error: Maximum allowed reconstruction error (unused in simple thresholding, used for adaptive step).
        
    Returns:
        Reduced coefficients array.
    """
    if method == 'threshold':
        # Step 1: Thresholding
        reduced = [0.0 if abs(c) < threshold else c for c in coefficients]
        return reduced
    
    elif method == 'quantize':
        # Step 3: Quantization to integers
        # Determine step size based on max_error if provided, else use fixed delta
        delta = 0.1 # Default step size, [G] typical value for normalized data
        if max_error is not None:
            # [G] Approximate relation: error ~ delta/2. Set delta = 2 * max_error
            delta = 2.0 * max_error
        
        quantized = [round(c / delta) * delta for c in coefficients]
        return quantized
    
    else:
        raise ValueError("Unknown method")
```

**Parameters:**
*   **Threshold ($\tau$):** Controls sparsity. Typical values depend on the coefficient distribution; often derived from the maximum coefficient magnitude (e.g., $10^{-3} \times \max|c_i|$) [S1][S2].
*   **Allowed Error ($\epsilon$):** The user-defined bound on reconstruction error [S1][S2].
*   **Quantization Step ($\Delta$):** Determines the number of representable levels. Not given in the sources as a fixed value; depends on data range.

**Edge Cases:**
*   **Precision Loss:** Aggressive quantization can introduce visible artifacts, especially in smooth gradients.
*   **Boundary Effects:** Thresholding near zero may discard meaningful low-frequency details if the threshold is too high.

## Subtopics

### Wavelet Thresholding
Uses the sparsity of wavelet coefficients. Best for data with localized features. Link: [Wavelet Compression](link-to-wavelet-article).

### Vector Quantization
Uses a learned dictionary. Best for complex, non-linear structures. Link: [Vector Quantization](link-to-vq-article).

### PCA Truncation
Sorts and keeps top components. Best for correlated data with dominant modes. Link: [PCA Compression](link-to-pca-article).

## Operating characteristics
*   **Time Complexity:** $O(N)$ for thresholding and simple quantization, where $N$ is the number of coefficients. Sorting for truncation is $O(N \log N)$ [S1][S2].
*   **Memory Complexity:** Reduces memory by storing only non-zero coefficients (sparse) or smaller integer types. The reduction ratio depends on the threshold/error settings.
*   **Performance:** Achieves high compression ratios (e.g., ~0.1% to ~13% of original size) with acceptable visual quality for volume rendering [S1][S2].
*   **Strengths:** Simple to implement, user-controlled error, effective for sparse data.
*   **Limitations:** Lossy; may introduce artifacts if parameters are not tuned correctly. Not suitable for lossless requirements.
*   **When to Use:** When memory bandwidth is a bottleneck and some visual error is acceptable.
*   **When Not to Use:** When exact data preservation is required (e.g., medical diagnosis where subtle features are critical).

## Background
Originating from general signal processing and data compression techniques, this method was adapted for compressed GPU-based direct volume rendering. It is discussed in the context of compact models like wavelets and tensors [S1][S2]. The approach leverages the fact that many volume datasets have compact representations where most coefficients are small or redundant.

## Sources
[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
