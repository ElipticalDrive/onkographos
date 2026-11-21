# UniSDF-based Latent NeRF

Also called: Latent NeRF Model with UniSDF

## Overview
UniSDF-based Latent NeRF is a geometry estimator and appearance model that utilizes UniSDF, a signed distance function (SDF) based volume rendering approach, to decouple geometry from appearance [S1]. It solves the problem of generating relightable 3D assets by extracting a mesh from the SDF for radiance cue computation via a standard rendering engine, while optimizing a latent NeRF for appearance [S1]. It sets itself apart from sibling methods by explicitly leveraging SDF-based mesh extraction to compute physically-based shading normals and radiance cues, rather than relying solely on differentiable volume rendering for all geometric cues [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Input images, camera poses, and a mask.
*   **Outputs:** A mesh representing the geometry, a latent NeRF model for appearance, and radiance cues (shading normals, depth, etc.) rendered from the mesh.

**Data Structures:**
*   **UniSDF Model:** Parameterized similarly to the original UniSDF for the DTU dataset [S1], with specific modifications to proposal sampling and encoding [S1].
*   **Hash Grid Encoding:** An additional hash grid encoding with 15 scales between a resolution of 32 and 4096, used exclusively for outputting predicted normals [S1].
*   **GLO Embeddings:** Applied via an MLP that predicts element-wise scale and shift values for the 'bottleneck' feature of UniSDF [S1].

**Numbered Steps:**
1.  **Geometry Estimation:** Optimize the UniSDF model on the input images. The number of proposal sampling rounds is reduced from two to one, using 64 proposal samples [S1].
2.  **Normal Prediction:** Use the additional hash grid encoding (15 scales, resolution 32 to 4096) to output predicted normals [S1].
3.  **Loss Computation:** Compute the asymmetric predicted normal loss $L_p$ and the smoothness loss $L_s$ during optimization [S1].
4.  **Mesh Extraction:** Convert the optimized SDF representation to a mesh using marching cubes with a threshold set to zero [S1].
5.  **Radiance Cue Rendering:** Import the extracted mesh into a standard rendering engine (Blender Cycles) with predefined materials based on the GGX material model to render radiance cues [S1].
6.  **Latent NeRF Optimization:** Optimize the latent NeRF model. Incorporate GLO embeddings by using an MLP to predict element-wise scale and shift values applied to the UniSDF 'bottleneck' feature [S1].

**Formulas:**
The asymmetric predicted normal loss $L_p$ is defined as [S1]:
$$ L_p = \sum_i \left( \lambda_1 \omega_i \|\nabla n_i - \nabla n'_i \|_2 + \lambda_2 \nabla \omega_i \| n_i - n'_i \|_2 + \lambda_3 \nabla \omega_i \|\nabla n_i - n'_i \|_2 \right) $$
where:
*   $\omega_i$ is the volume rendering weight of the $i$-th sample.
*   $\nabla$ denotes the stop-gradient operator.
*   $n_i$ is the $i$-th sample's density normal.
*   $n'_i$ is the $i$-th sample's predicted normal.
*   $\lambda_1 = 10^{-3}$, $\lambda_2 = 10^{-3}$, $\lambda_3 = 10^{-2}$ [S1].

The smoothness loss $L_s$ is defined as [S1]:
$$ L_s = \lambda_4 \sum_i \omega_i \| n'(x_i + \epsilon) - n'(x_i) \|_2^2 $$
where:
*   $x_i$ is the 3D position of the $i$-th sample.
*   $\epsilon \sim \mathcal{N}(0, \sigma^2 I)$ is an isotropic Gaussian random variable used to perturb the sample locations.
*   $\lambda_4 = 0.1$.
*   $\sigma = 0.01$ [S1].

**Parameters:**
*   **Optimizer:** Adam with $\beta_1 = 0.9$, $\beta_2 = 0.99$, and $\epsilon = 1 \times 10^{-15}$ [S1].
*   **Learning Rate:** Decayed logarithmically from $5 \times 10^{-3}$ to $5 \times 10^{-4}$ over 25k training iterations [S1].
*   **Warmup:** Cosine-scheduled warmup in the first 500 steps [S1].
*   **Proposal Samples:** 64 samples, with one round of proposal sampling [S1].
*   **Hash Grid:** 15 scales, resolution range 32 to 4096 [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
model = UniSDFModel()
hash_grid = HashGrid(num_levels=15, resolution_min=32, resolution_max=4096)
optimizer = Adam(params=model.parameters(), lr=5e-3, beta1=0.9, beta2=0.99, eps=1e-15)
warmup_steps = 500
total_steps = 25000

# Training Loop
for step in range(total_steps):
    # Learning rate scheduling
    if step < warmup_steps:
        lr = cosine_warmup(step, warmup_steps, initial_lr=5e-3, final_lr=5e-4)
    else:
        lr = logarithmic_decay(step - warmup_steps, total_steps - warmup_steps, start_lr=5e-3, end_lr=5e-4)
    optimizer.set_learning_rate(lr)

    # Forward pass
    samples = sample_rays(num_proposal_samples=64, num_rounds=1)
    sdf_values, densities = model.forward(samples.positions)
    predicted_normals = hash_grid.forward(samples.positions) # Additional hash grid for normals
    
    # Compute losses
    density_normals = compute_density_normals(densities)
    loss_p = compute_asymmetric_normal_loss(
        weights=samples.weights,
        density_normals=density_normals,
        predicted_normals=predicted_normals,
        lambda1=1e-3,
        lambda2=1e-3,
        lambda3=1e-2
    )
    
    # Smoothness loss
    epsilon = sample_gaussian_noise(samples.positions, sigma=0.01)
    perturbed_normals = hash_grid.forward(samples.positions + epsilon)
    loss_s = 0.1 * mean_squared_error(perturbed_normals, predicted_normals, weights=samples.weights)
    
    total_loss = loss_p + loss_s
    
    # Backward pass
    optimizer.zero_grad()
    total_loss.backward()
    optimizer.step()

# Post-processing: Mesh Extraction
mesh = marching_cubes(sdf=model.sdf_field, threshold=0.0)

# Radiance Cue Rendering
radiance_cues = render_with_blender_cycles(mesh, material_model='GGX')

# Latent NeRF Optimization (Parallel or Sequential)
# Incorporate GLO embeddings
bottleneck_feature = model.get_bottleneck_feature()
scale_shift = glo_mlp.predict(bottleneck_feature)
bottleneck_feature = bottleneck_feature * scale_shift.scale + scale_shift.shift
latent_nerf.optimize(radiance_cues, bottleneck_feature)
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Decouples geometry from appearance, allowing fixed geometry weights and optimized appearance weights [S1]. Enables mesh extraction for radiance cue computation [S1]. Modifications result in better and smoother geometry necessary for relighting objects with specular highlights [S1].
*   **Limitations:** Relies on external rendering engine (Blender Cycles) for radiance cues [S1].
*   **When to use:** When relightable 3D assets with specular highlights are required [S1].
*   **When not to use:** When end-to-end differentiable volume rendering without external tools is required [S1].

## Background
UniSDF-based Latent NeRF is based on the IllumiNeRF paper [S1]. It uses UniSDF [56] as the geometry estimator and Latent NeRF model [S1]. The method was implemented using JAX [11] [S1].

## Sources
[S1] IllumiNeRF: 3D Relighting without Inverse Rendering, https://doi.org/10.48550/arXiv.2406.06527, fetched 2026-10-29
