# Texture Map Coding (TMC)

Also called: Image-based Compression, SLF Coefficient Compression via Texture Map Coding

## Overview
Texture Map Coding (TMC) is a lossy compression method for Surface Light Field (SLF) coefficients that maps 3D surface points to 2D texture coordinates, allowing the use of standard image codecs [S1]. It treats each coefficient channel as an independent image, leveraging spatial redundancy in the 2D plane to achieve higher Rate-Distortion (RD) performance than hierarchical methods like RAHT, at the cost of higher computational complexity [S1].

## How it works
**Inputs:**
*   A set of 3D surface points $P$.
*   For each point $p \in P$, an $N$-dimensional vector of SLF coefficients $\alpha(p) = [\alpha_0(p), \dots, \alpha_{N-1}(p)]$ [S1].
*   A 3D-to-2D mapping function that assigns a 2D texture coordinate $u(p)$ to each point $p$ [S1].

**Outputs:**
*   A compressed bitstream containing:
    1.  The 3D-to-2D mapping parameters [S1].
    2.  Compressed images for each of the $N$ coefficient channels [S1].

**Data Structures:**
*   **Mapping Parameters:** Not specified in the sources. Must encode the geometric unwrapping or projection logic used to generate $u(p)$ [S1].
*   **Coefficient Images:** $N$ separate 2D images. Each image corresponds to one coefficient index $i \in [0, N-1]$. The pixel values are integers in the range $[0, 255]$ (8-bit representation) [S1].

**Steps:**
1.  **Mapping:** Map each 3D point $p$ to a 2D texture coordinate $u(p)$. This creates a 2D grid where neighboring positions in the 2D plane correspond to neighboring positions in 3D space [S1].
2.  **Channel Separation:** Decompose the $N$-dimensional coefficient vectors $\alpha(p)$ into $N$ separate 2D images, one for each coefficient index $i$ [S1]. Because B-Spline wavelet basis functions are orthogonal, each coefficient plane can be compressed independently [S1].
3.  **Rescaling and Quantization:** Uniformly rescale the coefficients for each channel to constrain values to the range $[0, 255]$. Round the rescaled values to integers [S1].
4.  **Image Compression:** Compress each resulting 2D image independently using an image codec. The reference implementation uses High Efficiency Video Coding (HEVC) [S1].
5.  **Encoding Mapping:** Encode the 3D-to-2D mapping parameters so the decoder can reconstruct the texture maps from the compressed images [S1].

**Reconstruction:**
On the decoder side, coefficients are recovered via entropy decoding, inverse quantization, and inverse image decoding. The recovered SLF coefficients $\hat{\alpha}$ are used to reconstruct the view maps as [S1]:

$$
\hat{f}(\omega|p) = \sum_{i=0}^{N-1} \hat{\alpha}_i(p) \cdot g_i(\omega)
$$

