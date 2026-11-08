# Hybrid Vector Quantization

Also called: Scheme 3, Hybrid method

## Overview
Hybrid Vector Quantization is a multiresolution compression scheme that applies Vector Quantization (VQ) to each resolution level separately to maintain multiresolution capability, while exploiting spatial redundancy by creating vectors from pixels in the three detail levels surrounding every pixel in the lowest resolution image [S1, unofficial]. It sets itself apart from independent VQ schemes by maintaining a fairly constant Signal-to-Noise Ratio (SNR) while keeping a high compression ratio when higher detail levels are added, whereas independent schemes see a sharp drop in compression ratio [S1, unofficial]. It is particularly suitable for volume data applications because it can be adapted to 3D, unlike JPEG which lacks a straightforward 3D counterpart for DCT quantization tables [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A multiresolution decomposition of the data (typically using Daubechies-6 wavelet basis) consisting of a Low Resolution Image (LRI) and detail levels DL1, DL2, DL3 [S1, unofficial].
*   **Output:** Compressed bitstream containing VQ indices for the constructed vectors [S1, unofficial].

**Data Structures:**
*   **Vector Layout:** For a pixel in the LRI, the vector is constructed from pixels in the three detail levels (DL1, DL2, DL3) surrounding that pixel [S1, unofficial].
*   **Vector Dimensions:** The dimensionality depends on the resolution level. Lower resolution levels use low-dimensional vectors, and higher levels use larger vectors to reflect coefficient significance [S1, unofficial]. Specifically, the source illustrates a structure where $N_1$ corresponds to $3 \times (4 \times 4)$, $N_2$ to $3 \times (2 \times 2)$, and $N_3$ to $3 \times (1 \times 1)$, though the exact mapping of these indices to specific spatial positions is defined by the spatial surrounding of the LRI pixel [S1, unofficial].

**Steps:**
1.  **Multiresolution Decomposition:** Decompose the input data using a wavelet transform (e.g., Daubechies-6) to obtain the LRI and detail levels DL1, DL2, DL3 [S1, unofficial].
2.  **Vector Construction:** For each pixel in the LRI, construct a hybrid vector by grouping pixels from the three detail levels (DL1, DL2, DL3) surrounding that pixel [S1, unofficial].
3.  **Codebook Generation:** Generate a codebook using a VQ training algorithm (e.g., LBG) on the constructed vectors [S1, unofficial].
4.  **Quantization:** Replace each constructed vector with its nearest neighbor index in the codebook [S1, unofficial].
5.  **Encoding:** Encode the indices using entropy coding [S1, unofficial].

**Formulas:**
*   Compression Ratio ($r$): $r = \frac{\text{Original Size}}{\text{Compressed Size}}$ [S1, unofficial].
*   SNR: Calculated as per standard image quality metrics [S1, unofficial].

**Parameters:**
*   **Wavelet Basis:** Daubechies-6 [S1, unofficial].
*   **Codebook Size ($N_c$):** 512 centroids used in 3D volume data experiments [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The vector extraction relies on the "surroundings" of every pixel in the LRI. Boundary pixels in the LRI may have fewer surrounding detail pixels, requiring padding or truncated vectors [G].
*   **Precision:** The method relies on the significance of wavelet coefficients; improper scaling of detail levels may affect vector distance calculations [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_vq_compress(data, wavelet_basis='db6', codebook_size=512):
    # Step 1: Multiresolution Decomposition
    lri, dl1, dl2, dl3 = wavelet_decompose(data, wavelet_basis)
    
    # Step 2: Vector Construction
    vectors = []
    for y in range(lri.height):
        for x in range(lri.width):
            # Extract surrounding pixels from DL1, DL2, DL3
            # Note: Exact spatial layout of 'surrounding' pixels 
            # is defined by the specific wavelet filter bank and downsampling
            # This is a placeholder for the specific extraction logic
            vec = extract_hybrid_vector(lri, dl1, dl2, dl3, x, y)
            vectors.append(vec)
            
    # Step 3: Codebook Generation (e.g., LBG algorithm)
    codebook = lbq_train(vectors, codebook_size)
    
    # Step 4: Quantization
    indices = []
    for vec in vectors:
        idx = find_nearest_neighbor(vec, codebook)
        indices.append(idx)
        
    # Step 5: Encoding
    compressed_data = entropy_encode(indices)
    return compressed_data, codebook

def extract_hybrid_vector(lri, dl1, dl2, dl3, x, y):
    # Placeholder for specific spatial extraction logic
    # The source indicates vectors are formed from pixels in DL1, DL2, DL3
    # surrounding the pixel (x,y) in LRI
    # Dimensions vary: N1=3x(4x4), N2=3x(2x2), N3=3x(1x1)
    # This function must return a flattened vector of these pixels
    pass
```

## Subtopics

### 3D Volume Data Compression
**Definition:** Generalization of 2D compression schemes 1 (independent) and 3 (hybrid) to 3D volume data, extracting vectors as shown in Figure 4 [S1, unofficial].
**Steps:** Apply Scheme 1 (independent vectors) or Scheme 3 (hybrid) to 3D volume data [S1, unofficial].
**Comparison:** In 3D, Scheme 1 has higher quality at large compression ratios but drops significantly in $r$ when DL2/DL3 are included. Scheme 3 increases quality with lower decreases in compression capabilities [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:** Not given in the sources.
**Measured Performance:**
*   **2D Images (Table IIIa):**
    *   LRI only: 126.7:1 ratio, 16.55 SNR, 8059.3 MSE.
    *   + DL3: 68.5:1 ratio, 19.34 SNR, 4240.5 MSE.
    *   + DL2: 48.0:1 ratio, 22.29 SNR, 2148.8 MSE.
    *   + DL1: 42.0:1 ratio, 24.35 SNR, 1338.9 MSE.
*   **3D Volume Data (Table IV, $N_c=512$):**
    *   Scheme 1 (LRI): 1017.7:1 ratio, 16.91 SNR.
    *   Scheme 1 (+ DL1): 17.7:1 ratio, 24.14 SNR.
    *   Scheme 3 (LRI): 1017.6:1 ratio, 16.97 SNR.
    *   Scheme 3 (+ DL1): 356.9:1 ratio, 21.0 SNR.
*   **Scaling:** The hybrid scheme maintains a fairly constant SNR while keeping a high compression ratio when higher detail levels are added, unlike Scheme 1 which sees a sharp drop in compression ratio [S1, unofficial].
**Strengths and Limitations:**
*   **Strengths:** Maintains high compression ratio with increasing detail levels; suitable for 3D volume data [S1, unofficial].
*   **Limitations:** For low compression ratios, JPEG may outperform Scheme 1; clear conclusions for hybrid vs JPEG are not drawn [S1, unofficial].
**When to use:** When multiresolution capability is required and the data is 3D volume data [S1, unofficial].
**When not to use:** When working with 2D images and JPEG compatibility is prioritized [S1, unofficial].

## Background
**Origin:** The method was introduced in a paper supported by research grants VA78/99 and 1FD97-0881 [S1, unofficial].
**History:** The paper focuses on compression schemes in the wavelet domain, specifically addressing the vector selection procedure for multiresolution data compression [S1, unofficial].
**Usage:** Suitable for volume data applications, including medical imaging (e.g., human hip isosurface visualization) [S1, unofficial].

## Sources
[S1] DTIC ADA411461: Multiresolution Compression Schemes for Medical Image and Volume Data, https://archive.org/details/DTIC_ADA411461, fetched 2026-10-26 (unofficial)
