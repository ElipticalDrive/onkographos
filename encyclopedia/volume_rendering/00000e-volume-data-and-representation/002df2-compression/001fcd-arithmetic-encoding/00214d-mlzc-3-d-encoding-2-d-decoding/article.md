# MLZC 3-D Encoding/2-D Decoding

Also called: 3-D/2-D MLZC, 3-D/2-D Multidimensional Layered Zero Coding, LZC, Layered Zero Coding, MLZC, Multi-Layered Zero Coding

## Overview
MLZC is a wavelet-based compression method that performs a fully 3-D Discrete Wavelet Transform (3-D-DWT) on volumetric data but encodes every resulting 2-D subband image independently to enable fast, localized 2-D decoding without reconstructing the entire volume [S1]. It differs from pure 3-D algorithms by allowing random access to single 2-D slices, and from pure 2-D algorithms by exploiting inter-slice correlation for higher compression efficiency [S1]. It builds on [Arithmetic Encoding](2127-arithmetic-encoding.md) using context-adaptive models derived from spatial and interband dependencies [S1].

## How it works
**Inputs:** Volumetric data $V$ (integer samples), target image index $z$ (optional, for decoding optimization), decomposition depth $L$, filter type (default 5-3 CDF) [S1].
**Outputs:** Embedded bitstream containing quantization layers for all subbands, with markers for 2-D random access [S1].

**Data Structures:**
1. **Subband Tree:** Result of 3-D-DWT. Coefficients $c(\mathbf{k}, l, j)$ where $\mathbf{k}=(x,y,z)$, $l$ is decomposition level, $j$ is subband direction [S1].
2. **Significance Map:** Binary array $\sigma_i[\mathbf{k}]$ for each quantization layer $i$ [S1].
3. **Context Model:** Weighted sum $\chi^{s,(idx)}[\mathbf{k}]$ of neighbor significance states [S1].
4. **Bitstream:** Segments of quantization layers. Markers separate segments in LPL/LPL-PROG modes [S1].

**Steps:**
1. **3-D-DWT via Lifting Scheme:** Apply a fully 3-D separable DWT using the lifting scheme to enable in-place, lossless integer-to-integer transformation [S1]. Define polyphase matrix $P(z)$ for synthesis filters $h, g$:
   $$P(z) = \begin{bmatrix} h_e(z) & g_e(z) \\ h_o(z) & g_o(z) \end{bmatrix}$$
   where even/odd components are:
   $$h_e(z^2) = \frac{h(z) + h(-z)}{2}, \quad h_o(z^2) = \frac{h(z) - h(-z)}{2z^{-1}}$$
   (Same for $g$) [S1]. Factor $P(z)$ into lifting steps (triangular matrices) [S1]. Apply filtering successively in $x, y, z$ directions [S1].
2. **Quantization Layer Generation:** Generate a sequence of quantization step sizes $Q_i$ for $i=0 \dots N-1$:
   $$Q_0 = 2^{\lfloor \log_2 (\max_{\mathbf{k},l,j} |c(\mathbf{k}, l, j)|) \rfloor}$$
   $$Q_i = Q_{i-1} / 2 \quad \text{for } i > 0$$
   [S1]. For each layer $i$, compute significance symbols:
   $$\sigma_i[\mathbf{k}] = \begin{cases} 1 & \text{if } \mathcal{Q}_i[\mathbf{x}(\mathbf{k})] \neq 0 \\ 0 & \text{otherwise} \end{cases}$$
   [S1].
