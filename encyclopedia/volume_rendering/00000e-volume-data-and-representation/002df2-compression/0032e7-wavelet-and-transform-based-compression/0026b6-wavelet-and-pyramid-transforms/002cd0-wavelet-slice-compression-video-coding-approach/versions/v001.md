# Wavelet Slice Compression (Video Coding Approach)
Also called: Temporal Wavelet Slice Compression, Video Coding Approach for Wavelet Compression

## Overview
Wavelet Slice Compression is a volumetric data compression method that partitions a 3D volume into 2D slices and treats them as a sequence of correlated frames, applying video coding techniques to exploit temporal and spatial coherence [S1]. Unlike methods that apply a full 3D transform across the entire volume, this approach separates spatial compression within slices from temporal compression between them, leveraging established video compression principles to achieve higher compression ratios for volumetric data with strong inter-slice correlation.

## How it works
**Inputs:** A 3D volumetric dataset $V$ of dimensions $W \times H \times D$, where $W$ and $H$ are width and height, and $D$ is the depth (number of slices).
**Outputs:** A compressed bitstream representing the volume.

**Data Structures:**
1.  **Slice Buffer:** A 2D array $S_d$ of size $W \times H$ for each slice $d \in [0, D-1]$.
2.  **Coefficient Block:** A 2D array of wavelet coefficients $C_d$ for each slice.
3.  **Motion Vectors (Optional):** If motion estimation is used, a list of vectors $\vec{m}_d$ for each slice relative to a reference slice.

**Steps:**
1.  **Partitioning:** The volume $V$ is partitioned into $D$ independent 2D slices $S_0, S_1, \dots, S_{D-1}$.
2.  **Spatial Transform:** Each slice $S_d$ is transformed using a 2D wavelet transform (e.g., Haar, Daubechies, or biorthogonal 9/7) to produce coefficient blocks $C_d$. This step is identical to standard 2D wavelet compression [S1].
3.  **Temporal Prediction:** To exploit temporal coherence, the method predicts the current slice's coefficients $C_d$ based on previously decoded slices.
    *   *Simple Approach:* Use the previous slice $C_{d-1}$ as a predictor. The residual $R_d = C_d - C_{d-1}$ is computed.
    *   *Advanced Approach (Video Coding):* Apply motion estimation between slices to find a displacement vector $\vec{m}_d$ such that a shifted version of a reference slice $C_{ref}$ best matches $C_d$. The residual is $R_d = C_d - C_{ref}(\vec{x} + \vec{m}_d)$.
4.  **Quantization:** The residual coefficients $R_d$ (or the original coefficients if no prediction is used) are quantized using a quantization step size $Q$. The quantized value is $Q(R_d) = \text{round}(R_d / Q)$.
5.  **Entropy Encoding:** The quantized residuals and motion vectors (if any) are entropy encoded (e.g., Huffman or Arithmetic coding) to produce the final bitstream.
6.  **Decoding:** The bitstream is entropy decoded to recover quantized residuals. Inverse quantization yields approximate residuals. If motion vectors are present, they are used to reconstruct the predicted slice. The inverse 2D wavelet transform is applied to each slice to recover the spatial data.

**Parameters:**
*   **Wavelet Filter:** Commonly Haar or biorthogonal 9/7 [S1].
*   **Quantization Step ($Q$):** Controls lossiness. Not given in the sources.
*   **Reference Frame:** Typically the previous slice ($d-1$) or a keyframe. Not given in the sources.

**Edge Cases:**
*   **Boundary Conditions:** Wavelet transforms require handling boundaries. Padding or symmetric extension is typically used [G].
*   **Degenerate Input:** If slices are uncorrelated, temporal prediction yields no benefit, and overhead may reduce compression efficiency.

## Subtopics
## Operating characteristics
**Time Complexity:** Spatial transform is $O(W \cdot H \cdot D)$. Temporal prediction is $O(W \cdot H \cdot D)$ for simple difference or $O(W \cdot H \cdot D \cdot M)$ for motion estimation with search range $M$. Entropy encoding is $O(N \log N)$ where $N$ is the number of coefficients.
**Memory Complexity:** $O(W \cdot H \cdot D)$ for input/output. Intermediate buffers for coefficients and residuals are also $O(W \cdot H \cdot D)$.
**Strengths:** High compression ratios for volumes with strong inter-slice correlation (e.g., medical imaging, time-varying data). Leverages mature video coding infrastructure.
**Limitations:** Less effective for volumes with weak inter-slice correlation. Motion estimation adds computational cost. Not as naturally suited for random access as full 3D transforms [S1].
**When to use:** Volumetric data with high temporal/spatial coherence between slices.
**When not to use:** Volumes with low inter-slice correlation or where random access to arbitrary slices is critical without decoding preceding slices.

## Background
This approach was proposed by Rodler [Rod99] as a way to exploit the similarities between volumetric data and video sequences, applying video coding principles to 3D wavelet compression [S1]. It builds on the shared principles of [Wavelet and Pyramid Transforms](3856-wavelet-and-pyramid-transforms.md) by using wavelet transforms for spatial decomposition but diverges in its handling of the third dimension.

## Sources
[S1] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
