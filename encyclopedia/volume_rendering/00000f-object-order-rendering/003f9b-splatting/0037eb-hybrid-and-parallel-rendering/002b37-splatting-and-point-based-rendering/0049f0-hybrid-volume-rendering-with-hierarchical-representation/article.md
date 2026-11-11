# Hybrid Volume Rendering with Hierarchical Representation

Also called: Hybrid Rendering Scheme, Smooth Transition between Splatting and Raycasting

## Overview
A hardware-accelerated hybrid volume rendering scheme that provides a smooth transition between texture-based (raycasting) and point-based (splatting) techniques by dividing the volumetric dataset into hierarchical nodes, each rendered with the most efficient method based on a division criterion [S1, unofficial]. It solves the problem of visual popping artifacts and performance inefficiencies inherent in using a single rendering mode for entire datasets by adapting the technique locally [S1, unofficial]. Unlike pure splatting [Splatting and Point-based Rendering](4975-splatting-and-point-based-rendering.md) or pure raycasting, this method dynamically selects the approximation scheme for each subpart of the dataset to minimize visual variation while optimizing frame rate [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A volumetric dataset (e.g., $512^3$ head model) [S1, unofficial].
- Output: A rendered image using a hybrid representation stored in GPU texture memory [S1, unofficial].

**Data Structures:**
- **Octree:** The input dataset is divided into a collection of hierarchical nodes, where each node represents a subpart of the volumetric dataset [S1, unofficial].
- **3D Texture:** Each region approximated by a single node (meeting the criterion requirement) is stored into a 3D texture of equivalent resolution [S1, unofficial].
- **Node Parameters:** Determined by a division criterion $E$, including sampling rate and 3D texture resolution [S1, unofficial].

**Algorithm Steps:**
1. **Hierarchical Construction:** Divide the input volumetric dataset into an octree structure using a feature-sensitive top-down approach during the construction of the point hierarchy [S1, unofficial].
2. **Criterion Evaluation:** Apply a division criterion $E$ to determine the parameters of the hybrid method for each node. This criterion orients the rendering scheme toward splatting or raycasting for each subpart [S1, unofficial]. Two specific criteria are proposed:
    - **Visual Variation Criterion:** Minimizes visual differences between splatting and raycasting to prevent popping artifacts [S1, unofficial].
    - **Variance Data Criterion:** (Mentioned in results but not detailed in the provided text) [S1, unofficial].
3. **Texture Storage:** Store each region approximated by a single node into a 3D texture of equivalent resolution [S1, unofficial].
4. **Rendering with Pre-integration:** For regions where segment lengths vary with kernel size, use the approximation scheme by Roettger et al. (2000). Weight the result of the 2D lookup (from Engel et al., 2001) with the length $l$ of the segment to ensure correct results while relying on the 2D table lookup [S1, unofficial].

**Formulas:**
The visual variation $v_r$ for a viewing ray $r$ is defined as the distance in color space between two colors computed along the ray by evaluating the volume rendering integral using splatting ($C_{splat}$) and raycasting ($C_{raycast}$) approximation schemes [S1, unofficial]:

$$
v_r = |C_{splat} - C_{raycast}|
$$

The visual variation $V$ over a complete contiguous region is expressed as the average value of the squared variation along all possible viewing rays traversing this region [S1, unofficial]:

$$
V = \frac{\int_{\Omega} v_r^2 \partial \Omega}{\Omega}
$$

where $\Omega$ is the sphere of all possible viewing rays [S1, unofficial].

**Parameters:**
- **Error Threshold ($\epsilon$):** Controls the quality/speed trade-off. Examples from the source: $\epsilon = 2\%$, $5\%$, $10\%$ [S1, unofficial].
- **Kernel Size:** Affects segment lengths in the pre-integration lookup [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Popping Artifacts:** Caused by visual differences between splatting and raycasting solutions. The method aims to minimize this by using the visual variation criterion [S1, unofficial].
- **Variable Segment Lengths:** When segment lengths vary with kernel size, standard 2D lookup tables may be incorrect. The method uses the Roettger et al. (2000) approximation scheme to weight the lookup result [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** On a $512^3$ head dataset [S1, unofficial]:
    - 2M splats with $\epsilon = 2\%$: 2.9 fps [S1, unofficial].
    - 386k splats with $\epsilon = 5\%$: 4.8 fps [S1, unofficial].
    - 250k splats with $\epsilon = 10\%$: 8.8 fps [S1, unofficial] [S1, unofficial].
- **Strengths:** Hardware-accelerated; smooth transition between techniques; reduces popping artifacts; efficient combination of methods for unstructured datasets [S1, unofficial].
- **Limitations:** Requires construction of octree and 3D textures; performance depends on the chosen criterion and error threshold [S1, unofficial].
- **When to use:** When a smooth transition between splatting and raycasting is needed, and hardware acceleration is available [S1, unofficial].
- **When not to use:** When strict real-time performance is required without the overhead of hierarchical construction, or when the dataset does not benefit from hybrid approximation [S1, unofficial].

## Background
The method was presented in the paper "FROM VOLUME SPLATTING TO RAYCASTING - A Smooth Transition" [S1, unofficial]. It builds on earlier work in volume splatting [Splatting and Point-based Rendering](4975-splatting-and-point-based-rendering.md) and raycasting. The hierarchical representation uses an octree structure as in Laur and Hanrahan (1991) [S1, unofficial]. The pre-integrated classification scheme relies on Engel et al. (2001) [S1, unofficial], and the approximation for variable segment lengths uses Roettger et al. (2000) [S1, unofficial]. The visual variation metric is inspired by Meissner et al. (2000) [S1, unofficial].

## Sources
[S1] , https://www.scitepress.org/PublishedPapers/2008/10960/pdf/index.html, fetched 2026-10-27 (unofficial)
