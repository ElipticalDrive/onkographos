# Thresholding and Quantization
Also called: Coefficient Thresholding and Quantization

## Overview
Thresholding and Quantization is a lossy compression technique that reduces the data size of a compact volume model by neglecting or approximating its coefficients. It solves the problem of excessive memory footprint in hierarchical or transform-based representations by allowing the user to define an acceptable error bound [S1, S2]. It is distinguished from its parent, Attribute Quantization, by focusing on the reduction of model coefficients (such as wavelet or tensor coefficients) rather than just attribute precision, and from its sibling, Quantization with Noise Injection, by relying on deterministic truncation or rounding rather than stochastic noise addition [S1, S2].

## How it works
**Inputs and Outputs:**
- **Input:** A compact model represented by a set of coefficients (e.g., wavelet coefficients, PCA coefficients, or tensor decomposition coefficients) [S1, S2].
- **Output:** A reduced set of coefficients or a quantized representation that approximates the original model within a user-defined error limit [S1, S2].

**Data Structures:**
The method operates on the coefficient arrays of the compact model. No specific new data structure is defined; the coefficients are stored in their native format (e.g., floating-point arrays) before processing [S1, S2].

**Steps:**
1. **Thresholding:** Identify coefficients that are close to zero or below a significance threshold. These coefficients are neglected (set to zero) [S1, S2]. This is particularly effective for wavelet transforms where energy is concentrated in few coefficients [S2].
2. **Truncation:** Alternatively, sort the coefficients by magnitude and truncate the least significant ones. This approach is typical for PCA-like models [S1, S2].
3. **Quantization:** Represent the remaining floating-point coefficients with integer values to reduce precision and storage size [S1, S2].
4. **Dictionary-Based Quantization (Optional):** Use a learned dictionary with a limited number of codewords. The coefficients are approximated by the closest dictionary entry, explicitly including a quantization step in the model design [S1, S2].

**Formulas:**
Not given in the sources. The sources describe the conceptual steps (thresholding, truncation, quantization) but do not provide specific mathematical formulas for the threshold calculation or quantization mapping [S1, S2].

**Parameters:**
- **Allowed Error:** The user defines the maximum error that can be introduced by the compression [S1, S2]. This parameter controls the trade-off between compression ratio and fidelity.

**Edge Cases and Pitfalls:**
- **Precision Loss:** Aggressive thresholding or low-bit quantization can lead to visible artifacts if the allowed error is set too low [S1, S2].
- **Degenerate Input:** If the model coefficients are already sparse or low-precision, further thresholding may yield minimal gains [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def threshold_and_quantize(coefficients, allowed_error, method='threshold'):
    """
    Compress coefficients using thresholding or truncation and quantization.
    
    Args:
        coefficients: List or array of floating-point coefficients.
        allowed_error: Maximum allowed error for the compression.
        method: 'threshold' for zeroing small values, 'truncate' for keeping top-k.
        
    Returns:
        Compressed coefficients (sparse or quantized).
    """
    if method == 'threshold':
        # Threshold coefficients close to zero
        # Note: Exact threshold value derivation from allowed_error is not specified in sources [S1, S2]
        threshold = compute_threshold_from_error(coefficients, allowed_error) [G]
        compressed = [0.0 if abs(c) < threshold else c for c in coefficients]
    elif method == 'truncate':
        # Sort and truncate least significant coefficients
        # Note: Exact number of coefficients to keep based on allowed_error is not specified [S1, S2]
        sorted_indices = sorted(range(len(coefficients)), key=lambda i: abs(coefficients[i]), reverse=True)
        num_to_keep = compute_num_to_keep_from_error(coefficients, allowed_error) [G]
        compressed = [0.0] * len(coefficients)
        for i in range(num_to_keep):
            compressed[sorted_indices[i]] = coefficients[sorted_indices[i]]
    else:
        raise ValueError("Unknown method")
        
    # Quantize floating point coefficients to integers
    # Note: Specific quantization scheme (e.g., uniform, scalar) is not detailed [S1, S2]
    quantized = quantize_to_integers(compressed) [G]
    
    return quantized

def compute_threshold_from_error(coefficients, allowed_error):
    # Placeholder for threshold calculation logic not provided in sources [S1, S2]
    # This would typically involve analyzing the coefficient distribution
    return 0.0 [G]

def compute_num_to_keep_from_error(coefficients, allowed_error):
    # Placeholder for truncation count logic not provided in sources [S1, S2]
    return len(coefficients) [G]

def quantize_to_integers(coefficients):
    # Placeholder for quantization logic not provided in sources [S1, S2]
    # This would map floats to a limited set of integer values
    return [int(round(c)) for c in coefficients] [G]
```

## Subtopics

### [Sensitivity Threshold Clustering](11422-sensitivity-threshold-clustering.md)
A variant that applies a data-driven sensitivity metric to determine which attributes warrant high-fidelity storage versus compressed representation, excluding sensitive vectors from clustering.

## Operating characteristics
- **Time Complexity:** Not given in the sources. Sorting and thresholding are generally $O(N \log N)$ or $O(N)$ depending on the implementation [S1, S2].
- **Memory Complexity:** Reduces memory footprint by storing fewer non-zero coefficients or lower-precision values [S1, S2].
- **Measured Performance:** Not given in the sources. The sources mention compression ratios achieved in specific examples (e.g., ~13%, ~2%, ~0.4%, ~0.1% for dental enamel data) but do not provide general performance metrics [S1].
- **Strengths:** Simple to implement; allows user-defined error bounds; effective for sparse representations like wavelets [S1, S2].
- **Limitations:** Lossy; performance depends on the sparsity of the coefficients; requires careful tuning of the allowed error [S1, S2].
- **When to use:** When memory is constrained and some loss of fidelity is acceptable; particularly effective for hierarchical models with sparse coefficients [S1, S2].
- **When not to use:** When lossless reconstruction is required; when the model coefficients are not sparse or do not benefit from thresholding [S1, S2].

## Background
The technique originates from the field of compressed GPU-based direct volume rendering, where compact models (such as wavelet transforms or tensor decompositions) are used to represent volume data [S1, S2]. It is a standard post-processing step for reducing the size of these models before further encoding (e.g., entropy coding) [S1, S2]. The approach is inspired by techniques used in image compression (e.g., JPEG 2000) and linear algebra (e.g., PCA) [S1, S2].

## Sources
[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
