# Shape Attribute

Also called: Forma

## Overview
Shape Attribute is a feature-based classification operator designed for segmented volumes to discriminate objects based on their geometric topology rather than just intensity or size [S1, unofficial]. It evaluates three specific shape aspects—tubiness (cylindrical), surfaceness (planar), and blobbiness (spherical)—to generate a transfer function that maps these geometric properties to visual attributes [S1, unofficial]. Unlike general intensity-based transfer functions, it requires pre-segmented data to isolate the object of interest before computing local shape descriptors [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A segmented volume dataset where each voxel is assigned to a specific object or region of interest.
*   **Output:** A scalar value or vector representing the degree of tubiness, surfaceness, and blobbiness for each voxel, which can be mapped to color and opacity in a transfer function.

**Data Structures:**
The method operates on a 3D grid of voxels. For each voxel $v$ within a segmented object, the algorithm computes local geometric features. The specific mathematical formulation for deriving the tubiness, surfaceness, and blobbiness values from the segmented volume is not provided in the source [S1, unofficial]. The source indicates that the operator is "prepared on segmented volumes," implying a preprocessing step where the volume is partitioned into distinct objects [S1, unofficial].

**Steps:**
1.  **Segmentation:** The input volume is segmented to isolate the target object(s). This step is a prerequisite for the Shape Attribute operator [S1, unofficial].
2.  **Local Analysis:** For each voxel within the segmented object, the local neighborhood is analyzed to determine its geometric character [S1, unofficial].
3.  **Shape Discrimination:** The operator calculates three metrics:
    *   **Tubiness:** Measures the cylindrical aspect of the local structure [S1, unofficial].
    *   **Surfaceness:** Measures the planar aspect of the local structure [S1, unofficial].
    *   **Blobbiness:** Measures the bubble-like (spherical) aspect of the local structure [S1, unofficial].
4.  **Mapping:** The resulting shape values are used to define a transfer function, allowing the user to visualize specific shape classes (e.g., highlighting tubular structures like blood vessels) [S1, unofficial].

**Parameters:**
*   **Segmentation Thresholds:** Not given in the sources.
*   **Neighborhood Size:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Boundary Effects:** Voxels at the boundary of a segmented object may have ambiguous shape characteristics due to incomplete neighborhood data [G].
*   **Noise:** Noise in the segmentation can lead to incorrect shape classification, particularly for thin structures where tubiness might be misclassified as surfaceness or blobbiness [G].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Allows for intuitive selection of shape classes (tubular, planar, spherical) in segmented volumes, enhancing the visualization of specific anatomical or structural features [S1, unofficial].
*   **Limitations:** Requires pre-segmented data, which adds a preprocessing step and potential error source [S1, unofficial]. The specific computational cost of the shape discrimination is not detailed [S1, unofficial].

## Background
The Shape Attribute operator (also known as Forma) was proposed by Praßni et al. in 2010 [S1, unofficial]. It is part of a broader effort to use pyramid schemes for categorizing objects within volumes [S1, unofficial]. The method is often compared with size-based and intensity-based transfer functions, offering a distinct geometric perspective for volume visualization [S1, unofficial].

## Sources
[S1] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
