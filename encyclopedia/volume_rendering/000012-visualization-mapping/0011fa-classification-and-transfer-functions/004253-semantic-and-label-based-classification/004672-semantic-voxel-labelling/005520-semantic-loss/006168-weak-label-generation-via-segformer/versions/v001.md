# Weak Label Generation via SegFormer

Also called: SegFormer Weak Labels

## Overview
Weak Label Generation via SegFormer generates weak 3D point labels for unlabeled driving data by training a SegFormer model on a mixture of external datasets and applying label merging strategies for cross-dataset generalization [S2]. It sets itself apart from sibling methods by relying on pre-trained foundation models to achieve annotation-free segmentation through feature-level distribution alignment, rather than requiring manual annotation or relying solely on geometric consistency [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Unlabeled real images collected from multiple cameras of self-driving cars; unlabeled LiDAR point clouds; external labeled datasets (Cityscapes, Mapillary, BDD, IDD) [S2].
*   **Outputs:** Weak 3D point labels for the LiDAR points; a SegFormer model trained for cross-dataset generalization [S2].

**Data Structures:**
*   **SegFormer Model:** A transformer-based image segmentation backbone [S2].
*   **NeRF Model:** A semantic radiance field that renders labels $\hat{S}(r)$ for camera rays $r$ [S2].
*   **LiDAR Points:** Approximately 20,000 to 40,000 points per frame of a standard 32-channel LiDAR [S2].

**Steps:**
1.  **Dataset Preparation:** Combine external datasets (Cityscapes, Mapillary, BDD, IDD) to train the SegFormer model [S2].
2.  **Label Merging:** Apply the label merging strategy from Lambert et al. (2020) to handle conflicts in label definitions and improve cross-dataset generalization [S2].
3.  **Weak Label Prediction:** Use the trained SegFormer model to predict 2D semantic labels $S(r)$ for pixels corresponding to camera rays $r$ in the unlabeled driving images [S2].
4.  **NeRF Integration:** Integrate the predicted 2D labels into the NeRF reconstruction process. For each LiDAR point, render a ray to the NeRF model to obtain the rendered label $\hat{S}(r)$ [S2].
5.  **Loss Calculation:** Compute the weak label loss using the cross-entropy between the rendered label and the predicted label:
    $$ L_l = \text{CE}(\hat{S}(r), S(r)) $$
    where $\text{CE}$ is the cross-entropy loss, $\hat{S}(r)$ is the rendered label by the NeRF model, and $S(r)$ is the label predicted by the SegFormer model [S2].
6.  **Outlier Mitigation:** Reduce the influence of outliers in the generated weak labels by leveraging multi-view geometric and video spatial-temporal consistency during NeRF reconstruction [S2].

**Parameters:**
*   **Loss Weights:** $w_{rgb}$ and $w_l$ balance RGB geometry reconstruction, LiDAR rendering, and semantic label rendering [S2].
*   **Typical Values:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Outliers:** Generated weak labels may contain many outliers [S2]. This is mitigated by using multi-view geometric and video spatial-temporal consistency in NeRF reconstruction [S2].
*   **Label Conflicts:** Conflicts in label definitions across datasets are addressed via the merging strategy from Lambert et al. (2020) [S2].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Enables annotation-free segmentation by aligning class relevancy distribution with pre-trained foundation models [S1, S2].
*   **Limitations:** Generated weak labels may have many outliers, requiring additional consistency checks [S2].
*   **When to Use:** When unlabeled driving data (images and LiDAR) is available but no manual annotations exist [S2].
*   **When Not to Use:** When high-precision manual annotations are available and the overhead of training SegFormer and merging labels is unjustified.

## Background
The method originates from the work on NeRF-LiDAR, which generates realistic LiDAR point clouds with Neural Radiance Fields [S2]. It builds on the concept of semantically-aware NeRFs, where feature-level distribution alignment with pre-trained foundation models allows for precise, annotation-free segmentation [S1]. The label merging strategy is adapted from Lambert et al. (2020) to ensure cross-dataset generalization [S2].

## Sources
[S1] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
[S2] NeRF-LiDAR: Generating Realistic LiDAR Point Clouds with Neural Radiance Fields, https://doi.org/10.1609/aaai.v38i7.28546, fetched 2026-10-29
