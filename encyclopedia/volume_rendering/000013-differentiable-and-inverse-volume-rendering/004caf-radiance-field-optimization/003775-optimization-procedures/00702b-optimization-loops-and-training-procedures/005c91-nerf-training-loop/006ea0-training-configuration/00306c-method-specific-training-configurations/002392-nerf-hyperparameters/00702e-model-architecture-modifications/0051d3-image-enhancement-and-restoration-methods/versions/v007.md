# Image Enhancement and Restoration Methods


## Overview
Image Enhancement and Restoration Methods modify the NeRF training pipeline to handle input image defects such as motion blur, defocus, or transient occlusions (e.g., pedestrians, vehicles), enabling the reconstruction of sharp, static scene representations from uncontrolled "in-the-wild" data [S1]. Unlike sibling methods that primarily alter network topology or input space encoding, these methods explicitly model the degradation process or the transient elements as part of the optimization, separating the static scene geometry and appearance from the artifacts [S1].

## Shared principles
The baseline NeRF architecture [Parent](3065-nerf-hyperparameters.md) defines a continuous volumetric scene representation $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$, where $\mathbf{x} \in \mathbb{R}^3$ is the spatial position, $\mathbf{d} \in \mathbb{R}^3$ is the viewing direction, $\sigma$ is the volume density, and $\mathbf{c} \in \mathbb{R}^3$ is the view-dependent color [G]. The rendering process integrates these outputs along rays using standard volume rendering equations [G].

Most modifications in this family retain the core volume rendering equation but alter one or more of the following components:
1.  **Input Space:** Augmenting $\mathbf{x}$ and $\mathbf{d}$ with additional signals such as latent codes $\mathbf{z}$, time $t$, illumination embeddings $\mathbf{z}_l$, or external feature vectors $\mathbf{f}$ from vision models [S1].
2.  **Network Topology:** Replacing the monolithic MLP with specialized branches (e.g., separate density and color networks), modular experts, or hybrid structures combining MLPs with convolutional or transformer backbones [S1].
3.  **Output Space:** Extending the output tuple $(\sigma, \mathbf{c})$ to include additional fields such as surface normals $\mathbf{n}$, semantic logits $\mathbf{s}$, uncertainty variance $\nu$, or blend weights $\Delta w$ [S1].
4.  **Representation Basis:** Substituting the coordinate-based implicit function with explicit structures like voxel grids, hash tables, Gaussian primitives, or planar surfaces, which change how features are stored and interpolated [S1].

## Subtopics

### [DeblurNeRF](11964-deblurnerf.md)
DeblurNeRF reconstructs a sharp NeRF from blurry inputs by modeling blur via a Deformable Sparse Kernel (DSK) that approximates spatially varying blur as a weighted sum of deformed rays. It differs from [IE-NeRF](13737-ie-nerf.md) and [Sharp-NeRF](14270-sharp-nerf.md) by using an MLP-based DSK to predict ray origin translations and kernel deformations, allowing joint optimization of the NeRF and blur parameters. Choose this method when dealing with motion blur or defocus where the point spread function varies spatially and you require a differentiable blur simulation module during training.

### [IE-NeRF](13737-ie-nerf.md)
IE-NeRF separates static scene rendering from transient occlusions using an image-dependent 2D transient mask and a pre-trained inpainting module. Unlike [DeblurNeRF](11964-deblurnerf.md) which addresses motion blur, or [Sharp-NeRF](14270-sharp-nerf.md) which focuses on sharpness, IE-NeRF specifically targets transient object removal in uncontrolled environments. It differs from [NeRF-W](11313-model-architecture-modifications.md) by using a single MLP to predict both static properties and a 2D transient mask, rather than a 3D transient field or dual NeRF pipelines. Choose this method when the input dataset contains dynamic elements like pedestrians or vehicles that must be removed to recover the static scene.

### [Sharp-NeRF](14270-sharp-nerf.md)
Sharp-NeRF is a grid-based framework that renders clean images from blurry inputs by modeling spatially varying blur using discrete learnable kernels and a sharpness prior. Unlike sibling methods that rely on MLPs to generate blur kernels or require extensive training times, Sharp-NeRF achieves faster training by using a decomposed-grid representation and quantized sharpness levels. Choose this method when computational efficiency is critical and the blur is primarily due to defocus or camera shake that can be quantized into a small set of discrete kernels.

#### SPIn-NeRF
SPIn-NeRF is a 3D inpainting method that leverages LaMa for color and depth inpainting, replacing pixel loss with perceptual loss for visual coherence and using depth inpainting to supervise 3D geometry [S1]. A key parameter is the use of perceptual loss instead of pixel loss to enhance visual quality [S1]. A pitfall is that it struggles in tasks where 2D inpainting results are perceptually inconsistent [S1].

## Background
The problem of reconstructing high-quality radiance fields from low-quality or uncontrolled input images has been addressed through various inpainting and deblurring techniques. Early approaches like NeRF-In [132] used simple 2D inpainting networks (e.g., MST) to guide pixel reconstruction. SPIn-NeRF [15] improved upon this by leveraging the LaMa inpainting model [131] for both color and depth, replacing pixel loss with perceptual loss to achieve more visually coherent results and using depth inpainting to supervise 3D geometry [S1]. However, SPIn-NeRF struggles when 2D inpainting results are perceptually inconsistent [S1]. Other methods like Removing-NeRF [133] utilize posed RGB-D images with 2D masks and optimize NeRF with LaMa-inpainted frames, employing confidence-based view selection to exclude inconsistent views [S1]. Reference-guided inpainting [134] performs 2D inpainting on a single reference image and uses monocular depth estimation and a bilateral solver to ensure view-dependent consistency [S1].

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
