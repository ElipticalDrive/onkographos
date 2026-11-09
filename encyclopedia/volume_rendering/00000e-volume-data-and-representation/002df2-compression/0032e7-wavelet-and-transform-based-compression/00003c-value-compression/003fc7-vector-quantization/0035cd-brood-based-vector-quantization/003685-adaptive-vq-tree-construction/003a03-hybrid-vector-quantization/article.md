# Hybrid Vector Quantization

Also called: Hybrid method, Scheme 3

## Overview
Hybrid Vector Quantization is a multiresolution compression scheme that applies Vector Quantization (VQ) to each resolution level separately to maintain multiresolution capability, while exploiting spatial redundancy by creating vectors from pixels in the three detail levels surrounding every pixel in the lowest resolution image [S1, unofficial]. It sets itself apart from sibling methods like [Texture Atlas Compression with Sparse Regions](7775-texture-atlas-compression-with-sparse-regions.md) by explicitly linking vector construction across multiple wavelet detail levels (DL1, DL2, DL3) to a single low-resolution pixel, rather than treating levels or regions in isolation [S1, unofficial]. This approach maintains a fairly constant Signal-to-Noise Ratio (SNR) while keeping a high compression ratio when higher detail levels are added, unlike Scheme 1 which sees a sharp drop in compression ratio [S1, unofficial].

## How it works
**Inputs:** A multiresolution decomposition of the data (typically using Daubechies-6 wavelet basis) consisting of a Low Resolution Image (LRI) and three Detail Levels (DL1, DL2, DL3) [S1, unofficial].
**Outputs:** A compressed bitstream containing codebook indices for vectors constructed from the LRI and surrounding detail pixels, allowing reconstruction at varying resolutions [S1, unofficial].

**Data Structures:**
The method constructs vectors based on the resolution level. The vector size decreases as the resolution level decreases, reflecting the significance of the wavelet coefficients [S1, unofficial].
*   **Level 3 (Highest Detail):** Vectors are formed from pixels in DL3 surrounding an LRI pixel. Size: $3 \times (4 \times 4)$ pixels [S1, unofficial].
*   **Level 2 (Mid Detail):** Vectors are formed from pixels in DL2 surrounding an LRI pixel. Size: $3 \times (2 \times 2)$ pixels [S1, unofficial].
*   **Level 1 (Lowest Detail):** Vectors are formed from pixels in DL1 surrounding an LRI pixel. Size: $3 \times (1 \times 1)$ pixels [S1, unofficial].

**Steps:**
1.  **Multiresolution Decomposition:** Decompose the input volume/image using a wavelet transform (e.g., Daubechies-6) to obtain the LRI and detail levels DL1, DL2, and DL3 [S1, unofficial].
2.  **Vector Extraction:** For each pixel position in the LRI, extract a vector composed of the corresponding pixel from the LRI and surrounding pixels from the detail levels.
    *   For the highest detail level (DL3), extract a $4 \times 4$ block from DL3, a $4 \times 4$ block from DL2, and a $4 \times 4$ block from DL1? No, the source specifies the vector dimensions directly as $3 \times (4 \times 4)$ for N1, $3 \times (2 \times 2)$ for N2, and $3 \times (1 \times 1)$ for N3. The "3" likely refers to the three detail levels or the spatial arrangement. The text states: "creating vectors out of pixels that come from the three details at each resolution level from the surroundings of every pixel in the lowest resolution image" [S1, unofficial].
    *   Specifically, the vector for the highest level (N1) consists of pixels from DL3, DL2, and DL1 surrounding the LRI pixel, with spatial extent $4 \times 4$ [S1, unofficial].
    *   The vector for the mid level (N2) consists of pixels from the three details with spatial extent $2 \times 2$ [S1, unofficial].
    *   The vector for the lowest detail level (N3) consists of pixels from the three details with spatial extent $1 \times 1$ [S1, unofficial].
3.  **Codebook Generation:** Train a codebook using VQ on the extracted vectors. The source notes that the construction procedure needs to retrieve the full vector across scales [S1, unofficial].
4.  **Encoding:** Replace each extracted vector with its nearest neighbor index in the codebook.
5.  **Storage:** Store the codebook and the sequence of indices. The multiresolution capability is preserved because each level's vectors are quantized separately [S1, unofficial].

**Parameters:**
*   **Wavelet Basis:** Daubechies-6 is used in the cited implementation [S1, unofficial].
*   **Codebook Size:** The source cites performance for $N_c = 512$ centroids [S1, unofficial].
*   **Vector Dimensions:** $3 \times 16$ (Level 3), $3 \times 4$ (Level 2), $3 \times 1$ (Level 1) [S1, unofficial].

