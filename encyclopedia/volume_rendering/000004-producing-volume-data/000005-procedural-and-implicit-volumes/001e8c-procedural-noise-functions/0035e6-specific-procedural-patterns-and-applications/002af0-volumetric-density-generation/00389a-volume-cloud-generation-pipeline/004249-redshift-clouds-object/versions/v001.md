# Redshift Clouds Object
Also called: Redshift Clouds
## Overview
The Redshift Clouds Object generates entire procedural skies, serving as a dedicated tool for volumetric cloud rendering within the Redshift renderer [S1]. It distinguishes itself from general procedural noise functions by providing a complete sky generation pipeline that supports both a fast, non-volumetric background mode and a fully volumetric mode where geometry (such as skyscrapers or aircraft) can pierce the cloud layer and interact with its lighting [S1]. It operates as a child of the Volume Cloud Generation Pipeline, building upon procedural density generation techniques [S1].

## How it works
The Redshift Clouds Object functions as a specialized volume object that defines the appearance and behavior of procedural cloud layers [S1].

**Inputs and Outputs:**
*   **Input:** Scene context, camera position, and lighting information. The object does not require external voxel data (VDB files) or explicit mesh geometry for the clouds themselves [S1].
*   **Output:** A rendered sky layer that can act as a background or a volumetric medium interacting with scene geometry and light [S1].

**Data Structures:**
The object maintains internal state for the procedural sky generation. Specific internal data structures, bit packing, or memory layouts are not defined in the available sources [S1].

**Steps:**
1.  **Mode Selection:** The user selects between two rendering modes:
    *   **Fast Mode:** Renders the cloud deck as a "clever background." This mode is optimized for performance and does not simulate volumetric scattering for rays passing through the cloud layer [S1].
    *   **Fully Volumetric Mode:** Enables full volumetric rendering. In this mode, the cloud layer is treated as a participating medium [S1].
2.  **Procedural Generation:** The object generates the cloud density and appearance procedurally. No voxel data (VDB) is read or required for this generation process [S1].
3.  **Volumetric Interaction (Volumetric Mode Only):**
    *   If the fully volumetric mode is active, the renderer calculates light transport through the cloud medium [S1].
    *   Geometry such as skyscrapers or aircraft can intersect the cloud layer. The renderer computes occlusion and lighting interactions, allowing the geometry to be lit by the cloud layer itself [S1].
    *   Light shafts (god rays) may be computed if volumetric lighting is enabled in the scene context, though specific algorithmic details for ray marching or scattering integration are not provided in the sources [S1].

**Formulas and Parameters:**
No mathematical formulas, scattering coefficients, or specific parameter values (such as cloud density thresholds or noise frequencies) are provided in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Performance:** The choice between fast background mode and fully volumetric mode significantly impacts rendering time. The fast mode is intended for scenarios where volumetric accuracy is not required [S1].
*   **Geometry Intersection:** In volumetric mode, the interaction between scene geometry and the cloud layer is supported. However, specific handling of degenerate cases or precision issues during intersection is not detailed [S1].

## Subtopics

### Redshift Environment Object
The Redshift Environment Object is used for homogeneous volumes, such as atmospheric haze and fog, which do not require voxel data [S1]. It fills the scene or a height-limited layer of it with a scattering medium [S1]. It is also the designated tool for volumetric lighting, specifically for creating visible god rays (light shafts) in dusty or humid air [S1]. Unlike the Clouds Object, it is not procedural in the same sense but defines a uniform or layered medium [S1].

## Operating characteristics
**Time and Memory Complexity:**
Specific time and memory complexity metrics are not provided in the sources [S1]. The existence of a "fast mode" suggests a trade-off between rendering speed and volumetric accuracy [S1].

**Measured Performance:**
No quantitative performance benchmarks, hardware specifications, or resolution-dependent scaling data are provided in the sources [S1].

**Strengths and Limitations:**
*   **Strengths:** Provides a dedicated, easy-to-use solution for procedural skies without requiring external voxel data or complex noise function setups [S1]. Supports both fast background rendering and fully volumetric interaction with scene geometry [S1].
*   **Limitations:** Lacks the detailed procedural control of general noise functions (as it is a high-level object). Volumetric mode may be computationally expensive compared to the fast background mode [S1].

**When to Use:**
Use the Redshift Clouds Object when generating procedural skies where a quick background representation is sufficient (fast mode) or when full volumetric interaction with geometry and lighting is required for clouds (volumetric mode) [S1]. Use the Redshift Environment Object for uniform atmospheric effects like fog or haze [S1].

## Background
The Redshift Clouds Object is part of the Redshift renderer's volumetric rendering capabilities, introduced as part of the software's support for procedural volumes and sky generation [S1]. It is documented in Maxon's C4D help resources, indicating its integration into the Cinema 4D ecosystem via the Redshift plugin [S1].

## Sources
[S1] Introduction to Volumetric Rendering, https://help.maxon.net/c4d/2027/en-us/Content/html/What_is_Volumetric_Rendering.html, fetched 2026-10-29
