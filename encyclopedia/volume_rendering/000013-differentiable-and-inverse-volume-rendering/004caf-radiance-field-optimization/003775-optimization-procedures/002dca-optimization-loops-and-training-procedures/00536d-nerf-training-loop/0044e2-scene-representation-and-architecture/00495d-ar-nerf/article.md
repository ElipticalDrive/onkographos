# AR-NeRF

Also called: Aperture Rendering Neural Radiance Fields, aperture rendering NeRF

## Overview
AR-NeRF is an extension of Neural Radiance Fields (NeRF) that employs an aperture camera model instead of a pinhole camera model to represent defocus effects and viewpoint changes in a unified manner within a common ray-tracing framework [S1]. It enables fully unsupervised learning of depth and defocus effects from single natural images by jointly leveraging defocus and viewpoint cues without explicit supervision or pretrained models [S1]. Unlike standard NeRF, which assumes a pinhole camera, AR-NeRF represents each pixel using a collection of rays converging at a focus plane, allowing the model to disentangle defocus-aware and defocus-independent representations through aperture randomized training [S1].

## How it works
AR-NeRF builds on the [Scene Representation and Architecture](11368-scene-representation-and-architecture.md) principles, specifically the use of coupled MLPs for density and color prediction, but modifies the ray generation and training loop to support aperture rendering.

### Inputs and Outputs
**Inputs:**
*   Latent code $\mathbf{z} \in \mathbb{R}^{L_z}$: Represents the scene identity and view-independent features [S1].
*   Aperture size $s$: A scalar parameter representing the radius of the aperture camera, sampled from a half-normal distribution with standard deviation $\sigma_s$ [S1].
*   Focus distance $f$: A variable representing the distance between the ray origin and the plane where all objects are in focus [S1].
*   Camera pose parameters: Restricted to a standard deviation of 0.1 radian to prevent sampled parameters from going beyond the real distribution [S1].

**Outputs:**
*   Rendered color $\mathbf{C}(\mathbf{r}) \in \mathbb{R}^3$ per pixel [S1].
*   Rendered depth $Z(\mathbf{r}) \in \mathbb{R}$ per pixel [S1].

### Data Structures and Network Layout
The core representation uses the standard NeRF MLP architecture described in the parent article [S1].
*   **MLP Inputs:** The MLP takes 3D position $\mathbf{x} \in \mathbb{R}^3$, view direction $\mathbf{d} \in S^2$, and latent code $\mathbf{z} \in \mathbb{R}^{L_z}$ as inputs [S1].
*   **MLP Outputs:** Predicts RGB color $\mathbf{c}(\mathbf{x}, \mathbf{d}) \in \mathbb{R}^3$ and volume density $\sigma(\mathbf{x}) \in \mathbb{R}^+$ [S1].
*   **Focus Distance MLP:** An additional MLP takes the latent code $\mathbf{z}$ as input to predict the focus distance $f$, under the assumption that $f$ is determined according to the rendered target [S1].

### Ray Generation and Aperture Sampling
In standard NeRF, each pixel corresponds to a single ray under the pinhole camera assumption [S1]. In AR-NeRF, each pixel corresponds to a collection of rays that converge at the focus plane, with the scale of the collection determined by the aperture size $s$ [S1].

To efficiently represent the aperture with limited rays, AR-NeRF uses stratified sampling [S1]. Specifically, it generates a finite bundle of **five rays** per pixel:
1.  One ray origin is placed at the center of the aperture.
2.  The origins of the other four rays are placed along the circumference of the aperture with equal intervals [S1].

The ray origins are sampled from offsets $|\mathbf{u}| \in [0, s]$ [S1]. The final color and depth for a pixel are calculated by integrating over the rendered values for all rays in the bundle and taking the average [S1].

