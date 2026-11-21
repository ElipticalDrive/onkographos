# GaussianEditor (Local)


## Overview
GaussianEditor is a method tailored for local editing on 3D Gaussian Splatting (3DGS) [S1]. It enables 3D inpainting, object removal, and object composition by controlling 3DGS using text prompts and semantic information derived from Gaussian semantic tracing [S2, unofficial]. Unlike global editing approaches, it focuses on delicately editing within a specific region of interest (RoI) identified in the 3D space [S1].

## How it works
**Inputs:**
- A pre-trained 3DGS scene representation.
- Text instructions or prompts defining the desired edit [S1].
- Semantic information extracted via Gaussian semantic tracing [S2, unofficial].

**Outputs:**
- An edited 3DGS scene with modified attributes (position, scale, rotation, color, opacity) within the target RoI [S1].

**Data Structures:**
The method operates on the standard 3DGS data structure defined in [Gaussian Data Structure](2273-gaussian-data-structure.md). The editing process modifies the attributes of the Gaussians falling within the identified RoI.

**Steps:**
1. **Extract Region of Interest (RoI):** Identify the target area for editing based on the provided text instructions [S1]. This involves mapping the semantic intent to a spatial region in the 3D scene [S2, unofficial].
2. **Align RoI with 3D Gaussians:** Map the identified text-based RoI to the corresponding 3D Gaussians through the image space [S1]. This step ensures that the editing operations are applied to the correct set of Gaussians that project to the target area in the rendered views [S1].
3. **Delicate Editing:** Modify the attributes of the Gaussians within the identified RoI [S1]. This may involve updating positions, scales, rotations, or appearance properties to achieve the desired edit (e.g., inpainting, removal, or composition) [S2, unofficial]. The editing is performed delicately to maintain consistency with the surrounding scene [S1].

**Parameters:**
- Text prompt: Defines the edit content [S1].
- Semantic tracing parameters: Control the accuracy of the RoI identification [S2, unofficial].

**Edge Cases and Pitfalls:**
- Inconsistency across views: Editing in one view may not align with others if the 3D-to-2D mapping is inaccurate [S1].
- Boundary artifacts: Sharp transitions between edited and unedited regions may occur if the Gaussian attributes are not smoothly interpolated [S1].

## Subtopics

### Hierarchical Gaussian Splatting (HGS)
An independent research variant of GaussianEditor that operates similarly to the appearance editing version but introduces hierarchical Gaussian splatting (HGS) for adaptive generative guidance [S2, unofficial]. This allows for more refined 3D editing tasks such as object inpainting by leveraging a hierarchical structure to guide the editing process [S2, unofficial].

## Operating characteristics
- **Efficiency:** Achieves higher editing efficiency compared to earlier NeRF-based editing methods [S1].
- **Quality:** Provides superior rendering quality for edited regions [S1].
- **Capabilities:** Supports complex local edits including inpainting, removal, and composition [S2, unofficial].
- **Limitations:** Relies on the accuracy of semantic tracing and text alignment; may struggle with ambiguous prompts or complex geometric changes [S1][S2, unofficial].

## Background
GaussianEditor was introduced as part of the latest series of work based on 3DGS, aiming to achieve higher editing efficiency and superior rendering quality compared to previous NeRF-based methods [S1]. It was developed to address the need for local, text-guided editing in 3D scenes represented by Gaussian splats [S2, unofficial]. The method has been implemented in variants such as GaussianEditor-iN2N and GaussianEditor-DDS, with GaussianEditor-iN2N showing superior results [S1].

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
