# Compression Domain Volume Rendering Pipeline

Also called: Compression domain rendering pipeline, Compression domain volume visualization, Progressive compression domain volume visualization, Wavelet compression domain rendering, Wavelet compression pipeline, Wavelet-based Compression Domain Volume Rendering

## Overview
A pipeline for networked volume rendering that compresses data in the wavelet domain, allowing progressive transmission, one-pass decompression, and direct rendering without full decompression [S1, unofficial]. It converts wavelet-transformed volume data into a sequential bitstream for distributed rendering, optimizing for both compression ratio and rendering speed via spatial clustering and significance-based sorting [S1, unofficial]. Unlike traditional methods, this approach avoids full decompression prior to splatting, reducing client-side memory requirements [S1, unofficial].

## How it works
The pipeline consists of a server-side compression stage and a client-side decompression/rendering stage [S1, unofficial]. The server prepares a bitstream that the client decodes and renders directly into an accumulation buffer [S1, unofficial].

### Inputs and Outputs
*   **Input:** 3D volume data (intensity or RGB) [S1, unofficial].
*   **Output:** A binary bitstream for transmission/storage, and a rendered image on the client [S1, unofficial].

### Data Structures
**Bitstream Structure:**
1.  **Huffman Tables:** Three tables (for tags and deltas), size ~2KB [S1, unofficial].
2.  **Metadata:**
    *   Colorspace basis vectors.
    *   Maximum wavelet depth $M$.
    *   Initial wavelet coefficients $w_0$.
    *   Delta factors $\Delta(\mathbf{C}, m, \text{type})$ for each color channel, decomposition level, and wavelet type [S1, unofficial].
3.  **Tag Stream:** Runlength-coded wavelet type, decomposition depth $m$, and color channel, encoded as variable-length Huffman tags [S1, unofficial].
4.  **Data Stream:** Huffman-encoded deltas $\delta$, sign flags, and spatial positions [S1, unofficial].

**Client Storage:**
*   8 mother-wavelet splats (footprints) [S1, unofficial].
*   3 Huffman look-up tables (precomputed 10-bit tables for efficiency) [S1, unofficial].
*   Total size < 5KB [S1, unofficial].

### Parameters
*   **Maximum Decomposition Level ($M$):** Typically $M=2$ for MRI volumes, $M=3$ for other datasets [S1, unofficial].
*   **Quantization Bits ($P$):** Typically 8 bits (matching standard framebuffers), but can be reduced to 3 bits without significant visual loss [S1, unofficial].
*   **Importance Factors ($I$):**
    *   RGB: $[1, 1, 1]$ [S1, unofficial].
    *   YIQ: $I(Y)=4, I(I)=1.5, I(Q)=0.6$ [S1, unofficial].
    *   PCA: Absolute values of eigenvalues [S1, unofficial].
*   **Spatial Clustering:**
    *   Position threshold (radius): Example value 3 [S1, unofficial].
    *   Coefficient threshold (score difference): Example values 2 or 4 [S1, unofficial].

### Steps

#### Server-Side Compression

1.  **Preprocess Data:**
    *   If the input is RGB, transform it into an optimized colorspace (YIQ or PCA) to decorrelate data and allow color-sensitive quantization [S1, unofficial].
    *   If intensity, skip colorspace transform [S1, unofficial].

2.  **Wavelet Transform:**
    *   Perform wavelet transform independently on each color channel up to maximum depth $M$ [S1, unofficial].

3.  **Compute Significance Scores:**
    *   For each coefficient $w$ in color channel $\mathbf{C}$, calculate the significance score $S$:
        $$ S(w, \mathbf{C}) = E(w(\mathbf{C})) \cdot I(\mathbf{C}) $$
        where $E$ is the wavelet energy and $I$ is the colorspace importance factor [S1, unofficial].

4.  **Sort Coefficients:**
    *   Sort all coefficients from all channels by their significance score $S$ in descending order [S1, unofficial].

5.  **Apply Spatial Clustering:**
    *   Iterate through the sorted list.
    *   Let $w_{max}$ be the coefficient with the highest remaining score.
    *   Check neighbors of $w_{max}$ within a predefined spatial radius.
    *   If a neighbor's score exceeds a threshold (e.g., $S_{max} - \text{threshold}$), select it as the next element in the bitstream sequence.
    *   Otherwise, select $w_{max}$ regardless of position.
    *   Label selected coefficients to avoid multiple storage [S1, unofficial].

