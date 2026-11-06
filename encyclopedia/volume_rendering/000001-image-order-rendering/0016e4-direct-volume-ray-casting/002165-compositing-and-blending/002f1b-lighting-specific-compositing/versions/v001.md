# Lighting-Specific Compositing


## Overview
Lighting-Specific Compositing refers to techniques that integrate lighting calculations (such as in-scattering or shadowing) directly into the volume compositing pipeline, rather than treating lighting as a post-process or separate pass. It solves the problem of efficiently approximating complex light transport effects like volumetric scattering and shadows within the ray-casting accumulation loop. It differs from general [Compositing](2436-compositing.md) by embedding light-specific state (e.g., scattered intensity buffers) into the accumulation logic, and from [Pre-Integration Lookup](2854-pre-integration-lookup.md) by avoiding precomputed 1D/2D lookup tables in favor of dynamic, direction-aware or channel-aware accumulation.

## Shared principles
The family shares the principle of augmenting the standard front-to-back compositing equation with additional state variables that track lighting-specific quantities. In standard compositing, the accumulated color $C$ and opacity $\alpha$ are updated per sample. In lighting-specific compositing, additional buffers or accumulators are maintained to store quantities such as in-scattered intensity or per-channel illumination contributions. These quantities are updated incrementally along the ray, often using weighted averages or specific blending functions that account for the statistical nature of the approximation (e.g., harmonic means for scattering). The final pixel color is derived by combining these lighting-specific accumulators with the standard opacity-weighted color accumulation.

## Subtopics

### [Incremental Blending for In-scattering](2880-incremental-blending-for-in-scattering.md)
A method for approximating volumetric light scattering by accumulating scattered light contributions from multiple directions into a single stored intensity buffer. It differs from [Intermixing](5139-intermixing.md) by targeting the in-scattering term specifically using a harmonic mean update strategy, and from [Incremental Lighting for In-Scattering](3401-incremental-lighting-for-in-scattering.md) by lacking per-channel indirect attenuation. Choose this when efficient single-buffer approximation of in-scattering is required without per-channel complexity. See [Incremental Blending for In-scattering](2880-incremental-blending-for-in-scattering.md).

### [Intermixing](5139-intermixing.md)
A technique for rendering multiple co-registered volume channels simultaneously by evaluating illumination models and combining results at different stages of the ray-casting pipeline. It differs from classification-level intermixing by evaluating illumination per channel rather than interpolating input values first, and from image-level intermixing by preserving depth information during accumulation. Choose this for multi-channel data fusion (e.g., RGB, opacity, scalar fields) where occlusion handling between channels is critical. See [Intermixing](5139-intermixing.md).

### Shadow Catcher Compositing
A technique for rendering shadow catchers where the camera sees the background darkened by the ratio of visible to total direct light, with alpha carrying shadow opacity. It is chosen when physically accurate shadow reception on transparent backgrounds is required for global illumination consistency.
- **Definition:** The background is darkened by the ratio of visible to total direct light; alpha carries the shadow opacity on a transparent background [S1, unofficial].
- **Steps:** Darken the background by the ratio of visible to total direct light. Store shadow opacity in the alpha channel on a transparent background [S1, unofficial].
- **Pitfalls:** Indirect rays must still see the ground to ensure correct global illumination interaction [S1, unofficial].

## Background
The concept of integrating lighting into compositing arises from the need to handle volumetric effects like in-scattering and shadows efficiently in direct volume rendering. Early methods often separated lighting and compositing, leading to inaccuracies or high memory costs. The specific techniques listed here, such as incremental blending for in-scattering and intermixing, have been developed to address these efficiency and accuracy trade-offs in real-time and offline rendering contexts. The shadow catcher technique is often associated with physically based rendering pipelines that require accurate shadow reception for compositing synthetic volumes into real-world backgrounds or for GI baking.

## Sources
[S1] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
