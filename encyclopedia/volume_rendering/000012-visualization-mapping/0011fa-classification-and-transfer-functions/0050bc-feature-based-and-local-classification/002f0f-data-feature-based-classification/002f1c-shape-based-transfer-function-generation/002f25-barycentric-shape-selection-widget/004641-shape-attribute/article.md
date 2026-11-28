# Shape Attribute
Also called: Forma
## Overview
Shape Attribute (Forma) is a feature-based classification operator prepared on segmented volumes to discriminate objects by examining three shape aspects: tubiness (cylindrical aspect), surfaceness (planar aspect), and blobbiness (bubble aspect) [S1, unofficial]. It generates a transfer function that maps these geometric properties to visual attributes, distinct from intensity- or size-based transfer functions [S1, unofficial]. The method requires the input volume to be partitioned into distinct objects prior to evaluation [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Input:** A segmented volume dataset [S1, unofficial].
*   **Output:** A transfer function that maps the computed shape aspects to visual attributes [S1, unofficial].
**Data Structures:**
Not given in the sources. [G]
**Steps:**
1. Prepare the volume dataset by segmenting it into distinct objects [S1, unofficial].
2. For each object, examine three geometric aspects to discriminate it: tubiness (cylindrical aspect), surfaceness (planar aspect), and blobbiness (bubble aspect) [S1, unofficial].
3. Construct a shape-based transfer function using the evaluated aspects to visualize the object [S1, unofficial].
**Formulas:**
Not given in the sources. [G]
**Parameters:**
*   **Segmentation Thresholds:** Not given in the sources. [G]
*   **Neighborhood Size:** Not given in the sources. [G]
**Edge Cases and Pitfalls:**
Not given in the sources. [G]
## Subtopics

### [Geometric Anisotropy Metrics (Shape Domain)](12138-geometric-anisotropy-metrics-shape-domain.md)
A set of three scalar values derived from the eigenvalues of a symmetric tensor to characterize shape within a triangular domain. Unlike the parent Shape Attribute operator which works on general segmented volumes, these metrics are specifically defined for diffusion tensors to distinguish between fiber-like, sheet-like, and isotropic structures. Choose this method when orientation-independent shape discrimination of tensor data is required.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [G]
*   **Memory Complexity:** Not given in the sources. [G]
*   **Strengths:** Enables intuitive selection of shape classes (tubular, planar, spherical/bubble) in segmented volumes, enhancing visualization of specific structural features [S1, unofficial].
*   **Limitations:** Requires pre-segmented data as a precondition, and the specific computational cost of the shape discrimination is not detailed [S1, unofficial].
## Background
The Shape Attribute operator (Forma) was proposed by Praßni et al. in 2010 [S1, unofficial]. It utilizes pyramid schemes for categorizing objects within volumes [S1, unofficial]. The method is positioned alongside size-based and intensity-based transfer functions, providing a distinct geometric perspective for volume visualization [S1, unofficial]. The approach adapts concepts originally described by Correa and Ma (2008) [S1, unofficial].
## Sources
[S1] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
