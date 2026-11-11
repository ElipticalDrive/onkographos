# 3D Pyramid Compressor

Also called: 3D pyramid coder, 3D pyramid compressor project

## Overview
The 3D Pyramid Compressor is a lossy compression technique for confocal microscopic image sequences that treats a stack of 2D slices as a 3D volume, exploiting multi-dimensional redundancy using a 3D differential image pyramid structure combined with vector quantization (VQ) [S1, unofficial]. Unlike video coders that use motion compensation for planar motion, this method exploits static 3D spatial correlation in microscopy data where planar motion is not the generating physical process [S1, unofficial]. It builds on [3D Volume Data Compression](8638-3d-volume-data-compression.md) by generalizing 2D pyramid schemes to 3D, extracting vectors from the differential pyramid to reduce entropy [S1, unofficial].

## How it works
**Inputs:** A stack of 2D images (e.g., confocal microscopy slices) of size $W \times H$ with $D$ depth slices.
**Outputs:** Compressed bitstream containing VQ indices for the differential pyramid levels.

**Data Structures:**
1.  **Image Pyramid (IP):** A list of $N$ levels. Level $i$ has dimensions $W_i \times H_i \times D_i$. $W_0=W, H_0=H, D_0=D$. Each subsequent level is shrunk by scale factor $s=8$ in 3D (i.e., dimensions divided by 2 in each axis).
2.  **Differential Pyramid (DP):** A list of $N$ levels. $DP_i$ has the same dimensions as $IP_i$.
3.  **Vector Quantizer:** A codebook mapping $k$-dimensional vectors to indices. Typical vector shapes are $2 \times 2 \times 2$ or $4 \times 4 \times 2$ (width $\times$ height $\times$ depth) [S1, unofficial].

**Parameters:**
*   $N$: Number of pyramid levels.
*   $s$: Scale factor. $s=8$ for 3D (shrinking by factor 2 in x, y, z) [S1, unofficial].
*   $L$: Pyramid level index, $0$ (top/coarsest) to $N-1$ (bottom/finest).

**Steps:**

1.  **Read Input:** Read the stack of 2D images sequentially into a 3D array $V$ of size $W \times H \times D$ [S1, unofficial].
2.  **Construct Image Pyramid (IP):**
    *   Set $IP_{N-1} = V$.
    *   For $i = N-2$ down to $0$:
        *   $IP_i = Fshrink(IP_{i+1})$ [S1, unofficial].
        *   $Fshrink$ maps each voxel at level $i+1$ to 8 voxels at level $i$ (inverse of expansion). In practice, this is a downsampling operation, typically averaging or selecting a representative voxel from the $2 \times 2 \times 2$ block [S1, unofficial].
3.  **Compute Differential Pyramid (DP):**
    *   For the bottom level ($i = N-1$): $DP_{N-1} = IP_{N-1}$ [S1, unofficial].
    *   For upper levels ($i = N-2$ down to $0$):
        *   $DP_i = IP_i - Fexpand(IP_{i+1})$ [S1, unofficial].
        *   $Fexpand$ enlarges the image size by factor 8 in 3D (upsampling by factor 2 in each axis). Interpolation methods such as nearest neighbor, bilinear, or bi-cubic can be used [S1, unofficial].
4.  **Vector Quantization:**
    *   For each level $i$ from $N-1$ down to $0$:
        *   Partition $DP_i$ into non-overlapping vectors of shape $w \times h \times d$ (e.g., $2 \times 2 \times 2$) [S1, unofficial].
        *   For each vector, find the closest codeword in the VQ codebook.
        *   Store the index of the codeword.
        *   *Note:* The sources mention introducing VQ with quantization noise feedback to mitigate error magnification, but do not provide the specific feedback algorithm details [S1, unofficial].
5.  **Reconstruction (Decoder):**
    *   For the top level ($i=0$): $DP'_0 = VQ^{-1}(Indices_0)$ [S1, unofficial].
    *   For upper levels ($i=1$ to $N-1$):
        *   $DP'_i = VQ^{-1}(Indices_i)$ [S1, unofficial].
        *   $IP'_i = DP'_i + Fexpand(IP'_{i-1})$ [S1, unofficial].
    *   The final reconstructed volume is $IP'_{N-1}$ [S1, unofficial].

**Formula:**
The number of pixels (voxels) in the pyramid is given by:
$$ NumOfPixels_{pyramid} = \left( \frac{L-s}{s^L-s} \right) NumOfPixels_{original} $$
where $L$ is the pyramid level and $s$ is the scale factor ($s=8$ for 3D) [S1, unofficial].
*Correction/Clarification:* The source formula appears to have a typo or specific notation. The text states: $NumOfPixels_{pyramid} = (1 + \frac{1}{s} + \frac{1}{s^2} + \dots + \frac{1}{s^{L-1}}) NumOfPixels_{original}$. For $s=8$, the redundancy is $1/7$ rather than $1/3$ for the 2D case ($s=4$) [S1, unofficial].

**Pitfalls:**
*   Quantization errors in upper pyramid levels are magnified during reconstruction because $Fexpand$ enlarges the image size. An error at the top corrupts 8 pixels in 3D per expansion step (or 16 in 2D) [S1, unofficial].
*   The histogram of the DP is highly peaked around zero, allowing advanced entropy coding to take advantage [S1, unofficial].

## Operating characteristics
*   **Redundancy:** The 3D pyramid structure is redundant. For scale factor $s=8$, the redundancy is $1/7$ of the original data size [S1, unofficial].
*   **Performance:** The method is designed for confocal microscopic image sequences where 3D spatial correlation is high [S1, unofficial].
*   **Strengths:** Exploits static 3D spatial correlation better than video coders that rely on planar motion compensation, which is not the generating physical process in microscopy [S1, unofficial].
*   **Limitations:** Quantization errors propagate and magnify from top to bottom levels, potentially degrading quality at high compression ratios [S1, unofficial].
*   **When to use:** Confocal microscopy data with high 3D correlation.
*   **When not to use:** Video or data with significant planar motion [S1, unofficial].

## Background
The 3D Pyramid Compressor was developed for confocal laser scanning microscopy (CLSM) data, which generates large (>100Mb) 3D volumetric image data [S1, unofficial]. The basic concept was presented in a project funded by Scottish Enterprise [S1, unofficial]. It extends Burt and Adelson's 1983 pyramid coder, which introduced quantization into the image pyramid structure for multi-resolution lossy compression [S1, unofficial]. The 3D version generalizes the $Fshrink$ and $Fexpand$ operations to 3D voxel data, where each voxel maps to 8 voxels, reducing redundancy compared to 2D [S1, unofficial].

## Sources
[S1] Confocal microscopic image sequence compression using vector quantization and 3D pyramids, http://eprints.gla.ac.uk/3795/1/confocal_image_sequence.pdf, fetched 2026-10-30 (unofficial)
