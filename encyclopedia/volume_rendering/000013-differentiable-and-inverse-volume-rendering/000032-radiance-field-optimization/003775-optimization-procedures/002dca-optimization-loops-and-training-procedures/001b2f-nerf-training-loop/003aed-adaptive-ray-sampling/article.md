# Adaptive Ray Sampling


## Overview
Adaptive ray sampling refers to a family of optimization procedures in neural radiance field (NeRF) training that dynamically determine the spatial distribution of sample points along viewing rays to improve reconstruction quality and training efficiency. Unlike uniform sampling, which treats all regions of a ray equally, these methods allocate computational resources to areas of high geometric or photometric complexity, such as object surfaces or textured regions. This family is distinguished from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by its focus on the *selection* of sample locations based on learned or heuristic priors, rather than the integration technique itself.

## Shared principles
All methods in this family share the goal of replacing or augmenting uniform ray sampling with a distribution that concentrates samples where they contribute most to the rendering loss. The core mechanism involves:
1.  **Evaluation:** Computing a metric (e.g., density, error, variance, depth prior) for a ray or region.
2.  **Distribution:** Deriving a probability density function (PDF) or a set of weights from this metric.
3.  **Resampling:** Drawing new sample points from this distribution, often using inverse transform sampling or stochastic methods.
4.  **Integration:** Rendering the ray using the adapted samples.

Common mathematical components include the volume rendering equation, where weights $w_i$ are derived from density $\sigma$ and transmittance $T$. Many methods use a coarse-to-fine strategy, where a cheap coarse pass informs the fine pass.

## Subtopics

### [Mip-NeRF 360 Sampling and Training](3393-mip-nerf-360-sampling-and-training.md)
Optimizes radiance fields for unbounded scenes using a disparity-based ray parameterization, a coordinate contraction function, and a coarse-to-fine training loop with a proposal network. It mitigates artifacts such as floaters and background collapse by regularizing the density distribution and ensuring efficient sampling through online distillation. The method differs from standard NeRF by using linear-in-disparity sampling and handles unbounded scenes via coordinate contraction.

### [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md)
A hierarchical sampling strategy that improves the learning of high-frequency details in neural radiance fields by allocating computational resources adaptively along rays. It solves the problem of uniform sampling being inefficient for sparse or complex geometry by using a coarse network to identify opaque regions and a fine network to densely sample those specific areas. Unlike standard uniform sampling which distributes points evenly across the entire ray segment, this method clusters additional samples near the object boundaries identified by the coarse pass, thereby improving detail reconstruction without a proportional increase in coarse sampling cost.

### [Probability-based Ray Sampling with Quadtree Subdivision](7481-probability-based-ray-sampling-with-quadtree-subdivision.md)
This method accelerates radiance field training by dynamically adjusting ray distribution through a quadtree structure on input views. It reduces the number of unnecessary network evaluations in low-detail regions while maintaining accuracy in complex areas, achieving 15–40% faster training times compared to baselines like NeRF, Plenoxels, and Instant-NGP. Unlike sibling methods that use fixed block sizes or uniform sampling within blocks, this approach subdivides leaf nodes based on rendering error and samples rays according to image context probability.

### [Hard Sample Mining](8673-hard-sample-mining.md)
Hard Sample Mining (HSM) is an optimization procedure that reduces training time and memory usage by subsampling a "hard minibatch" of point samples ranked by their propagated pixel loss, rather than using all samples for the gradient update. It differs from sibling methods like Nerfacc, which prune low-weight samples based on density/opacity, by explicitly using the error signal (pixel loss) to identify samples that contribute most to the optimization gradient.

### [TermiNeRF](8828-terminerf.md)
TermiNeRF is a method for efficient neural rendering that jointly trains a lightweight sampling network alongside a color network to predict the distribution of ideal sampling locations along a viewing ray. By concentrating samples where opacity is high and skipping free space, it significantly reduces the number of expensive color network evaluations required per pixel. Unlike hierarchical sampling methods that use a coarse network to guide fine sampling, TermiNeRF uses a single evaluation of the sampling network per ray to predict weights for a fixed set of bins, allowing for adaptive, non-uniform sampling without the overhead of multi-stage network queries.

