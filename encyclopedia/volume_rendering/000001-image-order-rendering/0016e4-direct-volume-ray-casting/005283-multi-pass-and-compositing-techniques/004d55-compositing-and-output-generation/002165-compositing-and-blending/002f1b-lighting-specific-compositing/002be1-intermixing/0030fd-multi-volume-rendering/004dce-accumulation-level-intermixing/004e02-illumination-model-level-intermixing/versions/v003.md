# Illumination model level intermixing

Also called: Illumination-level intermixing, Pre-compositing integration

## Overview
Illumination model level intermixing is a multi-modal volume rendering technique that combines ray samples from distinct volume datasets before computing final color and opacity values [S1, unofficial]. It solves the problem of rendering heterogeneous data (e.g., CT and SPECT) by integrating physical properties at the sample level, rather than blending pre-computed visual attributes [S1, unofficial]. Unlike accumulation-level intermixing, which composites individual volume contributions after their respective lighting calculations, this method merges raw data to apply a unified lighting model [S1, unofficial].

## How it works
**Inputs:** Two or more co-registered volume datasets (e.g., $V_1, V_2$) and their associated transfer functions.
**Outputs:** A single composite color and opacity value per ray sample.

**Data Structures:**
*   **Volume Data:** Scalar fields $V_i(x,y,z)$ representing density or intensity.
*   **Transfer Functions:** $T_i(\text{value}) \rightarrow (\text{color}_i, \text{opacity}_i)$.

**Steps:**
1.  **Sample:** For a given ray position $p$, retrieve scalar values $v_1$ from $V_1$ and $v_2$ from $V_2$.
2.  **Integrate:** Combine the raw values using a mixing function. Two primary strategies are defined:
    *   **Weighted Sum:** Compute a weighted density $v_{combined} = w_1 v_1 + w_2 v_2$.
    *   **Role Assignment:** Assign specific physical roles, such as $V_1$ as emission and $V_2$ as attenuation.
3.  **Lookup:** Use $v_{combined}$ to query the transfer function(s) to obtain the final color $C$ and opacity $\alpha$.
4.  **Composite:** Proceed with standard volume rendering integration (e.g., front-to-back compositing) using the unified $C$ and $\alpha$.

**Formulas:**
For weighted sum integration:
$$ v_{combined} = \sum_{i=1}^{n} w_i v_i $$
Where $w_i$ are weights and $\sum w_i = 1$ [S1, unofficial].

For role-based integration (e.g., Emission $E$ and Attenuation $A$):
$$ C = C_E(v_E), \quad \alpha = \alpha_A(v_A) $$
This allows emissive data (like metabolic activity) to be lit by the spatial context of attenuating data (like bone density) [S1, unofficial].

**Parameters:**
*   Weights $w_i$: Tuned per application.
*   Role mapping: Defined by the user based on modality semantics (e.g., CT for structure, SPECT for function) [S1, unofficial].

**Edge Cases:**
*   **Misregistration:** If volumes are not perfectly aligned, the combined value will be spatially incorrect.
*   **Scale Differences:** Raw values from different modalities (e.g., Hounsfield units vs. counts) must be normalized or weighted appropriately before summation [S1, unofficial].

## Subtopics

### [Multimodal Illumination Model](16316-multimodal-illumination-model.md)
Extends the standard integral by summing absorption and scattering across $n$ modalities within the radiance transfer equation. Choose when physically consistent lighting of heterogeneous media is required.

### [Linear Blending Integration](16472-linear-blending-integration.md)
Dynamically blends sample values during rendering using intensity-dependent weights. Choose when flexible, non-redundant integration of anatomical structures is needed.

## Operating characteristics
*   **Complexity:** Time complexity is $O(N)$ per ray, where $N$ is the number of samples. Memory complexity is $O(V)$ for storing multiple volumes.
*   **Performance:** Requires accessing multiple volumes per sample, increasing memory bandwidth usage compared to single-volume rendering.
*   **Strengths:** Physically plausible integration of different data types; avoids artifacts from post-compositing mismatched lighting.
*   **Limitations:** Requires strict co-registration; transfer function design is complex as it must map combined values to meaningful visual attributes.
*   **Use Cases:** Radiation therapy dose planning (emissive beam in attenuating CT); multimodal medical visualization (CT/SPECT) [S1, unofficial].

## Background
Originated in the context of multi-modal medical imaging visualization. Cai and Sakas demonstrated this method for visualizing emissive radiation beams within attenuating CT volumes for dose planning [S1, unofficial]. It addresses the limitations of image-level and accumulation-level intermixing by ensuring that lighting calculations are based on integrated physical properties rather than blended visual outputs.

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
