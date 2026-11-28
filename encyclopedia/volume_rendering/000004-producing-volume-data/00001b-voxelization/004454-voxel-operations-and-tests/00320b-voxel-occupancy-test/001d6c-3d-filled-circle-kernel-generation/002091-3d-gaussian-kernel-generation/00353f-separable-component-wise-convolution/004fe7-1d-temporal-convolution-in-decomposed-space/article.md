# 1D Temporal Convolution in Decomposed Space

Also called: Convolutional Filtering in Decomposed Space, Difference of Gaussian Filter along Time Dimension

## Overview
This method applies convolutional filtering operations directly within a decomposed tensor space, avoiding the computational cost of reconstructing the full-size dataset [S1, unofficial]. It specifically enables filtering along the time dimension without reconstructing multiple time steps, allowing for efficient detection of temporal changes [S1, unofficial]. It differs from its parent [Separable Component-Wise Convolution](7446-separable-component-wise-convolution.md) by focusing exclusively on the temporal axis within the decomposed representation, and from its sibling [Dr. TVAM Convolution](8425-dr-tvam-convolution.md) by utilizing a Difference of Gaussian kernel rather than other specific temporal smoothing strategies.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of decomposed tensor components representing a 4D volume (3D space + 1D time).
*   **Output:** Filtered tensor components with temporal variations highlighted or smoothed.

**Data Structures:**
The method operates on the low-rank tensor factors. No full 3D grid reconstruction is required during the filtering step [S1, unofficial].

**Steps:**
1.  Identify the temporal dimension within the decomposed tensor structure.
2.  Construct a 1D Difference of Gaussian (DoG) kernel along the time axis [S1, unofficial].
3.  Apply the convolution operation directly to the temporal factors of the decomposed tensors [S1, unofficial].
4.  The result represents areas of major change over time or smoothed temporal data, depending on the DoG parameters [S1, unofficial].

**Formulas:**
The specific mathematical formulation for the DoG kernel is not provided in the sources, but the operation is defined as a 1D convolution along the time dimension [S1, unofficial].

**Parameters:**
*   **Kernel Type:** 1D Difference of Gaussian [S1, unofficial].
*   **Kernel Size:** Not given in the sources.
*   **Performance Metric:** The filter computation takes less than 2ms for the datasets shown in the source [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Operating in decomposed space requires maintaining numerical stability in the tensor factors during convolution [S1, unofficial].
*   **Boundary Conditions:** Standard 1D boundary handling applies to the temporal axis, though specific padding strategies are not detailed [S1, unofficial].

## Subtopics

### VolRec Spatio-Temporal Convolution
This variant applies a custom 3x3x3 convolution kernel to the full-resolution interlaced volume to produce the final output, weighting newer and closer voxels more heavily [S2]. It differs from the pure decomposed temporal method by operating on the reconstructed full-resolution volume and combining spatial and temporal weights [S2]. Choose this when full-resolution spatio-temporal smoothing is required despite higher computational cost [S2].

*   **Definition:** Custom convolution kernel applied to the interlaced volume to produce the final full-resolution volume, weighting newer and closer voxels more heavily [S2].
*   **Steps:**
    1.  Apply spatial weights using a 3D Gaussian distribution, giving more importance to closer voxels [S2].
    2.  Assign temporal weights to every voxel of each A-scan depending on age, giving less relevance to older epochs [S2].
*   **Formula:** Temporal weight $T_w(x, y, \text{epoch})$ is calculated as:
    $$ T_w(x, y, \text{epoch}) = ((y s_x + x) - 1 - (\text{epoch} \mod (s_x s_y))) \mod (s_x s_y) $$
    Where $T_w$ is the temporal weight, $s_x$ and $s_y$ are strides, $x$ and $y$ are A-scan positions, and $\text{epoch}$ is the current epoch number [S2].
*   **Parameters:** Kernel size is 3x3x3 [S2].
*   **Pitfalls:** Time-based weights are dynamic and change in each epoch for each A-scan, causing implementation issues [S2].
*   **Comparison:** Larger kernel sizes did not manage to perform with greater precision in less processing time compared to 3x3x3 [S2].

## Operating characteristics
*   **Time Complexity:** The application of the 1D DoG filter leads to virtually no additional effort, computed in less than 2ms for the datasets shown [S1, unofficial].
*   **Memory Complexity:** Operates in decomposed space, avoiding the memory overhead of full-volume reconstruction during filtering [S1, unofficial].
*   **Strengths:** High efficiency; allows interactive filtering without reconstructing multiple time steps [S1, unofficial].
*   **Limitations:** Specific kernel sizes and precise mathematical parameters for the DoG filter are not provided in the sources [S1, unofficial].
*   **When to use:** When temporal filtering is needed on 4D volumetric data represented by tensor decompositions, and full reconstruction is too expensive [S1, unofficial].

## Background
This method originates from research on Tensor-Train-based 4D Time-Dependent Volume Rendering [S1, unofficial]. It leverages the linearity of tensor decompositions to enable efficient filtering operations that would otherwise require processing the full-size dataset [S1, unofficial]. For more information on basic filtering operations to decomposed tensors, see [BRSP18] (not listed in sources).

## Sources
[S1] Tensor-Train-based 4D Time-Dependent Volume Rendering, https://repositorio.ie.edu/entities/publication/348991b8-1dff-4bea-acbc-433ce708313d/full, fetched 2026-10-29 (unofficial)
[S2] VolRec: 4D real-time volumetric reconstruction of OCT data, https://doi.org/10.1007/s11227-025-06969-6, fetched 2026-10-26