### [Pixel Region and Depth Guided Ray Sampling](8866-pixel-region-and-depth-guided-ray-sampling.md)
Pixel Region and Depth Guided Ray Sampling is a ray sampling strategy that increases sampling probability in pixel areas exhibiting significant color and depth variation, reducing redundant rays in flat regions to accelerate network convergence and improve geometry perception. It differs from uniform sampling by dynamically allocating ray density based on image-space statistics (color variance) and geometric cues (depth boundaries) rather than a fixed grid. This method is designed to be a drop-in replacement for uniform sampling in existing NeRF pipelines, such as ENeRF, to achieve faster training and higher quality in textured regions.

### [NeRF-ID](8867-nerf-id.md)
NeRF-ID replaces the heuristic coarse-to-fine sampling procedure of vanilla NeRF with a differentiable 'proposer' module, enabling end-to-end training of the coarse network, proposer, and fine network to improve view synthesis quality. Unlike the original NeRF which uses a handcrafted procedure involving coarse densities to define a piece-wise constant probability density function (PDF) for resampling, NeRF-ID uses a learnable proposer that takes coarse features and positions as input to directly output fine sample locations.

### [Two-Stage Sampling with Occupancy Grid and Nonlinear Function](9146-two-stage-sampling-with-occupancy-grid-and-nonlinear-functio.md)
This method is a sampling strategy for neural implicit surface reconstruction that combines an Occupancy Grid to skip empty regions (coarse stage) and a nonlinear piecewise exponential function to guide sampling points closer to the object surface within occupied regions (fine stage). It reduces training time and sample count while maintaining reconstruction quality compared to uniform sampling in occupied cells. Unlike NeuS, which uses inverse transform sampling leading to random scatter near the surface, this method biases samples toward the surface using the nonlinear function. Unlike Instant-NGP, which uses uniform sampling within occupied grid cells, this method uses the nonlinear function to guide sampling within those cells.

### [Truncated Depth-Based Sampling](10613-truncated-depth-based-sampling.md)
Truncated Depth-Based Sampling (TDBS) is a ray sampling strategy that utilizes monocular depth priors informed by a truncated normal distribution to optimize ray point sampling, improving convergence speed and accuracy of pose estimation in joint NeRF and camera pose optimization. It sets itself apart from alternatives like NoPe-NeRF by explicitly modeling the probability of surface location around the depth prior using a truncated normal distribution, rather than merely considering overall depth distortion.

#### Error Map Importance Sampling
- **Step**: If `opt.error_map_size` is set, compute sampling weights: `sample_weight = error_map + 2 * error_map.mean(dim=-1, keepdim=True)`. Sample ray indices using multinomial distribution: `var.ray_idx_coarse = torch.multinomial(sample_weight, opt.nerf.n_rays//batch_size, replacement=False)`. Decode 1D sampled indices to 2D grid coordinates: `inds_x = ray_idx // error_map_size`, `inds_y = ray_idx % error_map_size` [S1, unofficial].
- **Parameter**: `error_map_size`: int, defines the resolution of the 2D error map grid [S1, unofficial].
- **Pitfall**: The integer division operator `//` triggers a PyTorch 1.10+ deprecation warning; use `torch.div` or `torch.floor_divide` instead [S1, unofficial].
- **Comparison**: Combines 1/3 importance sampling based on error map with 2/3 uniform random sampling to prevent overfitting to local error spikes [S1, unofficial].

## Background
Adaptive sampling in volume rendering originated from the need to efficiently render complex scenes where uniform sampling is computationally prohibitive. In the context of NeRF, the seminal work by Mildenhall et al. introduced hierarchical sampling to improve detail [S2, S3, unofficial]. Subsequent works like Mip-NeRF [S1, unofficial] and Mip-NeRF 360 [S1, unofficial] introduced disparity-based parameterization and proposal networks to handle unbounded scenes and anti-aliasing. Other approaches, such as TermiNeRF [S1, S2] and NeRF-ID [S1, unofficial], explored learnable sampling networks to replace heuristic rules. Methods like Quadtree Subdivision [S1, unofficial] and Hard Sample Mining [S1, unofficial] focused on image-space or loss-space adaptivity to accelerate training. Truncated Depth-Based Sampling [S1, unofficial] integrated monocular depth priors for joint pose optimization. These methods collectively form the adaptive sampling family, evolving from simple hierarchical schemes to complex, learnable, and prior-guided strategies.

## Sources
[S1] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
[S2] Mildenhall, B., et al. "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis." ECCV 2020.
[S3] Barron, J.T., et al. "Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields." ICCV 2021.
