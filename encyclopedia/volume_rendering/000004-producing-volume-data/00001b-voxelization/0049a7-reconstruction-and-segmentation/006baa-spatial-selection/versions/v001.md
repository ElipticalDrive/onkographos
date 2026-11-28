# Spatial Selection

Also called: Basis Selection for Reconstruction, Subspace Selection

## Overview
Spatial Selection restricts volumetric reconstruction or editing to a specific subset of the data domain, either by selecting a subspace of basis functions or by segmenting the rendered image to identify regions of interest [S1, book][S2]. It differs from global reconstruction methods by reducing computational cost and allowing localized interaction [S1, book]. It differs from simple rectangular cropping by supporting heterogeneous, object-based selections via image-space clustering [S2].

## How it works
Two distinct mechanisms exist for spatial selection: subspace basis selection for mathematical reconstruction and image-space clustering for interactive editing.

### Subspace Basis Selection
This method reconstructs a volume by limiting computation to a specific subrange of the basis matrices [S1, book].

**Inputs:**
- Original basis matrices $U^{(k)} \in \mathbb{R}^{I_k \times R_k}$ for each dimension $k \in \{1, 2, 3\}$ [S1, book].
- Target subspace indices $J_k \subseteq I_k$ [S1, book].

**Outputs:**
- Rank-reduced basis matrices $U_b^{(k)} \in \mathbb{R}^{J_k \times R_k}$ [S1, book].
- Partial spatial reconstruction $A_b$ [S1, book].

**Data Structures:**
- $U^{(k)}$: Matrix of size $I_k \times R_k$. $I_k$ is the full dimension size, $R_k$ is the rank [S1, book].
- $J_k$: List or boolean mask of indices to select from $I_k$ [S1, book].
- $U_b^{(k)}$: Matrix of size $J_k \times R_k$, containing only the rows of $U^{(k)}$ indexed by $J_k$ [S1, book].

**Steps:**
1. Define the target subspace $J_1 \times J_2 \times J_3 \subseteq I_1 \times I_2 \times I_3$ [S1, book].
2. For each dimension $k$, extract the rows of the original basis matrix $U^{(k)}$ corresponding to indices in $J_k$ to form $U_b^{(k)}$ [S1, book].
   $$ U_b^{(k)}[i, :] = U^{(k)}[J_k[i], :] \quad \forall i \in \{0, \dots, |J_k|-1\} $$
3. Perform partial spatial reconstruction $A_b$ using the reduced bases $U_b^{(k)}$ in place of the full bases in the reconstruction equations (Equations 1 or 2 from [S1, book]) [S1, book].

**Edge Cases:**
- If $J_k$ is empty, the reconstruction in that dimension yields zero or undefined values [G].
- If $J_k$ is not contiguous, the resulting basis is sparse in the original domain but dense in the reduced index space [S1, book].

### Image-Space Clustering for Region of Interest Selection
This method selects regions based on visual similarity in the rendered image, replacing rectangular selections with object-based segments [S2].

**Inputs:**
- Rendered volume image (RGB) [S2].
- User click position $(x, y)$ [S2].

**Outputs:**
- Selected region mask (set of pixels) [S2].

**Parameters:**
- Distance Metric: Euclidean distance in RGB color space [S2].
- Clustering Algorithm: GPU-accelerated k-means [S2].

**Steps:**
1. Perform image segmentation on the volume rendered image using GPU-accelerated k-means clustering with RGB Euclidean distance [S2].
2. Store resultant segments as individual masks [S2].
3. When a user clicks a position, identify the segment containing the clicked pixel [S2].
4. Form the selected region consisting of all pixels belonging to that segment [S2].

**Edge Cases:**
- Small or noisy regions may form distinct segments, leading to fragmented selections [G].
- Color similarity may group distinct anatomical structures if their appearance is similar [G].

## Subtopics

### Image-Space Clustering for Region of Interest Selection
A method to select regions of interest by segmenting the rendered image into contiguous visual objects using GPU-accelerated k-means clustering. It replaces rectangular selections with heterogeneously-shaped segments based on color similarity [S2]. It is chosen when the user wants to select objects based on appearance rather than geometric bounds. It results in larger view frustums, more selected voxels, more homogeneous colors across the selection, and consequently smoother transfer function components and relative visibility histograms compared to rectangular selection [S2].

## Operating characteristics
**Subspace Basis Selection:**
- **Time Complexity:** Reduced by the size of the subspace $J_k$ relative to $I_k$ [S1, book].
- **Memory Complexity:** Reduced storage for basis matrices $U_b^{(k)}$ [S1, book].
- **Strengths:** Enables variable resolution and spatial localization in tensor approximation [S1, book].
- **Limitations:** Requires pre-computed basis matrices [S1, book].

**Image-Space Clustering:**
- **Time Complexity:** Dominated by k-means clustering on the image resolution [S2].
- **Strengths:** Produces continuous transfer function components and smoother histograms than rectangular selection [S2].
- **Limitations:** Selections are limited to visible objects and may include unintended adjacent regions with similar colors [S2].

## Background
Subspace selection originates from multiscale tensor approximation for volume data, where reconstruction is limited to subranges of basis matrices to achieve spatial selectivity and variable resolution [S1, book]. Image-space clustering for region selection was proposed to provide a more robust representation of user interest for low-level iterative editing, specifically for transfer function editing using relative visibility histograms [S2].

## Sources
[S1] Multiscale Tensor Approximation for Volume Data, https://www.ifi.uzh.ch/dam/jcr:00000000-5bd6-887a-ffff-ffffb737772b/TA_Volumes.pdf, fetched 2026-10-29 (book)
[S2] Intuitive Transfer Function Editing Using Relative Visibility Histograms, https://doi.org/10.24132/jwscg.2018.26.1.6, fetched 2026-10-26
