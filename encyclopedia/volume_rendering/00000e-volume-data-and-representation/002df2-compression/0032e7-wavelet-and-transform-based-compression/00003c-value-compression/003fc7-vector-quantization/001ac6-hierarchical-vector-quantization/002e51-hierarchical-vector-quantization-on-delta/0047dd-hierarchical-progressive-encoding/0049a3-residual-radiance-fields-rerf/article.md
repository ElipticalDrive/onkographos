# Residual Radiance Fields — ReRF
Also called: ReRF, Residual Radiance Fields

## Overview
ReRF is a technique for compactly modeling long-duration dynamic scenes to enable streamable free-viewpoint videos [S1]. It solves the storage and streaming challenges of dynamic neural radiance fields by representing the spatial-temporal feature space using motion and residual grids in a sequential manner [S1]. Unlike sibling methods that suffer from blurring or deformation errors over long sequences, ReRF maintains high photorealism and sharpness regardless of frame count by using a keyframe-based residual compression strategy [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of dynamic scene captures (multiview video) and a target quantization factor for storage adjustment [S1].
*   **Output:** A compressed bitstream containing motion fields, residual grids, and PCA coefficients, enabling real-time decoding and rendering [S1].

**Data Structures:**
The method organizes feature grids into Groups Of Feature grids (GOF) [S1].
*   **GOF:** A sequence of $N$ frames. The implementation uses $N=20$ [S1].
*   **I-feature grid (Keyframe):** The first grid in a GOF, coded independently [S1].
*   **P-feature grid (Residual):** Subsequent grids in a GOF, coded as residuals relative to the previous grid [S1].
*   **Feature Matrix:** Feature grids are reshaped into matrices $f_t(m, n)$ and $r_t(m, n)$, where $m$ is the number of non-empty feature voxels and $n$ is the number of feature channels [S1].
*   **PCA Components:** For residual compression, Singular Value Decomposition (SVD) is approximated via PCA, yielding components $U, S, V$ such that $r_t \approx U \cdot \text{diag}(S) \cdot V^T$ [S1].

**Steps:**
1.  **Segmentation:** Divide the feature grid sequence into GOFs of size 20 [S1].
2.  **I-feature Grid Encoding:**
    *   For the first frame $f_1$ in a GOF, encode the full feature grid $f_t$ [S1].
    *   This grid contains the vital information for the subsequent P-feature grids in the group [S1].
3.  **P-feature Grid Encoding (Residual Compression):**
    *   For subsequent frames $t > 1$, compute the residual grid $r_t$ relative to the previous feature grid [S1].
    *   Reshape $r_t$ into a matrix $r_t(m, n)$ [S1].
    *   Perform PCA on $r_t(m, n)$ to obtain $U, S, V$ [S1].
    *   Encode the PCA components ($U, S, V$) and any necessary masks for empty space [S1].
4.  **Motion Field Encoding:** Encode the voxel motion field separately [S1].
5.  **Quantization:** Scale the quantization factor to flexibly adjust storage size [S1].

**Formulas:**
The residual grid $r_t$ is compressed using PCA approximation of SVD [S1]:
$$ r_t(m,n) \approx U \cdot \text{diag}(S) \cdot V^T $$
Where $U, S, V$ are the PCA components of the residual matrix [S1].

**Parameters:**
*   **GOF Size:** 20 frames [S1].
*   **Quantization Factor:** Adjustable to trade off between storage and quality [S1].

**Edge Cases and Pitfalls:**
*   **Large Motion:** While ReRF handles large motions better than deformation-based methods like DeVRF, extreme motion may still impact residual compactness [S1].
*   **Training Time:** Training is slow (~10 mins per frame) compared to storage efficiency [S1].
*   **Rendering Speed:** Current rendering speed (20 fps) may not be fluent for all interactive applications [S1].
*   **Capture Requirements:** Requires expensive and hard-to-construct multiview capture systems [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_rerf(feature_grids, gof_size=20, quantization_factor=1.0):
    """
    Encodes a sequence of feature grids into ReRF format.
    feature_grids: List of feature grids [f1, f2, ..., fT]
    """
    bitstream = []
    num_frames = len(feature_grids)
    
    for i in range(0, num_frames, gof_size):
        gof_frames = feature_grids[i : i + gof_size]
        
        # First frame is I-feature grid (Keyframe)
        f_key = gof_frames[0]
        # Encode full feature grid for keyframe
        bitstream.append(encode_feature_grid(f_key, quantization_factor))
        
        # Subsequent frames are P-feature grids (Residuals)
        for j in range(1, len(gof_frames)):
            f_prev = gof_frames[j-1]
            f_curr = gof_frames[j]
            
            # Compute residual grid
            r_t = compute_residual(f_curr, f_prev)
            
            # Reshape residual to matrix (m voxels, n channels)
            # Note: Implementation detail on how m and n are derived from grid structure
            # [G] Assuming a function to flatten non-empty voxels and channels
            r_matrix = reshape_to_matrix(r_t)
            
            # Perform PCA on residual matrix
            U, S, V = perform_pca(r_matrix)
            
            # Encode PCA components
            bitstream.append(encode_pca_components(U, S, V, quantization_factor))
            
        # Encode motion field for the GOF (if applicable)
        # [G] Specific motion field encoding logic not detailed in source beyond "encode motion field"
        bitstream.append(encode_motion_field(gof_frames, quantization_factor))
        
    return bitstream

def decode_rerf(bitstream, gof_size=20):
    """
    Decodes ReRF bitstream back to feature grids.
    """
    feature_grids = []
    current_gof_start = 0
    
    while current_gof_start < len(bitstream):
        # Decode I-feature grid
        f_key = decode_feature_grid(bitstream[current_gof_start])
        feature_grids.append(f_key)
        current_gof_start += 1
        
        # Decode P-feature grids
        for j in range(1, gof_size):
            if current_gof_start >= len(bitstream):
                break
                
            # Decode PCA components
            U, S, V = decode_pca_components(bitstream[current_gof_start])
            
            # Reconstruct residual matrix
            r_matrix = reconstruct_pca(U, S, V)
            
            # Reshape back to feature grid
            r_t = reshape_from_matrix(r_matrix)
            
            # Reconstruct current feature grid from previous and residual
            f_prev = feature_grids[-1]
            f_curr = reconstruct_feature_grid(f_prev, r_t)
            feature_grids.append(f_curr)
            
            current_gof_start += 1
            
        # Decode motion field (if needed for rendering)
        # [G] Motion field decoding logic
        current_gof_start += 1 # Skip motion field data in bitstream
        
    return feature_grids
```

## Subtopics

### Feature-level Residual Compression
A compression strategy for neural radiance fields that divides feature grid sequences into Groups Of Feature grids (GOF) to enable streaming and fast seeking [S1]. It uses I-feature grids (keyframes) coded independently and P-feature grids (residuals) coded relative to the previous grid [S1]. The GOF size determines the interval between keyframes, with a default implementation choosing a keyframe every 20 frames [S1]. This approach allows for variable bitrates by scaling the quantization factor [S1].

## Operating characteristics
*   **Storage:** Total average model size is 793KB for the tested sequences, which is three orders of magnitude smaller than the original uncompressed size (755.31KB origin vs 785.6MB uncompressed in one comparison context, though the text states "three orders smaller" and lists 793KB total) [S1]. Storage breakdown: Origin Size 755.31 KB, Residual 31.80 KB, Motion 0.68 KB, PCA others 4.86 KB [S1].
*   **Training Time:** Approximately 10 minutes per frame [S1].
*   **Rendering Speed:** 20 fps [S1].
*   **Decoding/Rendering Latency:** Average time to decode one frame is 47.03ms; average time to render one frame is 44.62ms [S1]. Total processing time with parallel decoding and rendering is about 50ms [S1].
*   **Scalability:** Storage can be flexibly adjusted by scaling the quantization factor [S1].
*   **Strengths:** Maintains high photorealism and sharpness even as the number of frames increases (tested up to 4000 frames) [S1]. Outperforms dynamic methods like DeVRF, INGP-T, and TiNeuVox in long sequences with large motions [S1].
*   **Limitations:** Training time is slow compared to storage efficiency [S1]. Rendering speed (20 fps) may not be sufficient for highly fluent interaction [S1]. Requires expensive multiview capture systems [S1].

## Background
ReRF was introduced in the paper "Neural Residual Radiance Fields for Streamably Free-Viewpoint Videos" [S1]. It is designed for long-duration dynamic scenes and enables online dynamic radiance fields streaming and free-viewing [S1]. The method builds on the concept of hierarchical progressive encoding by using residual grids and PCA for compression [S1].

## Sources
[S1] Neural Residual Radiance Fields for Streamably Free-Viewpoint Videos, https://doi.org/10.1109/cvpr52729.2023.00016, fetched 2026-10-26
