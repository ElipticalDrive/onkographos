# Lighting-Specific Compositing


## Overview
Lighting-Specific Compositing refers to a family of volume rendering techniques that modify the standard front-to-back compositing equation to incorporate complex lighting effects, such as in-scattering, multi-channel illumination, geometric boundary enhancement, and Phong shading, directly into the accumulation step [S1, unofficial]. Unlike standard compositing which treats samples as simple opaque/translucent entities, these methods compute lighting contributions ($\Delta C_i, \Delta \alpha_i$) per sample or per channel before blending them into the accumulator [S1, unofficial]. This family is distinct from [Pre-Integration Lookup](2854-pre-integration-lookup.md) (which approximates integration via texture lookups) and [Binary Swap Compositing](3264-binary-swap-compositing.md) (which focuses on parallel reduction structures), as it specifically addresses the *illumination model* within the compositing loop.

## Shared principles
All methods in this family operate on the standard ray-casting accumulator state $(C_{acc}, \alpha_{acc})$, initialized to $(0, 0)$ [S1, S51, unofficial]. The core mechanism replaces the standard sample color $c_i$ and opacity $\alpha_i$ with lighting-modified contributions $\Delta C_i$ and $\Delta \alpha_i$.

The standard optical model assumes light is emitted or scattered by the volume and attenuated by absorption [S1, unofficial]. The update step for each sample $i$ becomes:
$$
C_{acc}^{(i)} = C_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \cdot \Delta C_i
$$
$$
\alpha_{acc}^{(i)} = \alpha_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \cdot \Delta \alpha_i
$$
where $\Delta \alpha_i$ is the effective opacity contribution and $\Delta C_i$ is the effective color contribution, both derived from specific lighting models [S1, unofficial].

In iterative form using transmittance $T = 1 - \alpha_{acc}$:
$$
\begin{aligned}
C_{final} &\leftarrow C_{final} + \Delta C_i \cdot T \\
T &\leftarrow T \cdot (1 - \Delta \alpha_i)
\end{aligned}
$$
Where $T$ is initialized to $1.0$ [G].

The final pixel color is composited over a background $C_{bg}$:
$$
C_{final} = C_{acc} + (1 - \alpha_{acc}) C_{bg}
$$
[S1, S14, unofficial].

Early ray termination remains applicable: the loop breaks if $\alpha_{acc}$ exceeds a threshold (e.g., 0.99 or 1.0) [S10, S12, unofficial].

## Subtopics

### [Incremental Blending for In-scattering](2880-incremental-blending-for-in-scattering.md)
Approximates volumetric light scattering by accumulating scattered light from discrete directions. It sets itself apart by treating the scattering integral as a sum of directional components blended incrementally, allowing interactive updates. Choose this when global illumination effects are needed without full path tracing costs.

### [Intermixing](5139-intermixing.md)
Renders multi-channel volume data by evaluating illumination per channel and combining results during accumulation. It handles inter-channel occlusions naturally, unlike image-level intermixing. Choose this for visualizing complex interactions between distinct data sources (e.g., tissue types).

### [Boundary Enhancement](8071-boundary-enhancement.md)
Improves boundary visibility by modulating opacity based on geometric cues (silhouettes). It suppresses opacity when the view vector aligns with the surface normal, creating dark outlines. Choose this to enhance edge definition in volumes.

### [Phong Shading with Volume Rendering](12805-phong-shading-with-volume-rendering.md)
Applies Phong lighting (diffuse/specular) to volume samples using gradient-based normals. It adds surface-like shading to volumes, unlike standard compositing which lacks geometric shading. Choose this for surface-like appearance in volumetric data.

### Shadow Catcher Compositing
- **Definition**: A technique for rendering shadow catchers where the camera sees the background darkened by the ratio of visible to total direct light, with alpha carrying shadow opacity [S1, unofficial].
- **Step**: Darken the background by the ratio of visible to total direct light. Store shadow opacity in the alpha channel on a transparent background [S1, unofficial].
- **Pitfall**: Indirect rays must still see the ground to ensure correct global illumination interaction [S1, unofficial].

### Contour Rendering Compositing
- **Definition**: A compositing step for contour rendering that uses Maximum Intensity Projection (MIP) or Local Maximum Intensity Projection (LMIP) to combine ray samples [S2, unofficial].
- **Step**: Perform classification and shading for all ray samples. Composite using MIP (max intensity) or LMIP to prevent stronger contours from hiding weaker, closer contours [S2, unofficial].

### Light Value Blending
- **Definition**: Blends four light values (a, b, c, d) into a single packed light value, replacing any zero values with the fourth value before averaging [S3, unofficial].
- **Step**: If a == 0, set a = d. If b == 0, set b = d. If c == 0, set c = d. Return ((a + b + c + d) >> 2) & 0xFF00FF [S3, unofficial].

## Background
The concept of integrating lighting models into the compositing loop originates from the need to render physically plausible effects in participating media without the cost of full path tracing. Early work on importance-driven volume rendering highlighted the separation of classification, shading, and compositing [S2, unofficial]. The specific techniques listed here, such as incremental blending for in-scattering and Phong shading in volumes, have been developed in various graphics engines and research projects to provide interactive feedback for complex lighting scenarios [S1, unofficial]. Shadow catcher compositing is often used in architectural and VFX pipelines to composite CG elements into live-action footage with accurate shadow interactions [S1, unofficial].

## Sources
[S1] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
[S2] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
[S3] Greedy Meshing - GitHub, https://github.com/programmer1o1/GreedyMeshingMod, fetched 2026-10-27 (unofficial)
