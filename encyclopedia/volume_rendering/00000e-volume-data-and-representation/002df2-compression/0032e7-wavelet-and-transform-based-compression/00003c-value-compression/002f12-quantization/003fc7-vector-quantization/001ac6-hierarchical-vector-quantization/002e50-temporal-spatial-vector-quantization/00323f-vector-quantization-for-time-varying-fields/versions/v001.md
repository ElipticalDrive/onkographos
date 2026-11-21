# Vector Quantization for Time-Varying Fields

Also called: Bernardon et al. Vector Quantization, Vector Quantization for Time-Varying Data, temporal compression, vector quantization approach

## Overview
A compression technique extending vector quantization to encode scalar fields of time-varying unstructured grids based on temporal coherence in a hierarchical data structure [S1, S2, unofficial]. It dynamically decodes the scalar field for each time step, enabling integration with hardware-assisted ray casters or CPU-based rendering systems with minimal performance overhead [S1, S2, unofficial]. It sets itself apart by addressing the specific challenges of unstructured grids and temporal coherence, unlike static compression methods [S1, S2, unofficial].

## How it works
**Inputs:** Time-varying scalar field data on an unstructured grid (typically tetrahedral).
**Outputs:** Compressed bitstream; decoded scalar values for a specific time step.

**Data Structures:**
*   Hierarchical data structure: Stores scalar field values organized to exploit temporal coherence. The exact layout is not specified in the sources [S1, S2, unofficial].
*   Codebook: Implicitly used for vector quantization, but its size and structure are not detailed [S1, S2, unofficial].

**Steps:**
1.  **Encoding:** Encode the scalar field based on temporal coherence within the hierarchical data structure [S1, S2, unofficial]. This involves applying vector quantization to capture temporal patterns across time steps [S1, S2, unofficial].
2.  **Decoding (Dynamic):** For each time step, dynamically decode the scalar field from the compressed representation [S1, S2, unofficial].
3.  **Rendering Integration:**
    *   **GPU Path:** Integrate the decoding into a hardware-assisted ray caster. Decode each time step and update the texture containing scalar values using a fragment shader [S1, S2, unofficial].
    *   **CPU Path:** Alternatively, update scalar values for each time step directly on the CPU within the HAVS (Hardware-Assisted Volume Rendering System) [S1, S2, unofficial].

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   Time-varying topology: The more difficult case of time-varying topology has not been addressed for unstructured grids and remains an open topic of research [S1, S2, unofficial].
*   Brute force rendering: Currently, meshes with time-varying topology are rendered in a brute force manner [S1, S2, unofficial].

## Subtopics

### [Time-Varying Scalar Field (TVSF) Rendering with Vector Quantization (HAVS)](7965-time-varying-scalar-field-tvsf-rendering-with-vector-quantiz.md)
A GPU-based volume rendering method for time-varying scalar fields on unstructured meshes (tetrahedral) that uses Vector Quantization (VQ) to compress scalar data and a paging mechanism to handle arbitrary time steps without rendering stalls. It sets itself apart by incorporating a paging mechanism for arbitrary time steps. Choose it when rendering time-varying scalar fields on unstructured meshes with arbitrary time steps.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The resulting visualizations incur very little performance overhead to rendering the data statically [S1, S2, unofficial].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Minimal performance overhead during rendering; handles time-varying scalar fields on unstructured grids [S1, S2, unofficial].
*   **Limitations:** Does not address time-varying topology; requires hierarchical data structure and temporal coherence [S1, S2, unofficial].
*   **When to use:** When rendering time-varying scalar fields on unstructured grids where temporal coherence exists and minimal rendering overhead is desired [S1, S2, unofficial].
*   **When not to use:** When the grid topology changes over time, as this case is not addressed [S1, S2, unofficial].

## Background
Originated from work extending vector quantization [59] to time-varying unstructured grids, as described in [S1, S2, unofficial]. The approach was presented in a survey of GPU-based volume rendering of unstructured grids [S1, S2, unofficial]. It builds on ideas used in the structured case [S1, S2, unofficial]. The case of time-varying topology remains an open research topic [S1, S2, unofficial].

## Sources
[S1] A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S2] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
