# Ced-NeRF

Also called: Compact and Efficient Dynamic Neural Radiance Fields

## Overview
Ced-NeRF is a method for reconstructing dynamic scenes using neural radiance fields that combines a high-resolution voxel grid with a canonical space deformation model [S1]. It solves the problem of overfitting artifacts (floaters, noisy geometry) common in hybrid representations of dynamic scenes, which typically arise when the grid converges faster than the motion prediction [S1]. It sets itself apart from siblings like [AdaNeRF](6195-adanerf.md) and [Sparse and Efficient Representations](9326-sparse-and-efficient-representations.md) by eschewing sparse voxel grids or separated static/dynamic fields in favor of a single high-resolution grid that enables rapid convergence and fast rendering [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of training images with known camera poses and timestamps $t$.
*   **Outputs:** A predicted color $\bar{C}(\mathbf{r})$ for a camera ray $\mathbf{r}$, and implicitly, the learned deformation field $\Phi_d$ and radiance field $\Phi_r$.

**Data Structures:**
*   **Grid $G$:** A 3D voxel grid storing feature vectors. The specific dimensionality of the feature vectors is not given in the sources [S1]. The grid is indexed by 3D coordinates.
*   **Deformation Network $\Phi_d$:** A neural network mapping encoded spatial coordinates and time to a displacement vector.
*   **Radiance Network $\Phi_r$:** A neural network mapping encoded deformed spatial coordinates and viewing direction to density $\sigma$ and color $\mathbf{c}$.

**Position Encoding:**
Ced-NeRF uses two types of encoding [S1]:
1.  **Frequency Encoding $\gamma(\cdot)$:** Used for raw spatial coordinates $\mathbf{x}$ and viewing direction $\mathbf{d}$. For a coordinate $\mathbf{p}$ and $L$ frequency bands:
    $$
    \gamma(\mathbf{p}) = (\sin(2^0 \pi \mathbf{p}), \cos(2^0 \pi \mathbf{p}), \dots, \sin(2^{L-1} \pi \mathbf{p}), \cos(2^{L-1} \pi \mathbf{p}))
    $$
2.  **Grid Encoding $\gamma_{grid}(\cdot)$:** Used for the deformed spatial coordinate $\mathbf{x}'$. It retrieves features from the grid $G$ via linear interpolation of the 8 closest grid cells [S1]:
    $$
    \gamma_{grid}(\mathbf{x}') = \text{interp}(\mathbf{x}', G)
    $$

**Algorithm Steps:**
1.  **Ray Sampling:** For a camera ray $\mathbf{r} = \mathbf{o} + s\mathbf{d}$, sample $N$ points $\mathbf{r}_i = \mathbf{o} + s_i \mathbf{d}$ along the ray [S1].
2.  **Deformation Prediction:** For each sample point $\mathbf{r}_i$ with timestamp $t$:
    a. Encode the spatial coordinate $\mathbf{x} = \mathbf{r}_i$ and timestamp $t$ using frequency encoding $\gamma(\cdot)$ [S1].
    b. Compute the displacement using the deformation network: $\Delta \mathbf{x} = \Phi_d(\gamma(\mathbf{x}), \gamma(t))$ [S1].
    c. Calculate the deformed coordinate in canonical space: $\mathbf{x}' = \Delta \mathbf{x} + \mathbf{x}$ [S1].
3.  **Feature Retrieval:** Encode the deformed coordinate $\mathbf{x}'$ using grid encoding $\gamma_{grid}(\mathbf{x}')$ to retrieve features from $G$ [S1].
4.  **Direction Encoding:** Encode the viewing direction $\mathbf{d}$ using frequency encoding $\gamma(\mathbf{d})$ [S1].
5.  **Radiance Prediction:** Pass the grid features and direction features into the radiance network $\Phi_r$ to predict density $\sigma_i$ and color $\mathbf{c}_i$ [S1].
6.  **Volume Rendering:** Compute the final pixel color $\bar{C}(\mathbf{r})$ using standard volume rendering [S1]:
    $$
    \bar{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$
    where $\delta_i$ is the distance between samples $s_i$ and $s_{i+1}$, and $T_i = \prod_{j=1}^{i-1} \exp(-\sigma_j \delta_j)$ is the accumulated transmittance [S1].

**Parameters:**
*   **Grid Resolution:** High-resolution grid (specific dimensions not given) [S1].
*   **Frequency Bands $L$:** Not given in the sources [S1].
*   **Number of Samples $N$:** Not given in the sources [S1].

**Pitfalls:**
*   **Overfitting:** Hybrid representations for dynamic scenes often overfit due to the grid converging faster than the motion prediction, leading to floaters and noisy geometry on novel views [S1]. Ced-NeRF mitigates this by using a compact auxiliary network for deformation [S1].

## Operating characteristics
*   **Training Speed:** Fast convergence; reported to train in approximately 4 minutes on certain datasets [S1].
*   **Rendering Speed:** Fast rendering speeds due to the high-resolution grid lookup [S1].
*   **Quality:** Maintains high-quality rendering results, outperforming state-of-the-art methods in quality on dynamic scene datasets [S1].
*   **Comparison:** Outperforms HyperNeRF (9 min / 24.4 dB) and DyNeRF (21 min / 30.6 dB) on specific benchmarks, achieving 4 min / 34.2 dB [S1].
*   **Limitations:** Relies on a high-resolution grid, which may consume significant memory compared to sparse representations, though the sources claim it avoids the slow rendering of separated fields [S1].

## Background
Ced-NeRF was proposed to address the limitations of existing hybrid methods for dynamic neural radiance fields. While explicit-based methods like voxel grids accelerate training, they suffer from overfitting artifacts in dynamic scenes [S1]. The method builds on the canonical space approach introduced in DNeRF [S1] and utilizes the high-resolution grid encoding strategy from Instant-NGP [S1]. It was published in the 38th AAAI Conference on Artificial Intelligence (AAAI-24) [S1].

## Sources
[S1] Ced-NeRF: A Compact and Efficient Method for Dynamic Neural Radiance Fields, https://doi.org/10.1609/aaai.v38i4.28138, fetched 2026-10-26
