# Lighting-Specific Compositing


## Overview
Lighting-Specific Compositing is a family of volume rendering techniques that integrate illumination models (such as in-scattering, Phong shading, or boundary enhancement) directly into the compositing loop, rather than treating lighting as a post-processing step or a simple per-sample color multiplication. It solves the problem of physically inaccurate or visually flat rendering by allowing light transport effects to influence the accumulation of opacity and color at the sample level. It differs from standard direct volume ray casting by modifying the color and opacity terms ($c_i, \alpha_i$) before they enter the front-to-back accumulator, and from image-level compositing by preserving depth coherence for complex lighting interactions like occlusion and shadowing.

## Shared principles
All methods in this family operate on a sequence of samples $(c_i, \alpha_i)$ encountered along a ray, where $c_i$ is the sample color (RGB) and $\alpha_i$ is the sample opacity (A) [G]. The core mechanism involves an accumulator state $(C_{acc}, \alpha_{acc})$ initialized to $(0, 0)$ (or background color/opacity) and updated iteratively for each sample [G].

The standard optical model assumes that light is emitted or scattered by the volume and attenuated by absorption [G]. The standard front-to-back compositing equation updates these accumulators for each sample $i$ from front to back:
$$
C_{acc}^{(i)} = C_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \alpha_i C_i
$$
$$
\alpha_{acc}^{(i)} = \alpha_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \alpha_i
$$
where $C_{acc}^{(0)} = 0$ and $\alpha_{acc}^{(0)} = 0$ [G]. The term $(1 - \alpha_{acc}^{(i-1)})$ represents the transmittance, i.e., the fraction of light reaching sample $i$ from the camera side [G].

In iterative form using transmittance $T = 1 - \alpha_{acc}$:
$$
\begin{aligned}
C_{final} &\leftarrow C_{final} + c_i \cdot \alpha_i \cdot T \\
T &\leftarrow T \cdot (1 - \alpha_i)
\end{aligned}
$$
Where $T$ is initialized to $1.0$ and represents the accumulated transparency $\prod (1-\alpha_j)$ [G].

The final pixel color is often composited over a background color $C_{bg}$ using $C_{final} = C_{acc} + (1 - \alpha_{acc}) C_{bg}$ [G].

Early ray termination is a common optimization where the loop breaks if $\alpha_{acc}$ exceeds a threshold (e.g., 1.0), as further samples contribute negligibly to the color [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compositing_loop(samples, background_color):
    """
    Standard front-to-back compositing loop shared by the family.
    samples: list of (color: vec3, opacity: float) sorted front-to-back.
    background_color: vec3 for the scene background.
    """
    C_acc = vec3(0.0)
    alpha_acc = 0.0
    
    for c_i, alpha_i in samples:
        transmittance = 1.0 - alpha_acc
        C_acc = C_acc + transmittance * alpha_i * c_i
        alpha_acc = alpha_acc + transmittance * alpha_i
        
        # Early ray termination
        if alpha_acc >= 1.0:
            break
            
    # Composite over background
    return C_acc + (1.0 - alpha_acc) * background_color
```

## Subtopics

### [Incremental Blending for In-scattering](2880-incremental-blending-for-in-scattering.md)
Incremental Blending for In-scattering is a method for approximating light scattering (translucency) in volume rendering by accumulating indirect lighting components across multiple sampling directions. It solves the problem of computing in-scattering in a single-pass rendering pipeline by discretizing the scattering sphere into $J$ directions and blending the result of each direction into a stored accumulator. It differs from full scattering models by ignoring back-scattering during the intensity transport phase, which simplifies the hardware implementation at the cost of accuracy in dense, deep regions of the volume. It builds on [Lighting-Specific Compositing](5940-lighting-specific-compositing.md) by introducing indirect attenuation parameters and a dedicated light buffer for scattering propagation. See [Incremental Blending for In-scattering](2880-incremental-blending-for-in-scattering.md).

### [Intermixing](5139-intermixing.md)
Intermixing is a compositing strategy for multi-channel volume rendering that separately evaluates the illumination model for each data channel at every ray sample before combining the results into the final color and opacity accumulator. It solves the problem of correctly handling occlusions and depth ordering between different data modalities (e.g., CT and MRI) or deformed states of the same object, which image-level compositing fails to do due to missing depth information in intermediate results. It is distinguished from image-level intermixing by its preservation of depth ordering through per-sample combination, and from single-volume rendering by its ability to intermix multiple data sources. See [Intermixing](5139-intermixing.md).

### [Boundary Enhancement](8071-boundary-enhancement.md)
Boundary enhancement is a compositing technique designed to improve the visibility of object boundaries and material interfaces within volume data. It addresses the problem of ambiguous internal structures by emphasizing edges, often used in conjunction with silhouette techniques in medical illustration to clarify anatomical boundaries. Unlike standard direct volume rendering which blends raw transfer function outputs, this method modifies the opacity and color contributions specifically at gradient-heavy regions to sharpen perceived surfaces. It sets itself apart from sibling methods like [Incremental Blending for In-scattering](2880-incremental-blending-for-in-scattering.md) and [Intermixing](5139-intermixing.md) by focusing on geometric edge definition rather than light transport simulation or multi-pass accumulation strategies. See [Boundary Enhancement](8071-boundary-enhancement.md).

### [Phong Shading with Volume Rendering](12805-phong-shading-with-volume-rendering.md)
Phong Shading with Volume Rendering enhances standard direct volume ray casting by applying Phong illumination (diffuse and specular) to volume samples based on computed gradients. It differs from the standard composite shader by computing modified color and opacity terms that encode surface-like shading cues before front-to-back compositing. See [Phong Shading with Volume Rendering](12805-phong-shading-with-volume-rendering.md).

### Shadow Catcher Compositing
- **Definition:** A technique for rendering shadow catchers where the camera sees the background darkened by the ratio of visible to total direct light, with alpha carrying shadow opacity [S1, unofficial].
- **Steps:** Darken the background by the ratio of visible to total direct light. Store shadow opacity in the alpha channel on a transparent background [S1, unofficial].
- **Pitfalls:** Indirect rays must still see the ground to ensure correct global illumination interaction [S1, unofficial].

### Contour Rendering Compositing
- **Definition:** A compositing step for contour rendering that uses Maximum Intensity Projection (MIP) or Local Maximum Intensity Projection (LMIP) to combine ray samples [S2, unofficial].
- **Steps:** 1. Perform classification and shading for all ray samples. 2. Composite using MIP (max intensity) or LMIP to prevent stronger contours from hiding weaker, closer contours [S2, unofficial].

### Light Value Blending
- **Definition:** Blends four light values (a, b, c, d) into a single packed light value, replacing any zero values with the fourth value before averaging [S3, unofficial].
- **Steps:** If a == 0, set a = d. If b == 0, set b = d. If c == 0, set c = d. Return ((a + b + c + d) >> 2) & 0xFF00FF [S3, unofficial].

## Background
The concept of integrating lighting into the compositing loop evolved from early direct volume rendering techniques that relied on simple transfer functions. The specific implementations cited here, such as incremental blending for in-scattering and Phong shading in ray casting, were developed to address the limitations of flat shading and the computational cost of full light transport simulation. Shadow catcher techniques are often associated with physically based rendering pipelines that require accurate shadow integration for global illumination. Contour rendering compositing is used in medical visualization to highlight structural boundaries.

## Sources
[S1] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
[S2] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
[S3] Greedy Meshing - GitHub, https://github.com/programmer1o1/GreedyMeshingMod, fetched 2026-10-27 (unofficial)
