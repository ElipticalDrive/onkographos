# Application-Specific Methods


## Overview
Application-Specific Methods are optimization procedures that adapt the standard NeRF training loop to solve domain-specific problems, such as handling dynamic lighting, reconstructing from non-photographic sensors (SAR), synthesizing from single views, or modeling complex physical phenomena like mirror reflections. Unlike general-purpose training loops that assume static scenes and stable illumination, these methods introduce specialized neural fields, loss functions, and sampling strategies to address ill-posed inverse problems or specific physical constraints. They are distinguished from siblings like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by their focus on *what* is being optimized (e.g., disentangled lighting, surface elevation) rather than *how* rays are integrated, and from [Regularization and Loss Functions](8586-regularization-and-loss-functions.md) by their integration of these losses into a broader application-specific pipeline.

## Shared principles
The children of this family share the following mechanisms and mathematical foundations:

1.  **Specialized Neural Field Architectures**: Standard NeRF uses a single MLP mapping $(x, y, z, \theta, \phi) \to (\sigma, c)$. Application-specific methods often decompose this into multiple MLPs or modify the input/output space:
    *   **Decoupling**: Fields are split to isolate specific physical properties (e.g., static vs. dynamic geometry, illumination vs. radiance) [S1, 12477-rehearsalnerf.md].
    *   **Dimensionality Reduction**: Some methods map 2D coordinates to 3D properties (e.g., elevation and backscattering from $(x,y)$) to enforce surface constraints [S1, 12668-neural-sar-surface-reconstruction-via-inverse-rendering.md].
    *   **Canonical Warping**: Coordinates are transformed into a canonical space (e.g., via 3DMM) before querying the MLP to handle non-rigid or specific subject geometries [S1, 13429-portrait-nerf.md].

2.  **Domain-Specific Loss Functions**: The optimization objective $L$ is extended beyond photometric reconstruction error ($L_{photo}$) to include constraints derived from the application domain:
    *   **Geometric/Depth Supervision**: Using auxiliary depth maps or point clouds to constrain geometry [S1, 12755-360fusionnerf.md].
    *   **Semantic Consistency**: Using high-level features (e.g., CLIP) to ensure plausible content in unobserved regions [S1, 12755-360fusionnerf.md].
    *   **Physical Priors**: Enforcing symmetry, planarity, or specific illumination models (e.g., HSV decomposition) [S1, 13482-mirror-nerf.md, 12477-rehearsalnerf.md].
    *   **Statistical Models**: Using Negative Log-Likelihood (NLL) for speckle noise in SAR data instead of MSE [S1, 12668-neural-sar-surface-reconstruction-via-inverse-rendering.md].

3.  **Iterative or Multi-Stage Optimization**: Many methods do not optimize a single static field directly. Instead, they employ iterative loops where the output of one stage (e.g., super-resolved images, uncertainty maps, or rough meshes) informs the next stage of optimization [S1, 10371-cross-guided-optimization-of-radiance-fields-crop.md, 13429-portrait-nerf.md].

4.  **Volume Rendering Equation**: All methods ultimately rely on the standard volume rendering integral to synthesize images from the learned fields:
    $$ \hat{C}(r) = \sum_{i=1}^{N} T_i \alpha_i c_i = \sum_{i=1}^{N} T_i (1 - e^{-\sigma_i \delta_i}) c_i $$
    where $T_i = \prod_{j=1}^{i-1} e^{-\sigma_j \delta_j}$ is the accumulated transmittance, $\sigma_i$ is density, $c_i$ is color, and $\delta_i$ is the interval length [S1, 10371-cross-guided-optimization-of-radiance-fields-crop.md, 12755-360fusionnerf.md, 13482-mirror-nerf.md].

## Subtopics

### [Cross-Guided Optimization of Radiance Fields (CROP)](10371-cross-guided-optimization-of-radiance-fields-crop.md)
CROP is a framework for high-resolution novel view synthesis (HRNVS) that performs multi-view image super-resolution (MVSR) on training images *during* the radiance field optimization process. It improves upon standard NeRF by fusing single-image super-resolution (SISR) features with a voxel-based uncertainty field derived from rendering errors, ensuring multi-view consistency and high-frequency detail recovery. Unlike methods that rely solely on the spectral characteristics of coordinate-based networks for upscaling, CROP explicitly bridges 2D SISR and 5D radiance fields through an iterative update loop. It is chosen when high-resolution output is required from low-resolution inputs and multi-view consistency is critical.

### [RehearsalNeRF](12477-rehearsalnerf.md)
RehearsalNeRF is a method for learning disentangled neural fields under severe illumination changes by leveraging scenes captured under stable lighting (rehearsal stages) to enforce geometric consistency between different lighting conditions. It jointly optimizes five neural fields to decouple dynamic lighting from moving and static objects, addressing the "illumination-radiance ambiguity" where dynamic illumination radiance is entangled with object radiance. Unlike siblings that assume stable light sources or only handle subject motion, RehearsalNeRF explicitly handles varying illuminations and enables applications such as controlling lighting while stopping motion.

