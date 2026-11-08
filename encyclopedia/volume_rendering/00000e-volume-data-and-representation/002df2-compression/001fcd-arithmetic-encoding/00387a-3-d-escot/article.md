# 3-D ESCOT

Also called: 3-D Embedded Set Tree Coding, 3-D Embedded Subband Coding with Optimal Truncation, Three-Dimensional Embedded Block Coding with Optimized Truncation

## Overview
3-D ESCOT is an entropy coding method for volumetric wavelet data that achieves scalable lossy-to-lossless compression by abandoning the zerotree structure in favor of 3-D context modeling for arithmetic coding [S1]. It differs from its sibling [3-D SPIHT](2501-mlzc-3-d-encoding-2-d-decoding.md) by applying high-order conditional arithmetic coding to sign and refinement bits, not just significance bits, and by partitioning bit planes into fractional passes ordered by rate-distortion significance [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3-D volume of wavelet coefficients $C$ organized into subbands, typically decomposed using an integer wavelet transform (e.g., 9/7 or (2,2) filters) [S1].
*   **Output:** A layered bitstream supporting rate and resolution scalability, which can be truncated for lossy decoding or fully decoded for lossless reconstruction [S1].

**Data Structures:**
*   **Significance Map:** A binary flag for each coefficient indicating if its magnitude exceeds the current threshold.
*   **Context Model:** A state machine for arithmetic coding. The context is formed by combining:
    1.  **Intraslice Context:** The 8-connected neighbors of the current coefficient within the same slice (same as EBCOT) [S1].
    2.  **Interslice Context:** A single fused scalar value $\hat{c}$ derived from the 18 neighbors in the front and back slices (see Formula) [S1].
*   **Coding States:** The joint context space is the Cartesian product of the intraslice context states and the quantized interslice context states. The number of interslice coding states is 16 [S1].

**Formulas:**
The interslice context $\hat{c}$ is computed using a least-squares linear estimator to reduce the dimension of the context space from 18 to 1, preventing context dilution [S1]:

$$
\hat{c} = \arg \min_{c} \sum_{i=1}^{18} (c_i - c)^2
$$

where $c_i$ are the magnitudes of the 18 neighbors in the front and back slices [S1].

**Parameters:**
*   **GOP Size:** Group of Pictures (slices) processed together. Typical values are 16, 32, or 64 slices. Larger GOPs offer slightly better compression but double memory usage [S1].
*   **Decomposition Levels:** Typically 2 levels. Increasing levels hurts 3-D ESCOT performance because it reduces the number of samples available for context adaptation [S1].
*   **Filters:** Filters with scaling by bit shifts (e.g., Table II filters in [S1]) perform better than lifting-based scaling for lossless compression [S1].

**Steps:**
1.  **Initialize:** Set the initial threshold $T$ based on the maximum coefficient magnitude. Initialize the arithmetic encoder.
2.  **Process Bit Planes:** Iterate from the most significant bit plane to the least significant.
3.  **Fractional Bit Plane Partitioning:** For each bit plane, process coefficients in natural raster scan order using three distinct passes, ordered by decreasing Rate-Distortion (R-D) significance [S1]:
    *   **Pass 1: Significance Propagation.**
        *   For each coefficient not yet significant:
            *   Determine the context from intraslice neighbors and the fused interslice estimator $\hat{c}$ [S1].
            *   Encode the significance bit using arithmetic coding with the estimated conditional probability [S1].
            *   If the coefficient becomes significant, encode its sign bit using high-order conditional arithmetic coding [S1].
    *   **Pass 2: Magnitude Refinement.**
        *   For each coefficient already significant:
            *   Encode the refinement bit (the next bit of the magnitude) using high-order conditional arithmetic coding [S1].
    *   **Pass 3: Normalization.**
        *   Process any remaining bits or coefficients not fully handled in the previous passes to complete the bit plane [S1].
4.  **Rate-Distortion Optimal Truncation:** To achieve rate scalability, invoke the equal slope criterion for R-D optimal bit allocation among different subbands [S1]. Multiplex the bitstreams from different fractional bit planes in all subbands into one final layered bitstream according to decreasing R-D significance [S1].
5.  **Class-Specific Optimization (Optional):** For medical data, use regression coefficients and quantizer parameters optimized for specific image classes (anatomy, modality) synchronized via file headers [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_3d_escot(coefficients, gop_size, num_levels):
    # Initialize arithmetic encoder
    ae = ArithmeticEncoder()
    
    # Determine initial threshold based on max coefficient magnitude
    max_mag = max(abs(c) for c in coefficients)
    threshold = 2 ** floor(log2(max_mag))
    
    # Process each bit plane from MSB to LSB
    while threshold > 0:
        # Partition bit plane into fractional passes
        # Pass 1: Significance Propagation
        for coeff in coefficients_in_raster_order:
            if not coeff.is_significant:
                # Build context: 8 intraslice neighbors + fused interslice estimator
                intraslice_ctx = get_8_connected_neighbors(coeff)
                interslice_neighbors = get_front_back_neighbors(coeff)
                c_hat = linear_estimator(interslice_neighbors) # Eq 4 in [S1]
                context = cartesian_product(intraslice_ctx, quantize(c_hat, states=16))
                
                # Encode significance bit
                prob = estimate_prob(context, 'significance')
                ae.encode_bit(coeff.is_significant, prob)
                
                if coeff.is_significant:
                    # Encode sign bit using high-order conditional arithmetic coding
                    sign_prob = estimate_prob(context, 'sign')
                    ae.encode_bit(coeff.sign, sign_prob)
        
        # Pass 2: Magnitude Refinement
        for coeff in coefficients_in_raster_order:
            if coeff.is_significant:
                # Encode refinement bit using high-order conditional arithmetic coding
                intraslice_ctx = get_8_connected_neighbors(coeff)
                interslice_neighbors = get_front_back_neighbors(coeff)
                c_hat = linear_estimator(interslice_neighbors)
                context = cartesian_product(intraslice_ctx, quantize(c_hat, states=16))
                
                ref_prob = estimate_prob(context, 'refinement')
                ae.encode_bit(coeff.get_next_magnitude_bit(), ref_prob)
                
        # Pass 3: Normalization (finish remaining bits)
        # Implementation detail: Handle any bits left over or coefficients 
        # that require additional normalization steps not covered in passes 1-2.
        # [G] Specific normalization logic for edge cases is not detailed in [S1],
        # but typically involves finalizing the magnitude representation.
        
        # Update threshold for next bit plane
        threshold /= 2
        
    # Apply equal slope criterion for R-D optimal truncation across subbands
    # and multiplex bitstreams into final layered bitstream
    final_bitstream = multiplex_by_rd_significance(ae.get_bits())
    return final_bitstream
```

## Operating characteristics
*   **Complexity:** The complexity of 3-D ESCOT is about 50% more than 3-D SPIHT and 3-D CB-EZW [S1].
*   **Performance:** On a Pentium 1-GHz PC, encoding the whole 192 slices of Skull takes 20 seconds, while decoding takes 13 seconds for 3-D SPIHT (3-D ESCOT decoding time not explicitly given, but implied to be comparable or slightly higher due to context modeling) [S1].
*   **Compression:** 3-D ESCOT achieves the best lossless and lossy compression among 3-D wavelet-based schemes, gaining about 10% over 3-D CB-EZW in lossless coding [S1]. In lossy coding at 0.1 b/p, 3-D ESCOT is 0.69 dB better than 3-D SPIHT; at 0.5 b/p, it is 0.93 dB better [S1]. Compared to 3-D CB-EZW, 3-D ESCOT results are 3.3–4.0 dB better at 0.1 and 0.5 b/p respectively [S1].
*   **Scalability:** Provides fine-grained rate and resolution scalability through fractional bit plane multiplexing [S1].
*   **Limitations:** Increasing wavelet decomposition levels hurts compression performance because it reduces the number of samples available for context adaptation in arithmetic coding [S1]. Simply duplicating 2-D EBCOT context to adjacent slices causes context dilution, requiring the linear estimator to reduce context dimension [S1].

## Background
3-D ESCOT was introduced by X. Wu, Z. Xiong, D. Y. Yun, and W. A. Pearlman in "3-D embedded subband coding with optimal truncation (3-D ESCOT)," published in *J. Appl. Comput. Harmon. Anal.*, vol. 10, pp. 290–315, May 2001 [S1]. It was originally designed for scalable wavelet video compression but was later adapted for lossy-to-lossless compression of medical volumetric data [S1]. It is used in medical applications to allow fast searching/browsing in lossy mode and diagnosis in lossless mode [S1].

## Sources

[S1] Lossy-to-lossless compression of medical volumetric data using three-dimensional integer wavelet transforms, https://doi.org/10.1109/tmi.2003.809585, fetched 2026-11-01
