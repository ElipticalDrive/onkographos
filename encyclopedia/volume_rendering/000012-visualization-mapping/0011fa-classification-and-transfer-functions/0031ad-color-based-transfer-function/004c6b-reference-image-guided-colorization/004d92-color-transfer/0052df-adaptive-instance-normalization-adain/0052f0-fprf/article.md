# FPRF

Also called: FPRF

## Overview
FPRF is a method for photorealistic 3D style transformation that stylizes scenes without per-scene optimization, using a radiance field for geometry and a style dictionary with attention for content retrieval [S1]. It sets itself apart from iterative optimization-based siblings like SNeRF by leveraging Adaptive Instance Normalization (AdaIN) to enable rapid, zero-shot style manipulation with multiple reference images [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scene represented by a radiance field (geometry and content), and a set of reference images defining target styles.
*   **Output:** Stylized 3D renderings that maintain geometric consistency while adopting the artistic appearance of the references.

**Data Structures:**
*   **Radiance Field:** Implicit representation capturing scene geometry and content [S1].
*   **Style Dictionary Module:** A structure storing style features from multiple reference images, equipped with a style attention mechanism to retrieve matching styles for diverse content [S1].

**Steps:**
1.  **Scene Representation:** Encode the input 3D scene geometry and content into a radiance field [S1].
2.  **Style Retrieval:** Use the style dictionary module with style attention to identify and retrieve relevant style features from the provided reference images based on the current content [S1].
3.  **Stylization via AdaIN:** Apply Adaptive Instance Normalization (AdaIN) to align the feature maps of the radiance field's content with the retrieved style features [S1]. This step performs the transformation without requiring iterative optimization [S1].
4.  **Rendering:** Render the stylized scene from desired viewpoints [S1].

**Formulas:**
*   The core stylization operation relies on AdaIN [S1]. Let $x$ be the content feature map and $y$ be the style feature map. AdaIN normalizes $x$ and scales it using the mean and variance of $y$:
    $$ \text{AdaIN}(x, y) = \sigma_y \left( \frac{x - \mu_x}{\sqrt{\sigma_x^2 + \epsilon}} \right) + \mu_y $$
    where $\mu_x, \sigma_x$ are the mean and standard deviation of $x$, and $\mu_y, \sigma_y$ are the mean and standard deviation of $y$, and $\epsilon$ is a small constant for numerical stability [S1, G].

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   The method assumes the radiance field accurately captures the scene geometry and content [S1].
*   Style attention must effectively match content to reference styles; poor matching may lead to inconsistent stylization [S1].

## Subtopics

### StyleRF
StyleRF is a zero-shot 3D style transfer framework that performs style transformations in the feature space of a radiance field [S1]. It distinguishes itself by introducing sampling-invariant content transformation and deferring style transformation to achieve multi-view consistency and improved efficiency [S1].

## Operating characteristics
*   **Time Complexity:** Faster than iterative optimization methods because it avoids per-scene optimization loops [S1].
*   **Memory Complexity:** Depends on the radiance field representation and the style dictionary size [S1].
*   **Performance:** Outperforms previous 3D photorealistic appearance editing methods by achieving transformation with multiple style references in less training time [S1].
*   **Strengths:** Rapid style manipulation, support for multiple style references, no per-scene optimization required [S1].
*   **Limitations:** Relies on the quality of the radiance field and style attention mechanism [S1].
*   **When to use:** When rapid, zero-shot stylization of 3D scenes with multiple style references is needed.
*   **When not to use:** When per-scene optimization is required for higher fidelity or specific artistic control not covered by the style dictionary [S1].

## Background
FPRF was introduced as a method to achieve photorealistic style transformation in 3D scenes without optimization [S1]. It builds on Adaptive Instance Normalization (AdaIN) [S1] and utilizes a radiance field for geometry and a style dictionary with attention for style retrieval [S1]. It is cited in a survey on 3D editing based on NeRF and 3DGS [S1].

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
