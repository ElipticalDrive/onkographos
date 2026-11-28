# Lighting-Specific Compositing
## Overview
Lighting-Specific Compositing integrates lighting calculations (e.g., in-scattering, shadows) directly into the volume compositing pipeline rather than treating them as post-processes [G]. It solves the problem of efficiently approximating complex light transport effects within the ray-casting accumulation loop [G]. It differs from general [Compositing](2436-compositing.md) by embedding light-specific state into the accumulation logic, and from [Pre-Integration Lookup](2854-pre-integration-lookup.md) by avoiding precomputed lookup tables in favor of dynamic accumulation [G].
## Shared principles
The family augments the standard front-to-back compositing equation (see [Compositing and Blending](2515-compositing-and-blending.md)) with additional state variables that track lighting-specific quantities [G]. In lighting-specific compositing, extra buffers or accumulators are maintained to store quantities such as in-scattered intensity or per-channel illumination contributions [G]. These quantities are updated incrementally along the ray, often using weighted averages or specific blending functions that account for the statistical nature of the approximation [G]. The final pixel color is derived by combining these lighting-specific accumulators with the standard opacity-weighted color accumulation [G].
## Subtopics

### [Incremental Blending for In-scattering](2880-incremental-blending-for-in-scattering.md)
A method for approximating volumetric light scattering by accumulating scattered light contributions from multiple directions into a single stored intensity buffer. It differs from [Intermixing](5139-intermixing.md) by targeting the in-scattering term specifically using a harmonic mean update strategy, and from [Incremental Lighting for In-Scattering](3401-incremental-lighting-for-in-scattering.md) by lacking per-channel indirect attenuation. Choose this when efficient single-buffer approximation of in-scattering is required without per-channel complexity. See [Incremental Blending for In-scattering](2880-incremental-blending-for-in-scattering.md).

### [Intermixing](5139-intermixing.md)
A technique for rendering multiple co-registered volume channels simultaneously by evaluating illumination models and combining results at different stages of the ray-casting pipeline. It differs from classification-level intermixing by evaluating illumination per channel rather than interpolating input values first, and from image-level intermixing by preserving depth information during accumulation. Choose this for multi-channel data fusion where occlusion handling between channels is critical. See [Intermixing](5139-intermixing.md).

### Shadow Catcher Compositing
- **Definition:** A technique for rendering shadow catchers where the camera sees the background darkened by the ratio of visible to total direct light, with alpha carrying shadow opacity [S1, unofficial].
- **Steps:** Darken the background by the ratio of visible to total direct light. Store shadow opacity in the alpha channel on a transparent background [S1, unofficial].
- **Pitfalls:** Indirect rays must still see the ground to ensure correct global illumination interaction [S1, unofficial].

### Contour Rendering Compositing
- **Definition:** A compositing step for contour rendering that uses Maximum Intensity Projection (MIP) or Local Maximum Intensity Projection (LMIP) to combine ray samples [S2, unofficial].
- **Steps:** Perform classification and shading for all ray samples. Composite using MIP (max intensity) or LMIP to prevent stronger contours from hiding weaker, closer contours [S2, unofficial].

## Background
The concept of integrating lighting into compositing arises from the need to handle volumetric effects like in-scattering and shadows efficiently in direct volume rendering [G]. Early methods often separated lighting and compositing, leading to inaccuracies or high memory costs [G]. The specific techniques listed here have been developed to address these efficiency and accuracy trade-offs in real-time and offline rendering contexts [G]. The shadow catcher technique is often associated with physically based rendering pipelines that require accurate shadow reception for compositing synthetic volumes into real-world backgrounds or for GI baking [G].
## Sources
[S1] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
[S2] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