3. **Context Model Calculation:** For each coefficient $\mathbf{k}$ at layer $i$, calculate the spatial context value $\chi^{s,(idx)}[\mathbf{k}]$ based on neighbors in previous ($P$), current ($P$), and next ($P+1$) subband layers (spatial support) [S1]. Let $\mathbf{q}$ be the position of the MSB of the current coefficient [S1]. Let $\mathbf{i}, \mathbf{j}$ be offset vectors for spatial neighbors [S1]. Let $w_P, w_{P+1}, w_{P+2}, w_{P+3}$ be weights for layers $P, P+1, P+2, P+3$ [S1]. Let $\chi^{s,(n_p)}[\mathbf{k}]$ be the context from the previous layer [S1]. The logical OR is $\vee$. The context indices are $(idx)$ where $idx \in \{1n_p0, 1n_p1, 2n_p0, 2n_p1, 3n_p0, 3n_p1\}$ [S1]. Formulas:
   $$
   \begin{aligned}
   \chi^{s,(1n_p0)}[\mathbf{k}] &= w_P \sigma[\mathbf{k} - \mathbf{q}] + \chi^{s,(n_p)}[\mathbf{k}] \\
   \chi^{s,(1n_p1)}[\mathbf{k}] &= w_{P+1} \sigma[\mathbf{k} + \mathbf{q}] + w_P \sigma[\mathbf{k} - \mathbf{q}] + \chi^{s,(n_p)}[\mathbf{k}] \\
   \chi^{s,(2n_p0)}[\mathbf{k}] &= w_{P+1} (\sigma[\mathbf{k} - \mathbf{i} - \mathbf{q}] \vee \sigma[\mathbf{k} + \mathbf{i} - \mathbf{q}] \vee \sigma[\mathbf{k} - \mathbf{j} - \mathbf{q}] \vee \sigma[\mathbf{k} + \mathbf{j} - \mathbf{q}]) + w_P \sigma[\mathbf{k} - \mathbf{q}] + \chi^{s,(n_p)}[\mathbf{k}] \\
   \chi^{s,(2n_p1)}[\mathbf{k}] &= w_{P+2} \sigma[\mathbf{k} + \mathbf{q}] + w_{P+1} (\sigma[\mathbf{k} - \mathbf{j} - \mathbf{q}] \vee \sigma[\mathbf{k} + \mathbf{j} - \mathbf{q}] \vee \sigma[\mathbf{k} - \mathbf{i} - \mathbf{q}] \vee \sigma[\mathbf{k} + \mathbf{i} - \mathbf{q}]) + w_P \sigma[\mathbf{k} - \mathbf{q}] + \chi^{s,(n_p)}[\mathbf{k}] \\
   \chi^{s,(3n_p0)}[\mathbf{k}] &= w_{P+2} (\sigma[\mathbf{k} - \mathbf{i} - \mathbf{q}] \vee \sigma[\mathbf{k} + \mathbf{i} - \mathbf{q}]) + w_{P+1} (\sigma[\mathbf{k} - \mathbf{j} - \mathbf{q}] \vee \sigma[\mathbf{k} + \mathbf{j} - \mathbf{q}]) + w_P \sigma[\mathbf{k} - \mathbf{q}] + \chi^{s,(n_p)}[\mathbf{k}] \\
   \chi^{s,(3n_p1)}[\mathbf{k}] &= w_{P+3} \sigma[\mathbf{k} + \mathbf{q}] + w_{P+2} (\sigma[\mathbf{k} - \mathbf{i} - \mathbf{q}] \vee \sigma[\mathbf{k} + \mathbf{i} - \mathbf{q}]) + w_{P+1} (\sigma[\mathbf{k} - \mathbf{j} - \mathbf{q}] \vee \sigma[\mathbf{k} + \mathbf{j} - \mathbf{q}]) + w_P \sigma[\mathbf{k} - \mathbf{q}] + \chi^{s,(n_p)}[\mathbf{k}]
   \end{aligned}
   $$
   [S1]. For coarsest subbands ($l=L$), no parents exist; use only local-space contribution [S1]. For 3-D/2-D decoding, interband conditioning is excluded to constrain context dimensionality; use bi-dimensional contexts like $(0,6,0)$ [S1].
