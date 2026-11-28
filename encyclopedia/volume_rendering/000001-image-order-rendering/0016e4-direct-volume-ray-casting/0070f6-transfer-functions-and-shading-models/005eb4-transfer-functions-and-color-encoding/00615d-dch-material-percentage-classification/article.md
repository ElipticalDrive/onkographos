# DCH Material Percentage Classification

Also called: DCH Classification, Probabilistic Classifier

## Overview
DCH Material Percentage Classification is a probabilistic method for direct volume ray casting that estimates the fractional presence of materials within a voxel based on intensity statistics [S1, unofficial][S2, unofficial]. Unlike Levoy’s saliency-based approach which assumes regions of change, DCH assumes at most two materials meet per voxel to derive continuous material percentage volumes [S2, unofficial]. It sets itself apart by pre-computing these percentages via segmentation or statistical modeling, allowing optical properties to be derived directly from the material mix rather than raw scalar values [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume dataset and pre-computed material statistics, specifically the probability distributions $P_i(I)$ for each material $i$ [S1, unofficial][S2, unofficial].
*   **Output:** For each sample point along a ray, a color vector $\mathbf{c}_v$ and opacity scalar $\alpha_v$ derived from material percentages [S1, unofficial][S2, unofficial].

**Data Structures:**
*   **Material Percentage Volume:** A volume where each voxel stores the percentage $p_i$ of material $i$ [S1, unofficial].
*   **Probability Distribution:** $P_i(I)$, the probability that material $i$ has intensity value $I$, derived from statistics or physics [S1, unofficial][S2, unofficial].

**Steps:**
1.  **Pre-computation (Segmentation/Statistics):** Derive material percentage volumes from the input data, which are pre-computed via segmentation or intensity statistics [S1, unofficial].
2.  **Estimate Material Percentage ($p_i$):** For a voxel with intensity $I$, estimate the percentage $p_i$ of material $i$ using the probability $P_i(I)$ [S2, unofficial].
3.  **Derive Color and Opacity:** Calculate the color $\mathbf{C}_i$ and opacity $\alpha_i$ for the sample based on the material percentages [S1, unofficial][S2, unofficial].
    *   Let $R_i, G_i, B_i$ be the intrinsic color components of material $i$ [S1, unofficial][S2, unofficial].
    *   The color vector and opacity are defined as:
        $$ \mathbf{C}_i = (p_i R_i, \ p_i G_i, \ p_i B_i, \ p_i) $$
    *   Here, the opacity $\alpha_i$ is equal to the percentage $p_i$ [S1, unofficial][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def classify_voxel(intensity, materials):
    """
    Classifies a voxel based on intensity using probabilistic material percentages.
    Assumes at most two materials per voxel [S1, unofficial][S2, unofficial].
    
    materials: list of dicts with keys 'id', 'R', 'G', 'B', 'prob_func'
    prob_func(I) returns P_i(I) [S1, unofficial][S2, unofficial]
    """
    # Step 1: Estimate material percentages
    percentages = {}
    for m in materials:
        percentages[m['id']] = m['prob_func'](intensity)
        
    # Normalize probabilities to obtain material percentages p_i [G]
    total = sum(percentages.values())
    if total > 0:
        for m_id in percentages:
            percentages[m_id] /= total
    else:
        for m_id in percentages:
            percentages[m_id] = 0.0
            
    # Step 2: Derive Color and Opacity [S1, unofficial][S2, unofficial]
    # Ci = (pi * Ri, pi * Gi, pi * Bi, pi)
    color_r, color_g, color_b, opacity = 0.0, 0.0, 0.0, 0.0
    for m in materials:
        p = percentages[m['id']]
        color_r += p * m['R']
        color_g += p * m['G']
        color_b += p * m['B']
        opacity += p
        
    return (color_r, color_g, color_b), opacity
```

**Parameters:**
*   $p_i$: The percentage of material $i$ in the voxel [S2, unofficial].
*   $P_i(I)$: The probability that material $i$ has intensity value $I$ [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Normalization:** The sources do not specify how to handle cases where the sum of probabilities is zero or how to clamp the final opacity if the sum of percentages exceeds 1.0 [G: Typically, opacities are clamped to $[0,1]$ or normalized].
*   **Boundary:** The assumption of at most two materials per voxel is a simplification [S1, unofficial][S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The classification step requires evaluating probability functions for each material at each sample point, which is generally $O(M)$ per sample where $M$ is the number of materials [G]. Pre-computation is performed once [S1, unofficial].
*   **Memory Complexity:** Requires storing the material percentage volumes and the probability distributions [S1, unofficial][S2, unofficial].
*   **Strengths:** Provides a physically motivated way to handle mixed voxels [S1, unofficial][S2, unofficial].
*   **Limitations:** Relies on accurate pre-computed statistics or segmentation, and the two-material assumption limits its applicability to complex mixtures [S1, unofficial][S2, unofficial].
*   **When to use:** Suitable when the volume data contains distinct materials with known intensity distributions and the goal is to visualize material composition rather than just iso-surfaces [S1, unofficial][S2, unofficial].

## Background
DCH Material Percentage Classification is a probabilistic classifier used in direct volume rendering [S1, unofficial][S2, unofficial]. It contrasts with Levoy’s earlier work on saliency-based classification by focusing on material percentages rather than regions of change [S2, unofficial]. Probabilistic methods for deriving material volumes were noted in early direct volume rendering literature [S1, unofficial].

## Sources
[S1] PPT - Direct Volume Rendering (DVR): Ray-casting PowerPoint Presentation - ID:465503, https://slideserve.com/jariath/direct-volume-rendering-dvr-ray-casting, fetched 2026-10-24 (unofficial)
[S2] Direct volume rendering ray casting, https://slidetodoc.com/direct-volume-rendering-dvr-raycasting-jian-huang-this/, fetched 2026-10-29 (unofficial)