### Rendering Integration
For each ray $\mathbf{r}'$ in the bundle, the color $\mathbf{C}(\mathbf{r}')$ and depth $Z(\mathbf{r}')$ are calculated by integrating over the ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ within distance $t \in [t_n, t_f]$ [S1]:

$$
\mathbf{C}(\mathbf{r}') = \int_{t_n}^{t_f} T(t) \sigma(\mathbf{r}(t)) \mathbf{c}(\mathbf{r}(t), \mathbf{d}) dt
$$

$$
Z(\mathbf{r}') = \int_{t_n}^{t_f} T(t) \sigma(\mathbf{r}(t)) t dt
$$

where the transmittance $T(t)$ is defined as:

$$
T(t) = \exp \left( - \int_{t_n}^t \sigma(\mathbf{r}(s)) ds \right)
$$

The final pixel output is the average of $\mathbf{C}(\mathbf{r}')$ and $Z(\mathbf{r}')$ for all rays in the bundle [S1]. Note that while depth integration is used to account for ambiguity derived from defocus blur, the effect on depth accuracy is empirically found to be subtle [S1].

### Training Procedure: Aperture Randomized Training
AR-NeRF is trained using a GAN framework to disentangle defocus-aware and defocus-independent representations [S1]. The key innovation is **aperture randomized training**, where the aperture size $s$ and latent code $\mathbf{z}$ are varied randomly and independently during training [S1].

The loss function is modified from the standard GAN objective to sample $\mathbf{z}$ and $s$ independently:

$$
\mathcal{L}_{\text{AR-NeRF}} = \mathbb{E}_{\mathbf{I}^r \sim p^r(\mathbf{I})} [\log D(\mathbf{I}^r)] + \mathbb{E}_{\mathbf{z} \sim p^g(\mathbf{z}), s \sim p^g(s)} [\log (1 - D(G(\mathbf{z}, s)))]
$$

where $\mathbf{I}^r$ is a real image, $D$ is the discriminator, and $G$ is the generator (AR-NeRF) [S1]. The distribution $p^g(s)$ is represented using a half-normal distribution, with its standard deviation $\sigma_s$ parameterized to determine the range of aperture sizes in a data-driven manner [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_aperture_rays(pixel_center, aperture_size_s, num_rays=5):
    """
    Generates a bundle of rays for a single pixel using stratified sampling.
    
    Args:
        pixel_center: 2D coordinates of the pixel center in camera space.
        aperture_size_s: Scalar radius of the aperture.
        num_rays: Number of rays to sample (default 5).
        
    Returns:
        origins: Array of shape (num_rays, 3) for ray origins.
        directions: Array of shape (num_rays, 3) for ray directions.
    """
    # Note: Exact camera intrinsics and coordinate system details are not provided in [S1].
    # This implementation assumes a standard pinhole projection where direction is 
    # determined by pixel center and origin is offset by aperture vector u.
    
    origins = []
    directions = []
    
    # Ray 1: Center of aperture
    u_center = np.array([0.0, 0.0])
    origin = pixel_center + u_center
    # Direction is typically from origin through the focal plane. 
    # In AR-NeRF, rays converge at the focus plane. 
    # Assuming standard ray generation logic where d is fixed for the pixel 
    # and o varies, or d varies to converge. 
    # [S1] states rays converge at the focus plane. 
    # Standard implementation: origin = camera_center + u, direction = normalize(focus_point - origin)
    # However, [S1] simplifies this to "collection of rays... converge at the focus plane".
    # We assume the direction is computed based on the convergence point.
    
    # Placeholder for specific camera geometry calculation not detailed in [S1]
    # Typically: origin = o + u, direction = normalize(f - origin) where f is focus point
    # For the purpose of this pseudocode, we assume a helper function exists 
    # that computes the ray bundle given the aperture offset and focus distance.
    
    # Stratified sampling: 1 center, 4 circumference
    angles = np.linspace(0, 2 * np.pi, num_rays, endpoint=False)
    
    for i in range(num_rays):
        if i == 0:
            u = np.array([0.0, 0.0])
        else:
            # Place on circumference with equal intervals
            # Radius is s. 
            u = np.array([aperture_size_s * np.cos(angles[i]), 
                          aperture_size_s * np.sin(angles[i])])
        
        # Compute ray origin and direction based on camera model
        # [S1] does not provide the exact geometric formula for origin/direction 
        # from u and focus distance f. 
        # [G] Standard aperture camera model: 
        # Origin = CameraCenter + u (in camera coordinates)
        # Direction = normalize(FocusPoint - Origin)
        # This is a standard assumption for aperture rendering.
        origin = compute_origin(pixel_center, u) 
        direction = compute_direction(origin, focus_distance)
        
        origins.append(origin)
        directions.append(direction)
        
    return np.array(origins), np.array(directions)

def render_pixel(nerf_mlp, pixel_center, aperture_size_s, focus_distance, latent_code_z, 
                 near_plane, far_plane, num_samples=128):
    """
    Renders a single pixel using AR-NeRF's aperture rendering.
    
    Args:
        nerf_mlp: The trained NeRF MLP function.
        pixel_center: 2D pixel coordinates.
        aperture_size_s: Current aperture size.
        focus_distance: Current focus distance.
        latent_code_z: Latent code for the scene.
        near_plane, far_plane: Depth bounds.
        num_samples: Number of samples per ray.
        
    Returns:
        color: Rendered RGB color.
        depth: Rendered depth.
    """
    # 1. Generate ray bundle
    origins, directions = generate_aperture_rays(pixel_center, aperture_size_s)
    
    total_color = np.zeros(3)
    total_depth = 0.0
    num_rays = len(origins)
    
    for i in range(num_rays):
        o = origins[i]
        d = directions[i]
        
        # 2. Sample points along the ray
        t_vals = np.linspace(near_plane, far_plane, num_samples)
        points = o + t_vals[:, np.newaxis] * d
        
        # 3. Query MLP for density and color
        # [S1] MLP inputs: x, d, z
        # [G] Positional encoding is applied to x and d as per parent article
        densities, colors = nerf_mlp(points, d, latent_code_z)
        
        # 4. Volume rendering for this ray
        # Compute alpha and transmittance
        deltas = t_vals[1:] - t_vals[:-1]
        # Add a small epsilon to avoid division by zero or log(0)
        deltas = np.append(deltas, 1e-10)
        
        alpha = 1.0 - np.exp(-densities * deltas)
        T = np.cumprod(np.exp(-np.cumsum(densities * deltas)))
        T = np.concatenate(([1.0], T[:-1]))
        
        # Color integration
        # C(r) = sum(T_i * alpha_i * c_i)
        weights = alpha * T
        color = np.sum(weights[:, np.newaxis] * colors, axis=0)
        
        # Depth integration
        # Z(r) = sum(T_i * alpha_i * t_i)
        depth = np.sum(weights * t_vals)
        
        total_color += color
        total_depth += depth
        
    # 5. Average over the ray bundle
    final_color = total_color / num_rays
    final_depth = total_depth / num_rays
    
    return final_color, final_depth

def train_step(arnerf_model, discriminator, real_image, latent_code_z, aperture_size_s):
    """
    Performs one training step for AR-NeRF.
    
    Args:
        arnerf_model: The AR-NeRF generator (MLPs).
        discriminator: The GAN discriminator.
        real_image: Ground truth image from the dataset.
        latent_code_z: Sampled latent code.
        aperture_size_s: Sampled aperture size.
        
    Returns:
        loss: The computed loss value.
    """
    # 1. Generate fake image using AR-NeRF
    # [S1] Focus distance f is predicted by an MLP taking z as input
    focus_distance = arnerf_model.predict_focus_distance(latent_code_z)
    
    # Render the image (simplified: assumes full image rendering)
    fake_image = arnerf_model.render_image(latent_code_z, aperture_size_s, focus_distance)
    
    # 2. Compute GAN Loss
    # L_AR-NeRF = E[log D(I^r)] + E[log(1 - D(G(z, s)))]
    # Note: In practice, this involves backpropagation through the discriminator
    # and the generator.
    
    pred_real = discriminator(real_image)
    pred_fake = discriminator(fake_image)
    
    loss_d = -np.mean(np.log(pred_real + 1e-8)) - np.mean(np.log(1 - pred_fake + 1e-8))
    
    # Update discriminator and generator
    # [G] Standard GAN update steps (e.g., Adam optimizer)
    update_discriminator(discriminator, loss_d)
    update_generator(arnerf_model, loss_d)
    
    return loss_d
```

### Edge Cases and Pitfalls
*   **Unbounded Background:** Standard generative NeRF renders scenes in a tightly bounded 3D space, which prevents representing strong defocus effects at far distances [S1]. To address this, AR-NeRF implements a synthesis network using **NeRF++**, which is composed of a foreground NeRF in a unit sphere and a background NeRF modeled using inverted sphere parameterization [S1]. This allows representing strong defocus effects in the far background [S1].
*   **Aperture Size Range:** The aperture size $s$ is sampled from a half-normal distribution. The standard deviation $\sigma_s$ is parameterized to determine the range of aperture sizes in a data-driven manner [S1].
*   **Camera Pose Restriction:** To prevent sampled camera parameters from going beyond the real distribution, the range is restricted using a standard deviation of 0.1 radian in practice [S1].

## Operating characteristics
*   **Time Complexity:** Rendering time is proportional to the number of rays per pixel. AR-NeRF uses 5 rays per pixel, which is significantly fewer than the ~100 rays typically used in aperture rendering for high fidelity, but more than the 1 ray used in standard NeRF [S1]. This increases processing time and memory compared to standard NeRF but is efficient enough for training [S1].
*   **Memory Complexity:** The memory complexity is dominated by the MLP parameters and the storage of the scene representation, similar to standard NeRF [S1].
*   **Strengths:**
    *   Unifies defocus and viewpoint changes in a single ray-tracing framework [S1].
    *   Enables fully unsupervised learning of depth and defocus from single images [S1].
    *   Disentangles defocus-aware and defocus-independent representations via aperture randomized training [S1].
    *   Can explain and utilize other ray-tracing-related phenomena (e.g., viewpoints) in a unified manner, unlike AR-GAN which is specific to depth and defocus [S1].
*   **Limitations:**
    *   Requires the assumption of diversely defocused images in the dataset [S1].
    *   The discretized ray sampling (5 rays) is an approximation; increasing the number of rays would improve fidelity but at higher computational cost [S1].
    *   Depth accuracy is subtle and may require additional supervision or post-processing for precise metric depth [S1].

## Background
AR-NeRF was proposed by Takuhiro Kaneko et al. from NTT Communication Research Laboratories [S1]. The work was published in 2022 [S1]. It builds upon the motivation of AR-GAN [37] to learn depth and defocus effects but extends it to continuous radiance fields [S1]. The project page is available at https://www.kecl.ntt.co.jp/people/kaneko.takuhiro/projects/ar-nerf/ [S1].

## Sources
[S1] AR-NeRF: Unsupervised Learning of Depth and Defocus Effects from Natural Images with Aperture Rendering Neural Radiance Fields, https://arxiv.org/abs/2206.06100v1, fetched 2026-10-26
