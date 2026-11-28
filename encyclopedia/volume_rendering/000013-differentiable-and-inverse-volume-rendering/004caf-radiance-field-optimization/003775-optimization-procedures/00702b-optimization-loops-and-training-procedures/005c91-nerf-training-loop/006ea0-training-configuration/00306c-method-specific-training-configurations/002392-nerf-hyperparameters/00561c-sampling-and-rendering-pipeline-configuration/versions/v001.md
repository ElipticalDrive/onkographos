# Sampling and Rendering Pipeline Configuration

## Overview
This configuration defines the parameters and logic for generating rays and querying the neural radiance field during training and inference. It solves the problem of balancing rendering fidelity with computational cost by controlling sample density, noise injection, and multi-resolution strategies. It differs from sibling configurations by focusing specifically on the ray-sampling mechanics and the structure of the rendering kwargs dictionary, rather than initialization weights, architecture, or regularization terms.
## Shared principles
The core mechanism involves constructing a `render_kwargs` dictionary that is passed to the volumetric rendering function. This dictionary controls how rays are discretized into samples and how the network outputs are processed. Key shared concepts include:
- **Ray Discretization**: Rays are sampled at discrete points along their length. The number of samples ($N_{samples}$) and the distribution (linear vs. inverse depth) determine the spatial resolution of the query.
- **Noise Regularization**: Random noise is added to the sampled depths or the network's raw density output ($\sigma$) to prevent the network from memorizing the training views and to encourage smoothness [S1, unofficial].
- **Multi-Stage Rendering**: High-fidelity rendering often involves a coarse network for initial sampling and a fine network for denser sampling in promising regions [S1, unofficial].
- **Determinism vs. Stochasticity**: Training typically uses stochastic sampling (perturbation, noise) to improve generalization, while testing uses deterministic sampling to ensure consistent evaluation metrics [S1, unofficial].
## Subtopics

### Coarse/Fine Rendering Pipeline Setup
- **Definition**: A configuration that sets up the rendering arguments to support a two-stage sampling process involving a coarse network and a fine network, along with noise regularization for the density output [S1, unofficial].
- **Step**:
    1. Create a dictionary `render_kwargs_train` containing: `network_query_fn`, `perturb`, `N_importance`, `network_fine`, `N_samples`, `network_fn`, `use_viewdirs`, `white_bkgd`, and `raw_noise_std` [S1, unofficial].
    2. Determine NDC and linear depth settings: If the dataset type is not 'llff' or `args.no_ndc` is true, set `ndc=False` and `lindisp=args.lindisp` [S1, unofficial].
    3. Create `render_kwargs_test` by copying `render_kwargs_train`, but override `perturb=False` and `raw_noise_std=0.` to ensure deterministic output [S1, unofficial].
- **Parameter**:
    - `N_samples`: Number of coarse samples per ray. Default: 64 [S1, unofficial].
    - `N_importance`: Number of additional fine samples per ray. Default: 0 [S1, unofficial].
    - `perturb`: Jitter factor for sampling depths. Default: 1.0 [S1, unofficial].
    - `raw_noise_std`: Standard deviation of noise added to the raw sigma output. Default: 0.0 [S1, unofficial].
- **Code**:
  ```python
  render_kwargs_test['perturb'] = False
  render_kwargs_test['raw_noise_std'] = 0.
  ```
- **Pitfall**: Test rendering must disable perturbation and raw noise to ensure deterministic output for evaluation [S1, unofficial].

### Two-phase progressive sampling and importance sampling
- **Definition**: A training strategy divided into warm-up and finetuning phases to optimize efficiency, accelerate convergence, minimize early overfitting artifacts, and learn intricate details via denser sampling in high-complexity areas [S2].
- **Step**:
    1. Divide training into two phases: warm-up and finetuning [S2].
    2. During the warm-up phase, start with a small number of samples per ray and increase steadily to a target number [S2].
    3. During the finetuning phase, set the base number of samples per ray to match the final count of the warm-up phase [S2].
    4. Add an additional small number of sample points to the finetuning count [S2].
    5. Sample these additional points using an importance sampling technique [S2].
- **Parameter**:
    - Initial warm-up samples per ray: 64 [S2].
    - Additional finetuning samples: 64 [S2].

### Mask-based Ray Sampling
- **Definition**: A ray sampling strategy that excludes rays passing through the black outer areas of laparoscopic images to prevent artifacts caused by overlapping images where black areas intersect with valid color regions of other views [S3].
- **Step**:
    1. Create a single circular mask centered on the image [S3].
    2. Calculate the radius of the circular mask based on the circular view of the laparoscope [S3].
    3. Sample rays only from pixels that fall within the mask (i.e., pixels with color values), excluding rays from the black peripheral areas [S3].
    4. Allow the NeRF model to infer the content of the masked black areas based on color information from overlapping regions in other images [S3].
- **Parameter**:
    - Mask shape: Circular [S3].
    - Mask position: Centered on the image [S3].
- **Pitfall**: Artifacts occur at intersections between valid color lines of one image and black area lines of overlapping images if rays are sampled from the black areas [S3].

## Background
The configuration of sampling and rendering pipelines is fundamental to Neural Radiance Fields (NeRF) and its variants. The original NeRF paper introduced the concept of importance sampling and multi-resolution coarse/fine networks to improve rendering quality [S1, unofficial]. Subsequent works have adapted these pipelines for specific domains, such as dynamic volumetric scenes using two-phase progressive sampling [S2] and medical laparoscopy using mask-based sampling to handle non-circular fields of view [S3].

## Sources
[S1] YoujiaZhang/NeMF, https://github.com/YoujiaZhang/NeMF, fetched 2026-10-27 (unofficial)
[S2] ViSNeRF: Efficient Multidimensional Neural Radiance Field Representation for Visualization Synthesis of Dynamic Volumetric Scenes, https://doi.org/10.1109/pacificvis64226.2025.00029, fetched 2026-10-26
[S3] Towards Abdominal 3-D Scene Rendering from Laparoscopy Surgical Videos Using NeRFs, https://doi.org/10.1007/978-3-031-45673-2_9, fetched 2026-10-26
