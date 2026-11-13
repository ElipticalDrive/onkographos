# Redshift Clouds Object
Also called: Redshift Clouds
## Overview
The Redshift Clouds Object generates entire procedural skies, distinguishing itself from general noise functions by providing a complete sky generation pipeline that supports both a fast, non-volumetric background mode and a fully volumetric mode where geometry (such as skyscrapers or aircraft) can pierce the cloud layer and interact with its lighting [S1]. It builds on the Volume Cloud Generation Pipeline [8286-volume-cloud-generation-pipeline.md].

## How it works
The Redshift Clouds Object functions as a specialized volume object that defines the appearance and behavior of procedural cloud layers [S1].

**Inputs and Outputs:**
*   **Input:** Scene context, camera position, and lighting information. The object does not require external voxel data (VDB files) or explicit mesh geometry for the clouds themselves [S1].
*   **Output:** A rendered sky layer that can act as a background or a volumetric medium interacting with scene geometry and light [S1].

**Data Structures:**
Not given in the sources.

**Steps:**
1.  **Mode Selection:** The user selects between two rendering modes:
    *   **Fast Mode:** Renders the cloud deck as a "clever background" [S1].
    *   **Fully Volumetric Mode:** Treats the cloud layer as a participating medium [S1].
2.  **Procedural Generation:** The object generates the cloud density and appearance procedurally without reading voxel data [S1].
3.  **Volumetric Interaction (Volumetric Mode Only):**
    *   The renderer calculates light transport through the cloud medium [S1].
    *   Geometry such as skyscrapers or aircraft can intersect the cloud layer, allowing the geometry to be lit by the cloud layer itself [S1].

**Formulas and Parameters:**
No mathematical formulas, scattering coefficients, or specific parameter values are provided in the sources.

**Edge Cases and Pitfalls:**
The choice between fast background mode and fully volumetric mode significantly impacts rendering time, with the fast mode intended for scenarios where volumetric accuracy is not required [S1].

## Subtopics

### Redshift Environment Object
The Redshift Environment Object is used for homogeneous volumes, such as atmospheric haze and fog, which do not require voxel data [S1]. It fills the scene or a height-limited layer of it with a scattering medium [S1]. It is also the designated tool for volumetric lighting, specifically for creating visible god rays (light shafts) in dusty or humid air [S1]. Unlike the Clouds Object, it defines a uniform or layered medium rather than a procedural sky, and should be chosen when simulating atmospheric haze or volumetric lighting rather than distinct cloud formations [S1].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources. The existence of a "fast mode" suggests a trade-off between rendering speed and volumetric accuracy [S1].

**Measured Performance:**
No quantitative performance benchmarks, hardware specifications, or resolution-dependent scaling data are provided in the sources.

**Strengths and Limitations:**
*   **Strengths:** Provides a dedicated solution for procedural skies without requiring external voxel data or complex noise function setups [S1]. Supports both fast background rendering and fully volumetric interaction with scene geometry [S1].
*   **Limitations:** Volumetric mode may be computationally expensive compared to the fast background mode [S1].

**When to Use:**
Use the Redshift Clouds Object when generating procedural skies where a quick background representation is sufficient (fast mode) or when full volumetric interaction with geometry and lighting is required (volumetric mode) [S1]. Use the Redshift Environment Object for uniform atmospheric effects like fog or haze [S1].

## Background
The Redshift Clouds Object is documented in Maxon's Cinema 4D help resources as part of the software's volumetric rendering tools [S1].

## Sources
[S1] Introduction to Volumetric Rendering, https://help.maxon.net/c4d/2027/en-us/Content/html/What_is_Volumetric_Rendering.html, fetched 2026-10-29
