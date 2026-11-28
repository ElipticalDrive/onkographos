# NeuRBF
Also called: Neural Fields Representation with Adaptive Radial Basis Functions

## Overview
NeuRBF is a hybrid neural field model that enhances representation accuracy and compactness by combining adaptive radial basis functions (RBFs) with grid-based RBF interpolation [S1, S2]. It generalizes earlier feature grid-based neural field methods by introducing multi-frequency sinusoidal composition to extend the frequencies encoded by each basis function, which are then decoded by an MLP for volume rendering with SDF reconstruction capabilities [S1]. Unlike standard grid-based methods where feature positions are fixed, NeuRBF uses general radial bases with flexible kernel positions and shapes for higher spatial adaptivity, while retaining grid-based RBFs for interpolation smoothness [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A continuous coordinate vector $\mathbf{x} \in \mathbb{R}^D$ (typically $D=3$ for 3D space).
*   **Output:** A signal vector representing the scene property (e.g., density $\sigma$ and color $\mathbf{c}$ for NeRF, or signed distance for SDF) [S2].

**Data Structures:**
The method defines a signal $f$ as a composition $f = g_m \circ g_b$, where $g_b$ extracts local neural features and $g_m$ decodes them [S2].

1.  **Grid-based RBF Nodes:**
    *   Fixed positions $\mathbf{c}_i$ on a regular grid.
    *   Neural features $\mathbf{w}_i \in \mathbb{R}^F$.
    *   Shape parameter $\sigma$ (sidelength of grid cell), shared or uniform per dimension.
2.  **Adaptive RBF Nodes:**
    *   Learnable positions $\mathbf{c}_i \in \mathbb{R}^D$.
    *   Learnable neural features $\mathbf{w}_i \in \mathbb{R}^F$.
    *   Learnable anisotropic shape parameters $\Sigma_i \in \mathbb{R}^{D \times D}$ (symmetric covariance matrix).
3.  **Decoder MLP ($g_m$):**
    *   Takes concatenated features from grid-based and adaptive RBFs as input.
    *   Outputs the final signal.

**Formulas:**

1.  **Grid-based Linear Interpolation (Special Case of RBF):**
    $$ \varphi (\mathbf{x}, \mathbf{c}_i) = \prod_{j=1}^{D} \max \left(0, 1 - \frac{|\mathbf{x}_j - \mathbf{c}_{i,j}|}{\sigma} \right) $$
    where $\sigma$ is the sidelength of each grid cell, and $x_j, c_{i,j}$ are the $j$-th elements of $\mathbf{x}, \mathbf{c}_i$ [S2].

2.  **General RBF Field Formulation ($g_b$):**
    $$ g_b(\mathbf{x}) = \sum_{i \in U(\mathbf{x})} \varphi (\mathbf{x}, \mathbf{c}_i, \sigma_i) \mathbf{w}_i $$
    where $U(\mathbf{x})$ is the set of grid corner nodes that enclose $\mathbf{x}$, $\mathbf{c}_i \in \mathbb{R}^D$ and $\mathbf{w}_i \in \mathbb{R}^F$ are the position and neural feature of node $i$ [S2].

3.  **Inverse Quadratic Kernel (for Adaptive RBFs):**
    $$ \varphi (\mathbf{x}, \mathbf{c}_i, \Sigma_i) = \frac{1}{1 + (\mathbf{x} - \mathbf{c}_i)^T \Sigma_i^{-1} (\mathbf{x} - \mathbf{c}_i)} $$
    where $\Sigma_i$ is a symmetric covariance matrix. Since $\Sigma_i$ is symmetric, it has $D(D-1)/2$ unique parameters [S2].

4.  **Normalized Radial Basis Value:**
    $$ \tilde {\varphi }(\mathbf{x}, \mathbf{c}_i, \Sigma_i) = \frac{\varphi (\mathbf{x}, \mathbf{c}_i, \Sigma_i)}{\sum_{k \in U(\mathbf{x})}\varphi (\mathbf{x}, \mathbf{c}_k, \Sigma_k)} $$
    This normalization ensures stability when combining adaptive RBFs [S2].

5.  **Multi-frequency Sinusoidal Composition:**
    To extend channel-wise capacity, each channel of the feature vector $\mathbf{w}_i$ is combined with a different variant of the RBF using multi-frequency sinusoids. This allows each RBF to encode a wider range of frequencies without extra parameters [S2].
    *Implementation Note:* The sources describe this as composing the RBF with multi-frequency sinusoid functions. Specifically, the feature extraction $g_b$ is extended such that the scalar RBF value is modulated by sinusoidal functions of the coordinate or channel index to create a richer feature representation before aggregation [S2].

**Algorithm Steps:**

1.  **Initialization:**
    *   Initialize adaptive RBF positions using a weighted variant of K-Means [S2].
    *   For 2D image fitting and 3D SDF reconstruction, use direct weighted K-Means on the target data points [S2].
    *   For NeRF reconstruction, use a distillation-based approach because traditional K-Means cannot be directly applied due to indirect supervision (rendering loss vs. pixel values) [S2].
    *   Initialize grid-based RBFs on a regular grid with fixed $\sigma$ [S2].
    *   Initialize adaptive RBF features $\mathbf{w}_i$ and shape parameters $\Sigma_i$ [S2].

2.  **Feature Extraction ($g_b$):**
    *   For a query point $\mathbf{x}$, identify the enclosing grid nodes $U(\mathbf{x})$ for grid-based RBFs.
    *   Compute grid-based weights $\varphi(\mathbf{x}, \mathbf{c}_i)$ using Eq. (1) [S2].
    *   Compute adaptive RBF weights $\tilde{\varphi}(\mathbf{x}, \mathbf{c}_i, \Sigma_i)$ using Eq. (4) for relevant adaptive RBFs (typically those closest to $\mathbf{x}$) [S2].
    *   Aggregate features:
        $$ \mathbf{f}_{grid} = \sum_{i \in U(\mathbf{x})} \varphi (\mathbf{x}, \mathbf{c}_i) \mathbf{w}_i^{grid} $$
        $$ \mathbf{f}_{adaptive} = \sum_{i \in U_{adaptive}(\mathbf{x})} \tilde {\varphi }(\mathbf{x}, \mathbf{c}_i, \Sigma_i) \mathbf{w}_i^{adaptive} $$
        where $U_{adaptive}(\mathbf{x})$ is the set of adaptive RBFs influencing $\mathbf{x}$ (e.g., nearest neighbors) [S2].
    *   Concatenate features: $\mathbf{f}_{combined} = [\mathbf{f}_{grid}; \mathbf{f}_{adaptive}]$ [S2].
    *   Apply multi-frequency sinusoidal composition to $\mathbf{f}_{combined}$ to enhance frequency representation [S2].

3.  **Decoding ($g_m$):**
    *   Pass $\mathbf{f}_{combined}$ through an MLP to predict the final output (density, color, or SDF) [S1, S2].

4.  **Rendering:**
    *   Sample points along rays.
    *   Query $g_b(\mathbf{x})$ and $g_m(\mathbf{f})$ for each sample point.
    *   Perform volume rendering (e.g., differentiable volume rendering) using the predicted density and color [S1].

**Parameters:**
*   $\sigma$: Grid cell sidelength. Not given in the sources.
*   Number of RBFs: Calculated based on parameter budget [S2].
*   Sinusoidal frequencies: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Discontinuity:** Adaptive RBFs can cause discontinuities in $g_b(\mathbf{x})$ when the set of closest RBFs $U(\mathbf{x})$ changes (e.g., at boundaries between RBFs) [S2]. This is mitigated by combining adaptive RBFs with grid-based RBFs via feature concatenation, as grid-based RBFs provide smooth interpolation [S2].
*   **Initialization for NeRF:** Traditional K-Means fails for NeRF due to indirect supervision. A distillation-based approach is required [S2].
*   **Shape Parameter Symmetry:** $\Sigma_i$ must be maintained as a symmetric matrix to ensure valid covariance structure [S2].

## Subtopics

### Adaptive RBF Discontinuity Handling
A technique to resolve discontinuities in the aggregated neural feature $g_b(\mathbf{x})$ when using adaptive RBFs by combining them with grid-based RBFs via feature concatenation [S2]. When a point $\mathbf{x}$ crosses a boundary where the set of closest RBFs $U(\mathbf{x})$ changes (e.g., from $U(\mathbf{x}_1)=\{\text{RBF}_1\}$ to $U(\mathbf{x}_2)=\{\text{RBF}_2\}$), the adaptive RBF contribution can jump because $\varphi(\mathbf{x}_1, \mathbf{c}_1, \Sigma_1) \not\approx \varphi(\mathbf{x}_2, \mathbf{c}_2, \Sigma_2)$ and $\mathbf{w}_1 \not\approx \mathbf{w}_2$ [S2]. Grid-based RBFs avoid this because their weights $\varphi(\mathbf{x}_1, \mathbf{c}_1)$ and $\varphi(\mathbf{x}_2, \mathbf{c}_2)$ are both close to 0 at the boundary, ensuring smoothness. Concatenating both feature types balances fitting accuracy (from adaptive RBFs) and interpolation smoothness (from grid-based RBFs) [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the number of RBFs and the search radius for adaptive RBFs. Grid-based lookup is $O(1)$ for regular grids. Adaptive RBF lookup requires finding nearest neighbors, which can be $O(N)$ or $O(\log N)$ with spatial indexing.
*   **Memory Complexity:** Constant with respect to spatial resolution, determined by the number of RBFs and feature dimension $F$ [S1].
*   **Performance:** Achieves state-of-the-art performance across 2D image fitting, 3D SDF reconstruction, and NeRF synthesis [S1].
*   **Strengths:** Higher spatial adaptivity than fixed grids; compact representation; supports multi-frequency encoding without extra parameters [S2].
*   **Limitations:** Requires careful initialization (K-Means or distillation); potential discontinuities if not combined with grid-based RBFs [S2].
*   **When to use:** When high accuracy and compactness are needed, and the scene has complex structures that benefit from adaptive basis functions.
*   **When not to use:** When simplicity and speed of fixed-grid methods are prioritized over accuracy, or when indirect supervision makes distillation-based initialization impractical.

## Background
NeuRBF was introduced in September 2023 by Lele Chen, Jingyi Yu, Junsong Yuan, and Yi Xu from OPPO US Research Center, University at Buffalo, and ShanghaiTech University [S2]. It is a hybrid neural field model that enhances representation accuracy and compactness by combining adaptive radial basis functions (RBFs) with grid-based RBF interpolation [S1]. The method generalizes earlier feature grid-based neural field methods, introducing multi-frequency sinusoidal composition to extend the frequencies encoded by each basis function [S1]. It achieves state-of-the-art performance across 2D image fitting, 3D signed distance field reconstruction, and neural radiance field synthesis [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] NeuRBF: A Neural Fields Representation with Adaptive Radial Basis Functions, https://doi.org/10.1109/iccv51070.2023.00386, fetched 2026-10-29
