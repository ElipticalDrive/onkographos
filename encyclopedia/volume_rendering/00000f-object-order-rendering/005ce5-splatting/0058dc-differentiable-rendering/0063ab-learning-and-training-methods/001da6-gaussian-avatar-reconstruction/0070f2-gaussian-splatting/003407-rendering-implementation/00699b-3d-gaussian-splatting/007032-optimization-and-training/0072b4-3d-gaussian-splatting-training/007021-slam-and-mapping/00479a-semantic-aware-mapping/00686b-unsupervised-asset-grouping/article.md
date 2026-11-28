# Unsupervised Asset Grouping

Also called: Asset Grouping via JSD Clustering

## Overview
Unsupervised Asset Grouping is a method for categorizing discovered 3D objects by comparing the distributions of color and semantic features among image patches using Jensen-Shannon Divergence (JSD) and agglomerative clustering [S1]. It solves the problem of object categorization in the absence of explicit object pose information, where pixel-wise comparison is insufficient [S1]. Unlike sibling methods that may rely on supervised labels or rigid pose priors, this approach operates directly on the ground feature plane to group assets based on feature distribution similarity, enabling the construction of a cross-scene asset library [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An inferred RGB-DINO ground feature plane $M_{\text{rgb-dino}}$, constructed by interpolating encoded scene color $c$ and semantic features $f_{\text{dino}}$ via decoders $\text{Dec}_{\text{rgb}}$ and $\text{Dec}_{\text{dino}}$ [S1].
*   **Output:** A set of labeled object feature patches $K$, grouped into clusters, and a set of template objects selected from each cluster [S1].

**Data Structures:**
*   $M_{\text{rgb-dino}}$: A 2D feature plane where each pixel $(x, y)$ maps to a feature vector via interpolation [S1].
*   $K$: A set of labeled object feature patches obtained by discretizing $M_{\text{rgb-dino}}$ [S1].
*   $P_{\text{rgb-dino}}$: A set of patches used for layout extraction, where each patch $p$ is a sub-region of $M_{\text{rgb-dino}}$ [S1].

**Steps:**
1.  **Discretization:** Discretize the RGB-DINO ground feature plane $M_{\text{rgb-dino}}$ using a clustering algorithm (e.g., K-means) to produce a set of labeled object feature patches $K$ [S1].
2.  **Similarity Measurement:** For any two object patches $k_i, k_j \in K$, measure their similarity using the Jensen-Shannon Divergence over the distribution of labels, denoted as $\text{JSD}(k_i || k_j)$ [S1].
3.  **Clustering:** Perform agglomerative clustering using the JSD distance metric to group patches into categories [S1]. The number of clusters is determined by inspecting training views and can be adjusted to fit desired granularity [S1].
4.  **Template Selection:** Select a template object from each cluster either randomly or in a user-defined manner [S1].
5.  **Layout Extraction (Optional):** Extract scene layout in Bird's Eye View (BEV) by optimizing a rotation angle $\theta$ that minimizes the pixel-wise loss between the template RGB-DINO feature patch $\hat{p}$ and each copy $p$ rotated by $\theta$ [S1].

**Formulas:**
The similarity between patches is defined by JSD. The optimal rotation angle $\theta^*$ for layout extraction is computed as [S1]:

$$
\theta^* = \arg\min_\theta \sum_i \|\hat{p}_i - R_\theta(p_i)\|_2^2
$$

where:
*   $\hat{p}$ is the template RGB-DINO feature patch.
*   $p \in P_{\text{rgb-dino}}$ is a copy of the feature patch.
*   $R_\theta$ rotates the input feature patch by angle $\theta$ [S1].

**Parameters:**
*   **Clustering Algorithm:** K-means is suggested for discretization [S1].
*   **Number of Clusters:** Not fixed; set by inspecting training views [S1].

**Edge Cases and Pitfalls:**
*   **Absence of Pose:** Pixel-wise comparison is not ideal due to the absence of object pose information; distribution-based comparison (JSD) is required [S1].
*   **Granularity:** The number of clusters must be manually inspected/adjusted to fit user-defined categorization granularity [S1].

## Subtopics

### Cross-scene Asset Library Construction
A framework to represent a scene using template feature patches, BEV layout, shared vertical feature axes, and MLP decoders, enabling cross-scene asset discovery by sharing vertical axes among different scenes [S1]. It constructs a library storing template feature patches by sharing vertical axes across scenes, allowing newly configured scenes to be rendered without retraining using shared axes and decoders with new template patches and layout [S1].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Evaluated on a synthetic dataset of 10 scenes resembling common man-made environments [S1].
*   **Strengths:** Enables versatile scene editing at object-, category-, and scene-level; supports cross-scene analogue discovery; allows rendering new scenes without retraining [S1].
*   **Limitations:** Requires inspection of training views to set cluster count; relies on the quality of the RGB-DINO feature plane [S1].
*   **When to use:** When unsupervised categorization of assets and cross-scene asset library construction are required [S1].
*   **When not:** When explicit object pose information is available and pixel-wise comparison is sufficient [S1].

## Background
Originated in the paper "AssetField: Assets Mining and Reconfiguration in Ground Feature Plane Representation" [S1]. The method addresses the lack of constraints in previous voxel-based neural representations that prevented cross-scene analogue discovery, leveraging the potential of neural representations to share vertical feature axes among different scenes [S1].

## Sources
[S1] AssetField: Assets Mining and Reconfiguration in Ground Feature Plane Representation, https://doi.org/10.1109/iccv51070.2023.00301, fetched 2026-10-29
