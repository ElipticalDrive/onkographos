# Visibility Histogram Refinement

Also called: Correa and Ma technique

## Overview
Visibility Histogram Refinement is a method for automatically adjusting transfer function (TF) parameters to reveal structures that are occluded from the observer's current viewpoint [S1, unofficial]. It solves the problem of manual fine-tuning by combining visibility histograms with a non-assisted approach to refine initial TF specifications, ensuring that hidden regions and structures become visible [S1, unofficial]. It sets itself apart from sibling methods by focusing on the refinement of an existing specification to resolve occlusion, rather than generating a TF from scratch based on region size or occlusion metrics alone [S1, unofficial].

## How it works
**Inputs:**
1. A preliminary user-specified transfer function [S1, unofficial].
2. The volume dataset [S1, unofficial].
3. The current camera/viewpoint [S1, unofficial].

**Outputs:**
1. A refined transfer function that prioritizes the visibility of regions and structures hidden from the observer's position [S1, unofficial].

**Data Structures:**
The method relies on the concept of a visibility histogram, which represents the distribution of visible voxels. While the core "Visibility Histogram Refinement" technique described by Correa and Ma does not explicitly detail the internal bit-packing of the histogram in the provided sources, it operates on the principle of weighting data based on visibility proportions [S2].

**Steps:**
1. **Preliminary Specification:** The user performs an initial specification of the transfer function [S1, unofficial].
2. **Visibility Analysis:** The system analyzes the visibility of voxels from the current observer position. This involves determining which voxels are occluded by others along the view rays [S1, unofficial].
3. **Refinement:** The initial transfer function is refined. The refinement process adjusts the TF parameters (such as opacity and color mappings) to increase the visibility of regions and structures that were previously hidden or occluded [S1, unofficial]. This is achieved by leveraging the visibility information to weight or adjust the contribution of specific intensity ranges [S2].

**Parameters:**
Not given in the sources for the core Correa and Ma technique.

**Edge Cases and Pitfalls:**
- **Precision:** Not given in the sources.
- **Boundaries:** Not given in the sources.
- **Degenerate Input:** If the preliminary TF renders nothing visible, the refinement may struggle to identify hidden structures without additional user guidance or heuristics. Not given in the sources.

## Subtopics

### Relative Visibility Histogram
A histogram representing the difference between the local visibility distribution within a user-selected region of interest (ROI) and the global visibility distribution across the full volume, normalized to the range [-1, 1] [S2]. It is used to infer which 3D volume features correspond to a 2D selection [S2].

**Definition:**
- **Global Visibility Histogram ($H$):** Represents the visibility distribution of all voxels in the viewport from a given view, accounting for occlusion [S2].
- **Local Visibility Histogram ($H_L$):** Represents the visibility distribution for voxels contributing to a specific Region of Interest (ROI) in the rendered image (e.g., voxels in the frustum extended by a rectangular screen selection) [S2].
- **Relative Visibility Histogram ($H_G$):** Derived from the difference between $H_L$ and $H$ [S2].

**Steps:**
1. Compute the Global Visibility Histogram $H$ for the entire volume from the current view [S2].
2. Compute the Local Visibility Histogram $H_L$ for the selected ROI [S2].
3. Calculate the raw difference histogram $H_r = H_L - H$ [S2].
4. Apply a Gaussian smoothing kernel with parameters $n$ (size) and $\sigma$ (standard deviation) to $H_r$ to obtain $H_g$ [S2].
5. Normalize the smoothed histogram to the range [-1, 1] to obtain the final Relative Visibility Histogram $H_G$ [S2].

**Formula:**
$$H_r = H_L - H$$
$$H_g = \text{Gaussian}(H_r, n, \sigma)$$
$$H_G = H_g / \max(|H_g|)$$

Where:
- $H_r$ is the raw difference histogram [S2].
- $H_g$ is the smoothed histogram before normalization [S2].
- $H_G$ is the final relative visibility histogram [S2].
- $n$ is the size of the Gaussian kernel [S2].
- $\sigma$ is the standard deviation of the Gaussian kernel [S2] [G].

**Parameters:**
- $n$: The size of the Gaussian kernel used for smoothing the histogram [S2].
- $\sigma$: The standard deviation of the Gaussian kernel used for smoothing the histogram [S2].

**When to choose:**
Choose this variant when the user wants to interactively edit the transfer function based on a 2D selection in the rendered image, allowing for intuitive "painting" of 3D features by weighting colors and opacities based on the relative visibility of materials in the selected region [S2].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
Not given in the sources.

**Scaling:**
Not given in the sources.

**Strengths and Limitations:**
- **Strengths:** Allows users to manipulate visualization with no explicit knowledge of the transfer function; enables feature-level editing by manipulating colors and opacity values of components [S2].
- **Limitations:** Requires a preliminary user specification or ROI selection; relies on accurate visibility computation which can be computationally expensive for large volumes [S1, unofficial][S2].

**When to use it and when not:**
Use it when the goal is to refine an existing visualization to reveal occluded structures or when interactive, region-based transfer function editing is desired [S1, unofficial][S2]. Do not use it when a fully automated, unsupervised transfer function generation is required without user input [S1, unofficial].

## Background
The technique was proposed by Correa and Ma in 2009 [S1, unofficial]. It is part of a broader effort to automate and simplify transfer function assignment in volume rendering, specifically focusing on visibility-based optimization [S1, unofficial]. The concept of relative visibility histograms was further developed and applied in later works, such as the 2018 paper by Sereda et al. [S2].

## Sources
[S1] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
[S2] Intuitive Transfer Function Editing Using Relative Visibility Histograms, https://doi.org/10.24132/jwscg.2018.26.1.6, fetched 2026-10-26
