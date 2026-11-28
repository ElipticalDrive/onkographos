# Application-Specific Methods


## Overview
Application-Specific Methods extend the standard NeRF training loop to address domain-specific challenges such as super-resolution, dynamic illumination, synthetic aperture radar (SAR) imaging, and single-view 360° reconstruction. Unlike general-purpose training procedures, these methods modify the optimization objective, scene representation, or input data pipeline to handle constraints like multi-view consistency in super-resolution, disentangling dynamic lighting from geometry, or reconstructing surfaces from sparse radar data.

## Shared principles
The children of this family share the following mechanism: they all operate within the differentiable volume rendering framework, minimizing a loss function between rendered outputs and ground-truth observations. However, they diverge significantly in their specific representations (e.g., surface vs. volume, static vs. dynamic fields) and auxiliary supervision signals (e.g., uncertainty fields, optical flow, semantic features). There is no single shared mathematical formulation or data structure common to all children beyond the basic volume rendering equation.

## Subtopics

### [Cross-Guided Optimization of Radiance Fields (CROP)](10371-cross-guided-optimization-of-radiance-fields-crop.md)
CROP is a framework for multi-view image super-resolution (MVSR) that improves high-resolution novel view synthesis by iteratively refining low-resolution training images during optimization [10371-cross-guided-optimization-of-radiance-fields-crop.md]. It bridges 2D single-image super-resolution (SISR) and 5D radiance fields by fusing SISR features with a voxel-based uncertainty field to ensure multi-view consistency. Unlike methods relying solely on network spectral bias for upscaling, CROP explicitly uses rendering errors to guide image refinement. Choose this method when high-resolution novel views are required from low-resolution inputs and multi-view consistency is critical. [10371-cross-guided-optimization-of-radiance-fields-crop.md]

### RehearsalNeRF
RehearsalNeRF learns disentangled neural fields for scenes with severe illumination changes by leveraging a "rehearsal stage" video captured under stable lighting [12477-rehearsalnerf.md]. It jointly optimizes five neural fields (static objects, dynamic objects, lighting effects, intensity, and appearance) to decouple dynamic lighting from object radiance, addressing the illumination-radiance ambiguity. Unlike methods assuming stable light sources, it handles varying illuminations and dynamic objects. Choose this method for dynamic scenes with changing lighting where disentangled control of illumination and motion is needed. [12477-rehearsalnerf.md]

### [Neural SAR Surface Reconstruction via Inverse Rendering](12668-neural-sar-surface-reconstruction-via-inverse-rendering.md)
This method reconstructs a Digital Surface Model (DSM) and backscattering coefficient map from Synthetic Aperture Radar (SAR) images by minimizing the difference between observed and synthesized SAR intensities [12668-neural-sar-surface-reconstruction-via-inverse-rendering.md]. It models elevation $Z$ and backscattering $B$ within a bounding box using an MLP, reducing sampling degrees of freedom from a 3D volume to a 2D surface. Compared to standard volume rendering, this surface representation is more constrained, leading to faster and easier reconstruction. Choose this method for SAR data reconstruction where surface geometry and backscattering properties are the primary targets. [12668-neural-sar-surface-reconstruction-via-inverse-rendering.md]

### [360FusionNeRF](12755-360fusionnerf.md)
360FusionNeRF is a semi-supervised framework for synthesizing novel views from a single 360° RGB-D panorama [12755-360fusionnerf.md]. It extends OmniNeRF by introducing geometric supervision via auxiliary depth maps and semantic consistency via CLIP features to guide training and reduce occlusion artifacts. Unlike standard NeRF methods relying solely on color reconstruction, it explicitly penalizes geometric and semantic errors. Choose this method when only a single panoramic input is available and geometric/semantic consistency in novel views is required. [12755-360fusionnerf.md]

### Boundary-Constrained Synthesis
A method to synthesize scenes of arbitrary size without artifacts, specifically preventing artifacts at the boundaries resulting from obscured parts of the radiance field [S1]. It avoids artifacts resulting from obscured parts of the radiance field [S1]. A method to reduce artifacts at boundaries in synthesized NeRF scenes by incorporating boundary constraints [S1].
1. Divide radiance field into five parts: upper boundary $B_u$, lower $B_d$, left $B_l$, right $B_r$, and interior $I$. Boundary length is $l_b$ [S1].
2. Place four corner patches from original field onto corresponding corners of synthesized result [S1].
3. Synthesize each of the four boundaries using corresponding boundary patches from input samples (viewed as synthesizing individual strips) [S1].
4. Use synthesized boundaries as constraints for synthesizing the interior region using example patches from interior area $I$ [S1].
Two-phase method fails at boundaries because occluded interior parts are synthesized onto the boundary, exhibiting artifacts due to lack of pixel value supervision during DVGO training [S1].

## Background
The concept of application-specific optimization in NeRF emerged as the base NeRF method [11368-scene-representation-and-architecture.md] proved insufficient for specific domains like super-resolution, dynamic lighting, or non-photographic sensors. CROP [10371-cross-guided-optimization-of-radiance-fields-crop.md] addresses the limitation of coordinate-based networks in high-frequency detail synthesis. RehearsalNeRF [12477-rehearsalnerf.md] tackles the ill-posed nature of disentangling illumination from geometry in dynamic scenes. Neural SAR [12668-neural-sar-surface-reconstruction-via-inverse-rendering.md] adapts the framework to inverse rendering for radar data. 360FusionNeRF [12755-360fusionnerf.md] extends the approach to single-view 360° inputs using auxiliary semantic and geometric cues. Boundary-Constrained Synthesis [S1] addresses artifacts in large-scale scene synthesis.

## Sources
[S1] NeRF Synthesis with Shading Guidance, https://link.springer.com/chapter/10.1007/978-981-99-9666-7_16, fetched 2026-10-26
