# IllumiNeRF

Also called: 3D Relighting without Inverse Rendering

## Overview
IllumiNeRF is a 3D relighting pipeline that reconstructs a relightable 3D representation from posed input images under unknown lighting by generating multiple plausible relit images via a diffusion model and distilling them into a consistent 3D latent NeRF, avoiding inverse rendering [S1]. Unlike inverse rendering which recovers a single point estimate of geometry and materials, IllumiNeRF samples multiple plausible explanations of materials, geometry, and lighting, reconciling them into a single 3D representation that can be rendered from novel viewpoints under a target environment illumination [S1].

## How it works
**Inputs:** A dataset of $N$ input images $I_i$ with corresponding camera poses $\pi_i$, denoted as $D = \{(I_i, \pi_i)\}_{i=1}^N$, and a target environment illumination $L_T$ [S1].

**Outputs:** A trained Latent NeRF model capable of rendering novel views of the object under the target lighting $L_T$ [S1].

**Data Structures:**
1.  **Input Dataset $D$:** A list of tuples containing image tensors and camera pose parameters (extrinsics/intrinsics) [S1].
2.  **Geometry NeRF:** A standard Neural Radiance Field model trained on $D$ to represent the object's 3D geometry and appearance [S1].
3.  **Radiance Cues:** Rendered images representing the object's geometry under target lighting $L_T$ at various roughness levels, used as conditioning inputs for the diffusion model [S1].
4.  **Relighting Diffusion Model:** A pre-trained single-image 2D diffusion model conditioned on radiance cues and target lighting [S1].
5.  **Latent NeRF:** A NeRF model with a latent code per view (or per point) that is optimized to fit the sampled relit images [S1].

**Steps:**
1.  **Geometry Extraction:** Train a standard NeRF on the input dataset $D$ to extract the 3D geometry of the object [S1].
2.  **Radiance Cue Generation:** For each input view $i$, render the extracted geometry under the target environment illumination $L_T$ at various roughness levels to generate radiance cues [S1]. These cues serve as spatial and structural priors for the relighting process.
3.  **Diffusion-Based Relighting:** Independently relight each input image $I_i$ using the Relighting Diffusion Model. The model is conditioned on the radiance cues generated in Step 2 and the target lighting $L_T$ [S1].
4.  **Sample Generation:** Sample $S$ plausible relit images for each of the $N$ input views. This captures different plausible explanations of materials, geometry, and input lighting, treating the underlying explanations as samples of unobserved latent variables $Z$ [S1]. The total number of samples is $N \times S$.
5.  **Latent NeRF Optimization:** Optimize a Latent NeRF using the full set of $N \times S$ sampled relit images. The goal is to distill these samples into a single consistent 3D representation [S1]. The likelihood of the relit data is modeled as:
    $$ p(D^T | D) = \int p(D^T, Z | D) dZ = \int p(D^T | Z, D) p(Z | D) dZ $$
    where $Z$ is a latent variable implicitly representing the input images' lighting along with the object's material and geometry parameters [S1]. The Latent NeRF is trained to minimize the reconstruction error between its renders and the sampled relit images [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def illuminerf_pipeline(input_images, camera_poses, target_lighting, num_samples_per_view):
    """
    input_images: List of N images
    camera_poses: List of N camera poses
    target_lighting: Target environment illumination L_T
    num_samples_per_view: S, number of diffusion samples per view
    """
    # Step 1: Train standard NeRF for geometry
    geometry_nerf = train_standard_nerf(input_images, camera_poses)
    
    # Step 2: Generate radiance cues
    radiance_cues = []
    for i in range(len(input_images)):
        # Render geometry under target lighting at various roughness levels
        cue = render_geometry_with_lighting(geometry_nerf, camera_poses[i], target_lighting, roughness_levels=[0.1, 0.5, 0.9])
        radiance_cues.append(cue)
        
    # Step 3 & 4: Sample relit images using Diffusion Model
    sampled_relit_images = []
    for i in range(len(input_images)):
        for s in range(num_samples_per_view):
            # Condition diffusion model on radiance cue and target lighting
            relit_img = relighting_diffusion_model.sample(
                conditioning_image=radiance_cues[i],
                target_lighting=target_lighting
            )
            sampled_relit_images.append((relit_img, camera_poses[i]))
            
    # Step 5: Optimize Latent NeRF
    latent_nerf = initialize_latent_nerf()
    for epoch in range(num_training_epochs):
        loss = 0
        for relit_img, pose in sampled_relit_images:
            # Render from latent nerf
            rendered_img = latent_nerf.render(pose)
            # Compute reconstruction loss (e.g., MSE)
            loss += mse_loss(rendered_img, relit_img)
        # Update latent nerf parameters
        optimize(latent_nerf, loss)
        
    return latent_nerf
```

**Parameters:**
*   $N$: Number of input images.
*   $S$: Number of diffusion samples per view. Typical values are not specified in the sources, but the method relies on sampling multiple explanations [S1].
*   Roughness levels: Used for generating radiance cues. Specific values are not provided, but multiple levels are required [S1].

**Edge Cases and Pitfalls:**
*   **Consistency:** The Latent NeRF must reconcile potentially inconsistent samples from the diffusion model. If $S$ is too low, the representation may be biased; if too high, training may be unstable or slow [S1].
*   **Geometry Accuracy:** The quality of the initial geometry NeRF directly impacts the radiance cues. Poor geometry leads to poor conditioning for the diffusion model [S1].
*   **Lighting Ambiguity:** The method explicitly handles lighting ambiguity by sampling, but the final Latent NeRF represents an average or consensus of these samples, which may not perfectly match any single physical explanation [S1].

## Operating characteristics
*   **Time Complexity:** The method involves training a NeRF, running a diffusion model $N \times S$ times, and training a Latent NeRF on $N \times S$ images. Diffusion sampling is computationally expensive, making the overall pipeline slower than standard NeRF training [S1].
*   **Memory Complexity:** Requires storing the geometry NeRF, the diffusion model, and the Latent NeRF. The intermediate sampled images ($N \times S$) must be stored or streamed during Latent NeRF training [S1].
*   **Performance:** Evaluated on the Stanford-ORB dataset and TensoIR benchmarks, achieving state-of-the-art results compared to 3D inverse rendering baselines for relightable view synthesis [S1].
*   **Strengths:** Avoids the brittleness and computational expense of differentiable Monte Carlo rendering used in inverse rendering. Does not require explicit recovery of geometry, materials, and lighting [S1].
*   **Limitations:** Relies on the quality of the pre-trained diffusion model. The resulting 3D representation is a "latent" one, meaning it may not have physically accurate material properties in the traditional sense [S1].

## Background
IllumiNeRF was proposed as a simpler approach to 3D relighting, replacing 3D inverse rendering with a diffusion-based sampling and distillation strategy [S1]. It addresses the central problem of capturing an object’s appearance for accurate rendering in novel environments, which is crucial for AR/VR, photography, and game development [S1]. The work was presented at the 38th Conference on Neural Information Processing Systems (NeurIPS) [S1].

## Sources
[S1] IllumiNeRF: 3D Relighting without Inverse Rendering, https://doi.org/10.48550/arXiv.2406.06527, fetched 2026-10-29
