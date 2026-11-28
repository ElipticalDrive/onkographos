# AdaNeRF

Also called: AdaNeRF: Adaptive Sampling for Real-time Rendering of Neural Radiance Fields

## Overview
AdaNeRF is a method that employs fixed sample positions along each ray and incrementally introduces sparsity during training to achieve real-time rendering [S1]. It replaces the standard uniform sampling of vanilla NeRF with a lightweight Sampling Network that evaluates once per ray to predict sample importance weights, allowing a heavier Shading Network to process only the most significant points [S1]. This approach reduces the number of sample points required, thereby lowering rendering cost while maintaining image quality [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera ray parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), number of total samples $N$, and sparsity schedule parameters.
*   **Output:** Pixel color $\mathbf{C}(\mathbf{r})$.

**Data Structures:**
1.  **Sampling Network ($f_{\text{sample}}$):** A lightweight MLP.
    *   Input: Ray direction $\mathbf{d}$ (and potentially origin $\mathbf{o}$).
    *   Output: A vector of weights $\mathbf{w} \in \mathbb{R}^N$ corresponding to $N$ fixed positions along the ray.
2.  **Shading Network ($f_{\text{shade}}$):** A standard NeRF MLP.
    *   Input: 3D coordinates $\mathbf{x}$ and viewing direction $\mathbf{d}$.
    *   Output: Volume density $\sigma$ and color $\mathbf{c}$.
3.  **Fixed Sample Positions:** A set of $N$ pre-defined distances $\{\delta_1, \dots, \delta_N\}$ along the ray, typically distributed uniformly or according to a depth prior.

**Steps:**
1.  **Initialization:** Train a standard NeRF model to convergence to initialize the Shading Network and establish a baseline for scene representation [S1].
2.  **Architecture Split:** Decompose the model into the Sampling Network and the Shading Network [S1].
3.  **Weight Prediction:** For each ray, pass $\mathbf{d}$ through $f_{\text{sample}}$ to obtain importance weights $\mathbf{w}$ [S1].
4.  **Sparsity Introduction:** During training, incrementally reduce the number of active samples. This is achieved by masking or zeroing out weights $\mathbf{w}_i$ that fall below a threshold or by selecting only the top-$K$ indices based on $\mathbf{w}$ [S1].
5.  **Shading:** Evaluate $f_{\text{shade}}$ only at the selected sample positions $\mathbf{x}_i = \mathbf{o} + \delta_i \mathbf{d}$ for the active indices [S1].
6.  **Rendering:** Compute the final color using the standard volume rendering integral with the predicted $\sigma$ and $\mathbf{c}$ from the Shading Network [S1].
7.  **Fine-tuning:** Fine-tune the entire system (Sampling and Shading Networks) jointly to the desired number of samples to create a compact neural representation [S1].

**Formulas:**
The importance weight for the $i$-th sample is predicted by:
$$
\mathbf{w} = f_{\text{sample}}(\mathbf{d})
$$
The final color is computed as:
$$
\mathbf{C}(\mathbf{r}) = \sum_{i \in \text{Active}} T_i \alpha_i \mathbf{c}_i
$$
where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$, and $\text{Active}$ is the subset of indices determined by the sparsity mask derived from $\mathbf{w}$ [S1].

**Parameters:**
*   $N$: Total number of fixed sample positions. Typical values are not explicitly given in the sources, but the method aims for "real-time" performance, implying significantly lower $N$ than standard NeRF (e.g., 64-128 vs 192-256) [S1].
*   Sparsity Schedule: Controls the rate at which samples are pruned during training [S1].

**Edge Cases and Pitfalls:**
*   **Initialization:** The method requires a pre-trained NeRF to initialize the color network to achieve the best results, as training end-to-from scratch may lead to suboptimal convergence [S1].
*   **Geometric Ambiguity:** By avoiding the need for a depth map to guide sampling, the method improves quality in geometrically ambiguous regions compared to depth-guided approaches [S1].

## Operating characteristics
*   **Time Complexity:** Reduced compared to vanilla NeRF due to fewer evaluations of the Shading Network. The Sampling Network is lightweight, adding minimal overhead [S1].
*   **Memory Complexity:** Lower memory usage as fewer points are processed by the heavier Shading Network [S1].
*   **Performance:** Capable of real-time rendering after fine-tuning to the desired number of samples [S1].
*   **Strengths:** Adaptable to meet specific performance targets; improves quality in geometrically ambiguous regions by not relying on depth maps [S1].
*   **Limitations:** Requires a pre-trained NeRF for initialization [S1].

## Background
AdaNeRF was introduced by Kurz et al. [KNL* 22] in the paper "AdaNeRF: Adaptive Sampling for Real-time Rendering of Neural Radiance Fields" presented at ECCV 2022 [S2, unofficial]. It is part of the broader trend of accelerating NeRF by reducing the number of sample points, alongside methods like NSVF and ENeRF [S2, unofficial]. The approach builds on the concept of using a sampling network to localize surfaces or determine importance, similar to TermiNeRF [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
