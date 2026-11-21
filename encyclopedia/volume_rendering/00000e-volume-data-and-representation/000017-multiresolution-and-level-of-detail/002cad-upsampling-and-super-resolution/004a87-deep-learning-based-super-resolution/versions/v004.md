# Deep Learning-Based Super-Resolution
## Overview
This family uses deep neural networks to reconstruct high-resolution volumetric data or isosurface renderings from low-resolution inputs, addressing the loss of detail and artifacts inherent in traditional interpolation [G]. It distinguishes itself from sibling methods by leveraging learned spatial priors and non-linear mappings rather than fixed mathematical kernels, enabling significant upscaling factors and artifact suppression that simple upsampling cannot achieve [G].
## Shared principles
Shared principles are detailed in [Upsampling and Super-Resolution](5339-upsampling-and-super-resolution.md) [G].
## Subtopics

### [3D Super-Resolution (SR) Module](6577-3d-super-resolution-sr-module.md)
A 3D CNN integrated with an Attention Mechanism designed to reconstruct high-resolution volumetric data from low-resolution inputs, often serving as the final refinement stage in Implicit Neural Representation (INR) pipelines [6577-3d-super-resolution-sr-module.md]. It addresses reconstruction artifacts like ringing and blurring by prioritizing significant features, providing intrinsic denoising [6577-3d-super-resolution-sr-module.md]. Choose this when refining INR outputs into discrete voxel grids.

### [Contrastive Diffusion for Volume Super-Resolution (CD-TVD)](10411-contrastive-diffusion-for-volume-super-resolution-cd-tvd.md)
The first diffusion-based approach for scientific volume super-resolution, using a 3D Swin Transformer denoising network in a latent space [10411-contrastive-diffusion-for-volume-super-resolution-cd-tvd.md]. It combines contrastive pre-training on historical data with fine-tuning from a single high-resolution timestep [10411-contrastive-diffusion-for-volume-super-resolution-cd-tvd.md]. Choose this for scientific fluid dynamics data where domain-specific degradation-aware features are critical and few high-resolution examples are available.

### [Deep Learning-Based Super-Resolution for Volumetric Isosurface Rendering](12746-deep-learning-based-super-resolution-for-volumetric-isosurfa.md)
A method that reduces computational cost by replacing high-resolution sampling with a low-resolution geometric pass followed by a Fully Convolutional Neural Network (FCNN) inference [12746-deep-learning-based-super-resolution-for-volumetric-isosurfa.md]. It infers high-resolution normals, depth, and ambient occlusion (AO) maps from low-resolution inputs, avoiding runtime AO ray-tracing [12746-deep-learning-based-super-resolution-for-volumetric-isosurfa.md]. Choose this for real-time volumetric isosurface rendering where AO simulation is too expensive.

### [Deep Learning-Based Super-Resolution for Volumetric Isosurfaces](12747-deep-learning-based-super-resolution-for-volumetric-isosurfa.md)
Upsamples low-resolution volumetric isosurface renderings to high resolution using a trained FCNN (SRNet) that leverages temporal coherence via screen-space flow warping [12747-deep-learning-based-super-resolution-for-volumetric-isosurfa.md]. It predicts intermediate surface attributes (mask, normal, depth, AO) rather than direct color to prevent color bleeding [12747-deep-learning-based-super-resolution-for-volumetric-isosurfa.md]. Choose this for interactive navigation of isosurfaces where temporal stability and shading fidelity are required.

### SRCNN for Volume Super-Resolution
Zhou et al. introduced the first CNN-based approach to volume super-resolution, adapting the three-layer SRCNN architecture to operate on 3D patches extracted from medical CT volumes [S1, unofficial]. Trained with MSE loss at 2× upscaling, the method achieved 41.16 dB PSNR, approximately 4 dB above trilinear interpolation [S1, unofficial]. The method required separate models per imaging modality and was limited to small scale factors [S1, unofficial].

### Deep Learning Super Sampling (DLSS)
A technique that renders games at reduced resolution, upscales them with transformer models, and generates additional frames [S2]. It generates up to five frames for every frame the game renders on GeForce RTX 50 Series GPUs [S2].

### [Blur-Free Inconsistency Modeling](17175-blur-free-inconsistency-modeling.md)
A scheme to mitigate multi-view inconsistency from single-image super-resolution models by using a learnable Inconsistency Modeling (IM) module and a Blur Proposal (BP) module to constrain the loss [G]. It applies a pre-trained 2D super-resolution model (ISR) to low-resolution views, passes the output through the IM module (two residual blocks) to simulate inconsistency, and uses the BP module (a four-layer convolutional network) to predict per-pixel blur kernels $B_k$ from the detached high-resolution rendered image [G]. The blur kernel size $K$ is set to 5 for 4× super-resolution tasks, and the weight for the D-SSIM loss term $\beta$ is set to 0.2 [G].

## Background
The field began with Zhou et al., who introduced the first CNN-based approach to volume super-resolution by adapting the SRCNN architecture to 3D patches from medical CT volumes [S1, unofficial]. Subsequent developments included GAN-based methods like SSRTVD for time-varying data, which used dual discriminators for spatial and temporal coherence [S1, unofficial]. Diffusion-based approaches, such as CD-TVD, later extended the family to scientific volumes using contrastive pre-training [S1, unofficial].

## Sources
[S1] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S2] Neural Rendering: Graphics Learned From Images, https://www.ultralytics.com/glossary/neural-rendering, fetched 2026-10-27