**Edge Cases:**
*   **Boundary Conditions:** The source does not specify padding or boundary handling for the vector extraction around image/volume edges [S1, unofficial].
*   **Precision:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_vq_compress(data, wavelet_basis='db6', codebook_size=512):
    # Step 1: Multiresolution Decomposition
    lri, dl1, dl2, dl3 = wavelet_decompose(data, wavelet_basis)
    
    # Step 2: Vector Extraction
    # Extract vectors for the highest detail level (N1)
    # Vector size: 3 channels (DL1, DL2, DL3) x 4x4 spatial
    vectors_n1 = extract_vectors(dl1, dl2, dl3, lri, window_size=4)
    
    # Extract vectors for the mid detail level (N2)
    # Vector size: 3 channels x 2x2 spatial
    vectors_n2 = extract_vectors(dl1, dl2, dl3, lri, window_size=2)
    
    # Extract vectors for the lowest detail level (N3)
    # Vector size: 3 channels x 1x1 spatial
    vectors_n3 = extract_vectors(dl1, dl2, dl3, lri, window_size=1)
    
    # Step 3: Codebook Generation (VQ)
    # Combine vectors from all levels or train separately?
    # The text says "apply VQ to each resolution level separately"
    codebook_n1 = train_vq(vectors_n1, codebook_size)
    codebook_n2 = train_vq(vectors_n2, codebook_size)
    codebook_n3 = train_vq(vectors_n3, codebook_size)
    
    # Step 4: Encoding
    indices_n1 = quantize(vectors_n1, codebook_n1)
    indices_n2 = quantize(vectors_n2, codebook_n2)
    indices_n3 = quantize(vectors_n3, codebook_n3)
    
    return {
        'codebooks': [codebook_n1, codebook_n2, codebook_n3],
        'indices': [indices_n1, indices_n2, indices_n3],
        'lri': lri # LRI might be stored separately or as part of the lowest level
    }

def extract_vectors(dl1, dl2, dl3, lri, window_size):
    vectors = []
    # Iterate over LRI pixels
    for y in range(lri.height):
        for x in range(lri.width):
            # Extract window from each detail level centered at (x,y)
            # Note: Exact alignment and boundary handling not specified in source [S1, unofficial]
            w1 = get_window(dl1, x, y, window_size)
            w2 = get_window(dl2, x, y, window_size)
            w3 = get_window(dl3, x, y, window_size)
            # Concatenate into a single vector
            vec = concatenate(w1, w2, w3)
            vectors.append(vec)
    return vectors
```

## Subtopics

### [3D Volume Data Compression](8638-3d-volume-data-compression.md)
Generalization of the 2D hybrid method to 3D volume data, extracting vectors as shown in Figure 4. It extends the spatial redundancy exploitation to three dimensions.

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
Using $N_c = 512$ centroids and Daubechies-6 wavelets on the test data:
*   **LRI only:** Compression Ratio 126.7:1, SNR 16.55, MSE 8059.3 [S1, unofficial].
*   **+ DL3:** Compression Ratio 68.5:1, SNR 19.34, MSE 4240.5 [S1, unofficial].
*   **+ DL2:** Compression Ratio 48.0:1, SNR 22.29, MSE 2148.8 [S1, unofficial].
*   **+ DL1:** Compression Ratio 42.0:1, SNR 24.35, MSE 1338.9 [S1, unofficial].

The hybrid method maintains a fairly constant SNR while keeping a high compression ratio when higher detail levels are added [S1, unofficial]. In contrast, Scheme 1 sees a sharp drop in compression ratio when higher detail levels are added [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Maintains multiresolution capability; exploits spatial redundancy across scales; suitable for volume data applications because it can be adapted to 3D, unlike JPEG which lacks a straightforward 3D counterpart for DCT quantization tables [S1, unofficial].
*   **Limitations:** Performance relative to JPEG is not conclusive; JPEG outperforms Scheme 1 at low compression ratios, and the hybrid scheme's performance relative to JPEG is unclear ("no clear conclusions can be drawn") [S1, unofficial].

**When to use:**
Use when multiresolution access is required and the data is volumetric or can be adapted to 3D. It is preferred over Scheme 1 when maintaining compression ratio while adding detail is critical [S1, unofficial].

**When not to use:**
If the application is strictly 2D and JPEG compatibility is required, JPEG may be preferred [S1, unofficial].

## Background
The Hybrid Vector Quantization method (Scheme 3) was introduced in the context of multiresolution compression schemes for medical image and volume data [S1, unofficial]. It builds on the idea of applying VQ to each resolution level separately while exploiting spatial redundancy across detail levels [S1, unofficial]. The method is part of a family of wavelet-based compression schemes that use Daubechies-6 basis functions [S1, unofficial]. It was developed to address the limitations of previous schemes (like Scheme 1) which suffered from significant compression ratio drops when higher detail levels were included [S1, unofficial]. The method is suitable for volume data applications due to its adaptability to 3D, a feature JPEG lacks [S1, unofficial].

## Sources
[S1] DTIC ADA411461: Multiresolution Compression Schemes for Medical Image and Volume Data, https://archive.org/details/DTIC_ADA411461, fetched 2026-10-26 (unofficial)