4. **Encoding (Layered Zero Coding):** Split encoding into Zero Coding and Magnitude Refinement for each layer $i$ [S1]. Zero Coding encodes $\sigma_i[\mathbf{k}]$ using context-adaptive arithmetic coding with context $\chi^{s,(idx)}[\mathbf{k}]$ [S1]. Refinement encodes magnitude bits of significant coefficients [S1]. Scan subbands from lowest resolution to highest [S1].
5. **Bitstream Structuring (Marker Placement):** Choose one of three modes [S1]. G-PROG: No markers. All quantization layers for all subbands are encoded sequentially. Fully embedded [S1]. LPL (Layer Per Layer): All layers for a specific subband are in one segment. Markers placed only between the first ($L_1$) and last ($L_n$) layer of that subband [S1]. LPL-PROG: Markers placed between every quantization layer ($L_i$ and $L_{i+1}$) of the concerned subbands to allow scalable quality decoding [S1].
6. **Decoding (2-D Access):** Input target image index $z$ [S1]. Determine the set of subband images needed to reconstruct image $z$. The number of subbands $N_k$ depends on synthesis filter length and decomposition levels $L$ [S1].
   $$N_k = 4 \times [\text{GP}(L, a)_k + \text{GP}(L, d)_k] + \sum_{l=L-1}^1 [3 \times \text{GP}(l, a)_k + 4 \times \text{GP}(l, d)_k]$$
   where $\text{GP}$ determines low-pass/high-pass filtering along $z$ [S1]. Parse bitstream, stopping at markers relevant to the required subbands [S1]. Decode coefficients for these subbands [S1]. Perform Inverse DWT (IDWT) locally on the decoded coefficients to recover the single 2-D image [S1].

**Parameters:**
* $L$: Decomposition depth. $L=4$ for DSR, MRI, MR-MRI; $L=3$ for ANGIO [S1].
* Filter: 5-3 CDF (CDF 5/3) [S1].
* Context: $(0,6,0)$ is the standard 3-D extension for 2-D spatial context 6 [S1].

**Edge Cases:**
* Boundary Conditions: Lifting scheme guarantees perfect reconstruction for any signal extension, simplifying boundary management [S1].
* Coarsest Subbands: No parents identified; context uses only local spatial neighbors [S1].
* Lossy Oscillation: In lossy mode, PSNR oscillates with period one along $z$ (every other image has better quality) due to rounding/quantization noise [S1].

## Subtopics

### [2-D-PROG](5780-2-d-prog.md)
2-D-PROG is the bi-dimensional variant of the MLZC algorithm, serving as a benchmark for 2-D wavelet-based coding systems and enabling independent encoding of subbands within the 3-D/2-D MLZC framework. It distinguishes itself from general 2-D standards like JPEG2000 by its specific integration into the MLZC context, utilizing spatial support and interband conditioning to optimize lossless compression rates for medical data.

## Operating characteristics
* Complexity: Lifting scheme implementation reduces computational complexity by a factor of 4 compared to classical filter-bank DWT [S1].
* Performance: MLZC-LPL achieves 16.3% rate saving over JPEG2000 for DSR (3.03 vs 3.62 bit/voxel) and 2.2% for MRI (4.55 vs 4.65 bit/voxel) using context (0,6,0) [S1].
* Lossy Quality: Maximum PSNR increase of 4.7 dB (image 40) and 0.8 dB (image 41) over 2-D-PROG at 0.5 bit/pixel for MRI datasets [S1].
* Decoding Delay: Suboptimal in current implementation, but expected to be <1 second per image after optimization [S1].
* Strengths: Fast random access to 2-D slices without decoding the entire volume; lossless capability; embedded bitstream [S1].
* Limitations: Quality oscillation along $z$ in lossy mode; sparseness of conditioning space in LPL-PROG with 3-D contexts can degrade performance [S1].
* Usage: Medical imaging (PACS) for DSR, MRI, MR-MRI, ANGIO datasets [S1].

## Background
MLZC was proposed by Gloria Menegaz and Jean-Philippe Thiran in 2003 [S1]. It combines the 3-D wavelet transform with context-adaptive arithmetic coding, adapting the multidimensional layered zero coding (LZC) strategy [S1]. It addresses the trade-off between the coding efficiency of 3-D algorithms and the fast access of 2-D systems [S1].

## Sources
[S1] Three-dimensional encoding/two-dimensional decoding of medical data, https://doi.org/10.1109/tmi.2003.809689, fetched 2026-10-26