6.  **Delta-Coding, Normalization, and Quantization:**
    *   For each color channel $\mathbf{C}$, wavelet type, and decomposition level $m$:
        *   Compute the delta factor $\Delta$:
            $$ \Delta(\mathbf{C}, m, \text{type}) = \frac{\text{Range}}{2^P} $$
            where Range is the coefficient range and $P$ is the number of quantization bits [S1, unofficial].
        *   Transmit $\Delta$ once per type/level/channel [S1, unofficial].
        *   For each coefficient $n$ in the sorted sequence:
            *   Compute the normalized quantized difference $\delta$:
                $$ \delta(\mathbf{C}, m, \text{type}, n) = \text{round}\left(\frac{|w_n(\mathbf{C}, m, \text{type}) - \tilde{w}_{n-1}(\mathbf{C}, m, \text{type})|}{\Delta(\mathbf{C}, m, \text{type})}\right) $$
                where $\tilde{w}_{n-1}$ is the previously decoded coefficient (or $w_0$ for the first) [S1, unofficial].
            *   Encode the sign of the coefficient as a single-bit flag `sign(n)` [S1, unofficial].

7.  **Encode:**
    *   Encode spatial position, wavelet type, and color channel losslessly for each coefficient [S1, unofficial].
    *   Perform runlength coding on wavelet type, decomposition depth $m$, and color channel sequences [S1, unofficial].
    *   Encode tags using Huffman coding [S1, unofficial].
    *   Encode deltas $\delta$ using Huffman coding [S1, unofficial].
    *   Assemble the bitstream: [Huffman Tables] [Metadata] [Tag Stream] [Data Stream] [S1, unofficial].

#### Client-Side Decompression and Rendering

1.  **Receive and Parse:**
    *   Receive the bitstream.
    *   Load Huffman tables and metadata (basis vectors, $M$, $w_0$, $\Delta$) [S1, unofficial].

2.  **Decode Coefficients:**
    *   Decode the tag stream to determine wavelet type, depth, and channel for each coefficient [S1, unofficial].
    *   Decode the data stream to retrieve $\delta$, `sign(n)`, and spatial position [S1, unofficial].
    *   Reconstruct the coefficient $\tilde{w}_n$ using the delta factor:
        $$ \tilde{w}_{n+1}(\mathbf{C}, m, \text{type}) = \text{sign}(n+1) \left( |\tilde{w}_n(\mathbf{C}, m, \text{type})| - \delta(\mathbf{C}, m, \text{type}, n) \cdot \Delta(\mathbf{C}, m, \text{type}) \right) $$
        [S1, unofficial].
    *   Store decoded coefficients in a local cache [S1, unofficial].

3.  **Render:**
    *   For each decoded coefficient, compute the corresponding wavelet footprint (splat) at its spatial position [S1, unofficial].
    *   Weight the footprint by the coefficient value $\tilde{w}$ [S1, unofficial].
    *   Splat the weighted footprint directly into an accumulation buffer (software or hardware) [S1, unofficial].
    *   Use alpha-blending or hardware accumulation operations [S1, unofficial].
    *   Spatial clustering allows FOV culling: splats outside the field of view are detected and skipped [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision:** Quantization to $P$ bits introduces error. Reducing $P$ to 3 bits is possible but reduces quality [S1, unofficial]. The relative quantization error metric $Q_E$ counts the relative energy difference of all wavelet coefficients and their quantized counterparts: $Q_E = \left( \frac{\sum_{C \in \{C_1, C_2, C_3\}} \sum_w r(w(C))}{\sum_{C \in \{C_1, C_2, C_3\}} \sum_w E(w(C))} \cdot 100 \right) [\%]$, where $r(w(\mathbf{C})) = E(w(\mathbf{C}) - \tilde{w}(\mathbf{C}))$ [S1, unofficial].
*   **Boundaries:** Wavelet transform boundaries must be handled consistently between server and client [G].
*   **Degenerate Input:** If a channel has zero range, $\Delta$ calculation must handle division by zero or zero range appropriately [G].

## Operating characteristics
*   **Time Complexity:** Decompression rate ~30 k wavelet-coefficients/sec on an Indy R4400 workstation [S1, unofficial].
*   **Memory Complexity:** Client storage < 5KB (8 splats + 3 Huffman tables) [S1, unofficial].
*   **Performance:** Interactive framerates achieved on SGI-Indy (R4400/150MHz) and SGI Maximum Impact (R10000/195MHz) for 160x180 pixel resolution [S1, unofficial].
*   **Compression Ratio:**
    *   MRI (256x128x256, M=2, Haar): 0.825% (upper row) to 3.26% (lower row) of original size [S1, unofficial].
    *   Visible Human RGB (128x128x128, M=3, Haar): >95% compression gain with SNR > 60 dB [S1, unofficial].
*   **Strengths:** Low client memory, progressive transmission, direct rendering without full decompression, hardware-friendly [S1, unofficial].
*   **Limitations:** Haar wavelets cause blocky artifacts; higher-order splines increase computation cost [S1, unofficial].

## Background
Developed for distributed environments where clients have limited computational power and memory [S1, unofficial].

## Sources
[S1] Compression Domain Volume Rendering for Distributed Environments, https://cgl.ethz.ch/Downloads/Publications/Tutorials/1998/tu_Gro98-4.pdf, fetched 2026-10-26 (unofficial)