### [Neural SAR Surface Reconstruction via Inverse Rendering](12668-neural-sar-surface-reconstruction-via-inverse-rendering.md)
This method reconstructs a Digital Surface Model (DSM) and a backscattering coefficient map from Synthetic Aperture Radar (SAR) images by minimizing the difference between observed SAR intensities and synthesized intensities using a differentiable rendering model. It models elevation $Z$ and backscattering $B$ within a bounding box using a Multi-Layer Perceptron (MLP), reducing sampling degrees of freedom from 3D volume to 2D surface. Unlike volume-based approaches, this surface representation is more constrained, leading to faster and easier reconstruction of surfaces. It is chosen for remote sensing applications where SAR data is the primary input.

### [360FusionNeRF](12755-360fusionnerf.md)
360FusionNeRF is a semi-supervised learning framework for synthesizing novel views from a single RGB-D 360° panorama image. It extends OmniNeRF by introducing geometric supervision via auxiliary depth maps and semantic consistency via CLIP features to guide progressive training and reduce artifacts caused by occlusion in unobserved regions. Unlike standard NeRF methods that rely solely on color reconstruction, 360FusionNeRF explicitly penalizes geometric errors and semantic inconsistencies to ensure plausible completions of missing scene parts.

### [Portrait-NeRF](13429-portrait-nerf.md)
Portrait-NeRF is a method for estimating Neural Radiance Fields (NeRF) from a single headshot portrait image. It addresses the fundamental ill-posedness of training a coordinate-based MLP from a single view by pretraining the network weights using a meta-learning framework on a light stage dataset, followed by test-time finetuning on the input image. Unlike standard NeRF which requires tens to hundreds of views, or 3D model-based methods that often exclude hair and torso, Portrait-NeRF reconstructs the full head region (including hair and upper torso) using an implicit neural representation. It is distinguished from sibling methods like [CROP](10371-cross-guided-optimization-of-radiance-fields-crop.md) and [Mirror-NeRF](13482-mirror-nerf.md) by its reliance on meta-learning initialization and canonical coordinate warping via a 3D Morphable Model (3DMM) rather than multi-view consistency or explicit surface constraints.

### [Mirror-NeRF](13482-mirror-nerf.md)
Mirror-NeRF is a neural rendering framework that learns accurate mirror geometry and reflection by unifying the radiance field and tracing rays according to Whitted-style light transport, enabling novel view synthesis and scene manipulation. Unlike NeRF and NeRFReN, which treat mirror reflections as separate virtual scenes or interpolate memorized reflections, Mirror-NeRF uses a unified radiance field with explicit reflection probability and Whitted-style ray tracing to synthesize physically consistent reflections from novel viewpoints.

#### RapNeRF
RapNeRF is a NeRF model tailored for view extrapolation, contrasting with standard NeRFs that excel at interpolation [S1]. It uses Random Ray Casting (RRC) and a Ray Atlas (RA) [S1]. Training begins with a baseline NeRF to recover a rough mesh [S1]. A Ray Atlas (RA) is created by extracting a rough 3D mesh from a pretrained NeRF and mapping training ray directions onto its vertices [S1]. Random Ray Casting (RRC) is employed: for a training ray hitting a surface point $v = o + tz d$, a backward ray is cast from $v$ toward a new origin $o'$ with uniformly sampled angular perturbations [S1]. RRC and RA augment training rays with a predetermined probability [S1].

#### SNeS
SNeS is a method improving geometry by learning probable symmetries for partly symmetric and partly hidden in-scene objects through soft symmetry constraints on geometry and material properties [S1].

#### Boundary-Constrained Synthesis
Boundary-Constrained Synthesis is a method to synthesize scenes of arbitrary size without artifacts, specifically preventing artifacts at the boundaries resulting from obscured parts of the radiance field [S2]. It avoids artifacts resulting from obscured parts of the radiance field [S2]. The steps are: 1. Divide radiance field into five parts: upper boundary $B_u$, lower $B_d$, left $B_l$, right $B_r$, and interior $I$. Boundary length is $l_b$. 2. Place four corner patches from original field onto corresponding corners of synthesized result. 3. Synthesize each of the four boundaries using corresponding boundary patches from input samples (viewed as synthesizing individual strips). 4. Use synthesized boundaries as constraints for synthesizing the interior region using example patches from interior area $I$ [S2]. A pitfall is that the two-phase method fails at boundaries because occluded interior parts are synthesized onto the boundary, exhibiting artifacts due to lack of pixel value supervision during DVGO training [S2].

## Background
The concept of application-specific optimization in NeRF emerged as the basic NeRF framework [S1] proved insufficient for complex real-world scenarios such as dynamic lighting, single-view reconstruction, and non-visible spectrum data. Early extensions focused on improving geometry via depth supervision [S1]. Subsequent works like RapNeRF (May 2022) addressed view extrapolation limitations [S1], while SNeS (June 2022) introduced symmetry constraints [S1]. More recent developments include RehearsalNeRF for dynamic illumination [S1], CROP for super-resolution [S1], and specialized pipelines for SAR [S1], portraits [S1], and mirrors [S1]. These methods represent a shift from generic volume rendering to physically or semantically constrained inverse rendering.

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] NeRF Synthesis with Shading Guidance, https://link.springer.com/chapter/10.1007/978-981-99-9666-7_16, fetched 2026-10-26