where $\omega = (\theta, \phi)$ indicates the direction from $p$ to the viewpoint, and $g_i(\omega)$ are the basis functions [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tmc_compress(points, coefficients_3d, mapping_function, codec):
    """
    Compress SLF coefficients using Texture Map Coding.
    
    Args:
        points: List of 3D points p.
        coefficients_3d: List of N-dimensional vectors alpha(p) for each point.
        mapping_function: Function mapping p -> (u, v) 2D coordinates.
        codec: Image compression function (e.g., HEVC encoder).
        
    Returns:
        compressed_mapping: Encoded 3D-to-2D mapping parameters.
        compressed_images: List of compressed bitstreams for each coefficient channel.
    """
    # Step 1 & 2: Map points to 2D and separate channels
    # Initialize N empty 2D grids (images)
    N = len(coefficients_3d[0])
    images = [create_empty_2d_grid() for _ in range(N)]
    
    for p in points:
        u, v = mapping_function(p)
        alpha = coefficients_3d[p]
        
        # Assign coefficients to respective channel images
        for i in range(N):
            images[i][u][v] = alpha[i]
            
    # Step 3: Rescale and Quantize
    for i in range(N):
        min_val = min(images[i].flatten())
        max_val = max(images[i].flatten())
        
        # Uniform rescaling to [0, 255]
        if max_val != min_val:
            scale = 255.0 / (max_val - min_val)
            offset = -min_val * scale
        else:
            scale = 0.0
            offset = 0.0
            
        for u in range(width):
            for v in range(height):
                val = images[i][u][v]
                # Rescale and round to integer
                images[i][u][v] = int(val * scale + offset)
                
    # Step 4: Compress each image
    compressed_images = []
    for i in range(N):
        # Compress the 2D integer grid using the codec
        bitstream = codec.encode(images[i])
        compressed_images.append(bitstream)
        
    # Step 5: Encode mapping
    compressed_mapping = encode_mapping_parameters(mapping_function)
    
    return compressed_mapping, compressed_images

def tmc_reconstruct(compressed_mapping, compressed_images, codec, basis_functions):
    """
    Reconstruct SLF from TMC compressed data.
    
    Args:
        compressed_mapping: Encoded mapping parameters.
        compressed_images: List of compressed bitstreams.
        codec: Image decompression function.
        basis_functions: List of basis functions g_i(omega).
        
    Returns:
        reconstruction_function: Function returning f_hat(omega|p)
    """
    # Decode mapping
    mapping_function = decode_mapping_parameters(compressed_mapping)
    
    # Decode images and inverse quantize
    N = len(compressed_images)
    recovered_coefficients = {} # Map p -> alpha_hat(p)
    
    for i in range(N):
        # Decode image
        image = codec.decode(compressed_images[i])
        
        # Inverse quantize (assuming same scale/offset logic or stored parameters)
        # Note: Sources do not specify if scale/offset are stored; 
        # assuming codec or metadata handles this or it is fixed [G]
        for u, v in image.coordinates():
            # Map back to 3D point p
            p = mapping_function.inverse(u, v)
            val = image[u][v]
            
            # Reconstruct alpha_i(p)
            # Note: Exact inverse scaling formula depends on encoder side storage [G]
            alpha_hat_i = val / scale - offset 
            
            if p not in recovered_coefficients:
                recovered_coefficients[p] = [0] * N
            recovered_coefficients[p][i] = alpha_hat_i
            
    def reconstruct_view(p, omega):
        if p not in recovered_coefficients:
            return 0
        alpha_hat = recovered_coefficients[p]
        # Sum over N coefficients
        result = 0
        for i in range(N):
            result += alpha_hat[i] * basis_functions[i](omega)
        return result
        
    return reconstruct_view
```

**Parameters:**
*   **Bit Depth:** 8-bit representation ($[0, 255]$) is used in the reference implementation [S1].
*   **Codec:** HEVC is used in the reference implementation (MPEG PCC Test Model Category 2, version 0.0 TMC2v0) [S1].
*   **Mapping:** Adaptive texture mapping is applied in the reference implementation [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Quantization to 8-bit integers introduces lossy errors [S1].
*   **Mapping Loss:** The 3D-to-2D mapping must be accurately encoded; otherwise, the decoder cannot reconstruct the texture maps [S1].
*   **Sparsity:** The distribution of points on a discretized surface can be sparse compared to the whole space, which may affect the efficiency of standard image codecs if not handled by the mapping strategy [S1].

## Operating characteristics
*   **Performance:** TMC achieves higher Rate-Distortion (RD) performance than RAHT [S1].
*   **Complexity:** TMC is less computationally efficient than RAHT [S1].
*   **Scalability:** Not given in the sources.
*   **Strengths:** High RD performance; leverages mature, efficient image codecs (e.g., HEVC) [S1].
*   **Limitations:** Higher computational cost; requires a valid 3D-to-2D mapping that preserves spatial locality [S1].
*   **When to use:** When RD performance is prioritized over computational efficiency [S1].
*   **When not to use:** When computational resources are limited or when RAHT's scalability is required [S1].

## Background
*   **Origin:** TMC is described in the context of Surface Light Field (SLF) compression using point cloud codecs [S1].
*   **History:** It is part of the MPEG Point Cloud Compression (PCC) standards, specifically associated with Test Model Category 2 (TMC2) [S1].
*   **Usage:** Used for compressing SLF coefficients, where 3D surface points are mapped to 2D images for compression [S1].

## Sources
[S1] Surface Light Field Compression Using a Point Cloud Codec, https://doi.org/10.1109/jetcas.2018.2883479, fetched 2026-10-28
