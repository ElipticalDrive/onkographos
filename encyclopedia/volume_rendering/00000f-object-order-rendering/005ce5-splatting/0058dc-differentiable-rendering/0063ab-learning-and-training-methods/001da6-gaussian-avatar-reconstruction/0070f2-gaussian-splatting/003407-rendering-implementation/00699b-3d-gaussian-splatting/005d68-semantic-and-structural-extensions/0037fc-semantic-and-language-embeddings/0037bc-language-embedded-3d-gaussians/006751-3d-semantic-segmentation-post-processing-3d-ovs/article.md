# 3D Semantic Segmentation Post-processing (3D-OVS)

Also called: 3D-OVS Semantic Segmentation Pipeline

## Overview
3D-OVS is a post-processing pipeline for semantic segmentation in 3D Gaussian Splatting systems that resolves ambiguity by selecting the optimal semantic level from multiple candidate relevancy maps. It solves the problem of inconsistent or noisy semantic predictions by thresholding, binarizing, and evaluating average relevancy within masks to choose the best binary mask [S1]. It differs from direct thresholding by explicitly comparing multiple semantic levels (derived from different granularities) to mitigate outlier impact [S1].

## How it works
**Inputs:**
- Three relevancy maps for a single class query, representing three distinct semantic levels defined by SAM (Segment Anything Model) [S1].
- A threshold value $T = 0.4$ [S1].

**Outputs:**
- A single binary mask representing the final semantic segmentation for the queried class [S1].

**Data Structures:**
- **Relevancy Map:** A 2D image (or 3D volume projected to 2D) where each pixel/voxel contains a float32 score in $[0, 1]$ indicating the relevance of that location to the semantic query [S1].
- **Binary Map:** A 2D image where each pixel/voxel is 0 or 1, derived from thresholding the relevancy map [S1].

**Steps:**
1. **Thresholding:** For each of the three relevancy maps, apply a hard threshold $T = 0.4$.
   - If score $< 0.4$, set to 0.
   - If score $\ge 0.4$, set to 1.
   - This produces three binary maps [S1].
2. **Mask Region Calculation:** Identify the region of interest (mask) for each binary map. This is typically the set of pixels where the binary value is 1 [S1].
3. **Average Relevancy Calculation:** For each binary map, calculate the average relevancy score of the original (pre-thresholded) relevancy map values within the mask region defined by that binary map.
   - Let $R_i$ be the $i$-th relevancy map.
   - Let $B_i$ be the corresponding binary map.
   - Let $M_i = \{p \mid B_i[p] = 1\}$ be the mask region.
   - Calculate average relevancy $A_i$:
     $$ A_i = \frac{1}{|M_i|} \sum_{p \in M_i} R_i[p] $$
   - If $|M_i| = 0$, $A_i$ is undefined or treated as 0 [G].
4. **Selection:** Select the binary map $B_k$ corresponding to the highest average relevancy $A_k$ as the final predicted binary mask [S1].

**Parameters:**
- Threshold $T = 0.4$ [S1].

**Edge Cases and Pitfalls:**
- **Empty Mask:** If a binary map has no pixels above threshold, the average relevancy cannot be calculated. The implementation must handle this, e.g., by assigning a score of 0 or ignoring that candidate [G].
- **Ties:** If multiple maps have the same highest average relevancy, the selection strategy is not specified in the sources [S1]. A deterministic tie-breaker (e.g., lowest index) is recommended [G].
- **Precision:** Relevancy scores are floats; thresholding is a hard cut, which may lead to jagged boundaries if scores are near 0.4 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_best_semantic_mask(relevancy_maps, threshold=0.4):
    """
    Selects the best semantic binary mask from three relevancy maps.
    
    Args:
        relevancy_maps: List of 3 numpy arrays (H, W), float32, values in [0, 1].
        threshold: Float, threshold for binarization.
        
    Returns:
        best_binary_map: numpy array (H, W), uint8, 0 or 1.
    """
    binary_maps = []
    average_relevancies = []
    
    for r_map in relevancy_maps:
        # Step 1: Thresholding
        b_map = (r_map >= threshold).astype(np.uint8)
        binary_maps.append(b_map)
        
        # Step 2 & 3: Calculate average relevancy within mask
        mask_region = b_map == 1
        num_pixels = np.sum(mask_region)
        
        if num_pixels == 0:
            # Edge case: empty mask
            avg_rel = 0.0
        else:
            # Sum of relevancy scores in mask region
            sum_rel = np.sum(r_map[mask_region])
            avg_rel = sum_rel / num_pixels
            
        average_relevancies.append(avg_rel)
        
    # Step 4: Select best
    best_index = np.argmax(average_relevancies)
    best_binary_map = binary_maps[best_index]
    
    return best_binary_map
```

## Subtopics

### Semantic Surface Trimming
A post-processing or training-time strategy to prune unwanted background elements (e.g., sky) from 3D Gaussian reconstructions by learning a semantic field and using it to mask out non-object regions [S2]. It assigns each Gaussian a learnable semantic feature vector, renders pixel-level semantics via alpha blending, and trains these features using cross-entropy loss against pseudo-labels from a pretrained model (e.g., Grounded-SAM) [S2]. It is chosen when background noise (like sky) degrades reconstruction quality [S2]. It differs from direct thresholding by learning semantic features that are robust to noisy pseudo-labels, yielding more accurate semantics than directly using pretrained semantic maps [S2]. On the TNT dataset, it increases the F1-score from 0.38 to 0.40 [S2].

## Operating characteristics
- **Time Complexity:** Not given in the sources. The post-processing involves thresholding and averaging over pixels, which is linear in the number of pixels $N$ for each of the 3 maps: $O(N)$ [S1].
- **Memory Complexity:** Requires storing 3 relevancy maps and 3 binary maps temporarily: $O(N)$ [S1].
- **Measured Performance:** On the 3D-OVS dataset, the method consistently outperforms other state-of-the-art methods in Accuracy and mIoU [S1]. On the TNT dataset, semantic trimming (a related post-processing step) increases F1-score by 0.02 [S2].
- **Scaling:** Performance scales with image resolution; higher resolution means more pixels to process [S1].
- **Strengths:** Mitigates outlier impact by selecting the best semantic level; robust to noisy pseudo-labels when combined with semantic trimming [S1, S2].
- **Limitations:** Relies on the quality of the initial relevancy maps; threshold is fixed at 0.4, which may not be optimal for all scenes [S1].
- **When to use:** When multiple semantic levels are available and ambiguity exists between them [S1].
- **When not to use:** When a single semantic level is sufficient or when real-time performance is critical and the overhead of evaluating multiple maps is prohibitive [S1].

## Background
Originated in the LangSplat paper (CVPR 2024) by the authors of 3D Language Gaussian Splatting [S1]. It is used in 3D-OVS (3D Object Localization and Semantic Segmentation) tasks to improve the accuracy of semantic predictions by leveraging multiple semantic levels [S1]. The method builds on the concept of 3D language Gaussians, which encode object-level embeddings for phrase grounding [S1].

## Sources
[S1] LangSplat: 3D Language Gaussian Splatting, https://doi.org/10.1109/cvpr52733.2024.01895, fetched 2026-10-26
[S2] VCR-GauS: View Consistent Depth-Normal Regularizer for Gaussian Surface Reconstruction, https://doi.org/10.52202/079017-4434, fetched 2026-10-29
