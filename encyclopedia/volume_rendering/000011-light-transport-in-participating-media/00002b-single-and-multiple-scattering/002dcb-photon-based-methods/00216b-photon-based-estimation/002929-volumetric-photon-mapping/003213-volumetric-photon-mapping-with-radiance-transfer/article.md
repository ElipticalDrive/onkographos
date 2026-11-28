# Volumetric Photon Mapping with Radiance Transfer
## Overview
Volumetric Photon Mapping with Radiance Transfer is a rendering technique that integrates volumetric photon mapping into a pre-designed radiance transfer procedure to achieve real-time, realistic volume rendering [S1]. It separates direct and indirect lighting computations and allows effective updates to the photon map when transfer functions change [S1].
## How it works
**Inputs:** Not given in the sources.
**Outputs:** Not given in the sources.
**Data Structures:** A data format for assessing scene areas affected by altering the transfer function to allow effective updates to the photon map [S1]. Standard photon map structures are not detailed in the sources.
**Steps:**
1. Render direct and indirect lighting separately using a raycaster [S1].
2. Incorporate the volumetric photon mapping algorithm into the customary predesigned radiance transfer procedure [S1].
3. Assess scene areas affected by transfer function alterations to enable effective photon map updates [S1].
**Formulas:** Not given in the sources.
**Parameters:** Not given in the sources.
**Edge Cases and Pitfalls:** Not given in the sources.
## Subtopics

### [Photon Mapping for RRTE](16476-photon-mapping-for-rrte.md)
A two-pass rendering algorithm for the Refractive Radiative Transfer Equation that computes curved photon and ray trajectories analytically or numerically to model continuous refraction, addressing standard photon mapping's constant index of refraction assumption. Choose it when rendering participating media with spatially varying refractive indices to produce altered caustic shapes and more accurate reflections.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Designed to produce an interactive character for real-time realistic volume rendering [S1].
- **Strengths:** Real-time capability; handles wide intensity ranges; modular integration into radiance transfer [S1].
- **Limitations:** Dependence on the pre-designed radiance transfer procedure; complexity in updating the photon map when transfer functions change [S1].
- **When to use:** Real-time volume rendering applications where interactive performance is critical [S1].
- **When not to use:** Not given in the sources.
## Background
Described by Zhang et al. [S1]. It incorporates the volumetric photon mapping algorithm into a customary predesigned radiance transfer procedure [S1]. The workflow builds on Daniel et al.'s creation of a data format for assessing scene areas affected by altering transfer functions [S1].
## Sources
[S1] Real-time volume rendering for three-dimensional fetal ultrasound using volumetric photon mapping., https://doi.org/10.1186/s42492-024-00177-4, fetched 2026-10-26
