# Generative Radiance Fields (GRAF)


## Overview
Generative Radiance Fields (GRAF) is a method for 3D-aware image synthesis and shape estimation from unposed images, enabling volume reconstruction without explicit camera pose information [S1]. Unlike standard NeRF which requires known camera poses, GRAF uses an adversarial training scheme where a generator conditioned on pose, shape, and appearance codes learns to synthesize images that fool a patch-based discriminator [S1]. It builds on the original NeRF representation and rendering equation, replacing the supervised reconstruction loss with an adversarial objective to handle datasets where relative camera positions are unknown [S1].

## How it works

### Inputs and Outputs
*   **Inputs:** A dataset of unposed 2D RGB images [S1]. During training, the generator receives a requested viewing pose (elevation and azimuth angles), and optionally shape and appearance codes [S1]. The discriminator receives patches from real images and generated images [S1].
*   **Outputs:** Synthesized RGB images of the scene from specified viewpoints, depth maps, and the learned 3D implicit representation (NeRF weights) [S1].

### Data Structures
The core representation is a Multilayer Perceptron (MLP) acting as the radiance field, identical to the original NeRF architecture [S1].
*   **MLP Network:** 8 fully-connected layers [S1].
*   **Channels:** 256 channels per layer [S1].
*   **Activation:** ReLU for hidden layers [S1].
*   **Input to MLP:** 5D coordinate consisting of 3D position $(x, y, z)$ and 2D viewing direction $(\theta, \phi)$ [S1].
*   **Output of MLP:** Volume density $(\sigma)$ and emitted RGB color $(c)$ [S1].
*   **Generator Conditioners:** Pose (elevation and azimuth angles), Shape Code (latent vector controlling volume density distribution), and Appearance Code (latent vector controlling color/albedo) [S1].
*   **Discriminator:** Patch-based network operating on square arrays of pixels [S1].

### Numbered Steps
1.  **Patch Selection:** Instead of predicting full images, the generator selects a patch position and a distance between pixels within that patch [S1]. A patch represents a square array of pixels [S1].
2.  **Conditioning:** The generator takes the requested pose, and optionally shape and appearance codes, as input [S1].
3.  **Ray Marching:** For each pixel in the patch, a ray is cast based on the requested pose [S1].
4.  **NeRF Query:** At each sample point along the ray, the MLP is queried with the 3D position and viewing direction to obtain density and color [S1].
5.  **Volume Rendering:** The color and density values are integrated along the ray using the standard NeRF rendering equation to produce the color for each pixel in the patch [S1].
6.  **Adversarial Training (Generator):** The generated patch is compared against a real patch from the dataset by the discriminator [S1].
7.  **Discriminator Update:** The discriminator learns to differentiate real patches from generated ones [S1].
8.  **Generator Update:** The generator's weights are updated to minimize the discriminator's ability to distinguish generated patches from real ones, effectively matching the probabilistic distribution of real image pixels [S1].
9.  **Receptive Field Decay:** The size of the receptive field of the patch (or the distance between pixels) is decreased during training [S1]. This allows the model to first capture global contours and progressively improve local details [S1].
10. **Inference:** To synthesize an image, the generator is fed a specific viewing angle (pose) [S1]. Optionally, shape and appearance codes can be varied to change the volume density or color, respectively [S1].

### Formulas
The internal NeRF representation and rendering equation are identical to the original NeRF [S1]. Let $C(r)$ be the rendered color for a ray $r$ passing through sample points $t_i$:
$$C(r) = \sum_{i=1}^{N} T_i \left( 1 - \exp(-\sigma_i \delta_i) \right) c_i$$
Where:
*   $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the accumulated transmittance [S1].
*   $\sigma_i$ is the volume density predicted by the MLP at $t_i$ [S1].
*   $c_i$ is the color predicted by the MLP at $t_i$ [S1].
*   $\delta_i$ is the distance between sample points $t_i$ and $t_{i+1}$ [S1].

### Parameters
*   **MLP Layers:** 8 fully-connected layers [S1].
*   **MLP Channels:** 256 [S1].
*   **Activation:** ReLU [S1].
*   **Image Resolution:** 256 × 256 pixels (used in experiments for computational reasons) [S1].
*   **Typical Values:** Not given in the sources.

### Edge Cases and Pitfalls
*   **Overfitting:** Training for a high number of iterations leads to overfitting, resulting in a heavily under-regularized depth map with many floating artifacts in the background, which decreases the Intersection over Union (IoU) score [S1].
*   **Underfitting/Early Stopping:** Stopping training after a smaller number of iterations results in a more accurate depth map but a blurry, non-detailed RGB image [S1].
*   **Pose Ambiguity:** While GRAF does not require pose input, the predicted pose may not be precise, shifting the depth map from ground truth, although the shape may be more plausible [S1].
*   **Memory Constraints:** Predicting color for every pixel in the target image in the same batch is too memory-intensive, necessitating the patch-based approach [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid(x):
    return 1.0 / (1.0 + math.exp(-x))

def tanh(x):
    return math.tanh(x)

def relu(x):
    return max(0.0, x)

def fully_connected(x, weights, bias):
    return [sum(x[j] * weights[j][k] for j in range(len(x))) + bias[k] for k in range(len(bias))]

def concatenate_vectors(a, b):
    return a + b

def sample_batch(dataset, batch_size):
    return [dataset[i] for i in range(batch_size)]

def sample_poses(batch_size):
    return [(math.random.uniform(0, math.pi), math.random.uniform(0, 2 * math.pi)) for _ in range(batch_size)]

def sample_shape_codes(batch_size, code_dim):
    return [math.random.normal(0, 1, code_dim) for _ in range(batch_size)]

def sample_appearance_codes(batch_size, code_dim):
    return [math.random.normal(0, 1, code_dim) for _ in range(batch_size)]

def define_patch_grid(patch_size, num_pixels_per_side):
    grid = []
    for i in range(num_pixels_per_side):
        for j in range(num_pixels_per_side):
            grid.append((i * patch_size, j * patch_size))
    return grid

def cast_ray(pose, pixel_offset):
    theta, phi = pose
    origin = [0.0, 0.0, 0.0]
    direction = [math.sin(theta) * math.cos(phi) + pixel_offset[0], math.sin(theta) * math.sin(phi) + pixel_offset[1], math.cos(theta)]
    return {"origin": origin, "direction": direction}

def sample_ray(ray, num_samples, near, far):
    t_vals = [near + i * (far - near) / num_samples for i in range(num_samples)]
    samples = []
    for t in t_vals:
        pos = [ray["origin"][k] + t * ray["direction"][k] for k in range(3)]
        samples.append({"position": pos, "t": t})
    return samples

def mlp_forward(input_5d, shape_code, appearance_code, weights_list, biases_list):
    x = concatenate_vectors(input_5d, shape_code)
    x = concatenate_vectors(x, appearance_code)
    for i in range(8):
        x = fully_connected(x, weights_list[i], biases_list[i])
        if i < 7:
            x = [relu(v) for v in x]
    density = sigmoid(x[0])
    color = [tanh(v) for v in x[1:4]]
    return density, color

def volume_render(densities, colors, distances):
    transmittance = 1.0
    rendered_color = [0.0, 0.0, 0.0]
    for i in range(len(densities)):
        sigma = densities[i]
        delta = distances[i]
        c = colors[i]
        alpha = 1.0 - math.exp(-sigma * delta)
        weight = transmittance * alpha
        rendered_color = [rendered_color[k] + weight * c[k] for k in range(3)]
        transmittance *= (1.0 - alpha)
    return rendered_color

def compute_discriminator_loss(real_scores, fake_scores):
    return sum(real_scores) - sum(fake_scores)

def compute_generator_loss(fake_scores):
    return -sum(fake_scores)

def update_weights(model, loss, learning_rate):
    pass

def train_graf(dataset, generator, discriminator, num_iterations, learning_rate):
    receptive_field_size = 10.0
    num_pixels_per_side = 16
    batch_size = 4
    code_dim = 128
    near, far = 0.1, 10.0
    num_samples = 128

    for iteration in range(num_iterations):
        real_images = sample_batch(dataset, batch_size)
        poses = sample_poses(batch_size)
        shape_codes = sample_shape_codes(batch_size, code_dim)
        appearance_codes = sample_appearance_codes(batch_size, code_dim)

        generated_patches = []
        for i in range(batch_size):
            patch_pixels = define_patch_grid(receptive_field_size, num_pixels_per_side)
            rendered_patch = []
            for pixel_offset in patch_pixels:
                ray = cast_ray(poses[i], pixel_offset)
                samples = sample_ray(ray, num_samples, near, far)
                densities = []
                colors = []
                distances = []
                for k in range(len(samples) - 1):
                    densities.append(mlp_forward(samples[k]["position"], shape_codes[i], appearance_codes[i], generator.weights, generator.biases))
                    colors.append(mlp_forward(samples[k]["position"], shape_codes[i], appearance_codes[i], generator.weights, generator.biases))
                    distances.append(samples[k+1]["t"] - samples[k]["t"])
                rendered_patch.append(volume_render(densities, colors, distances))
            generated_patches.append(rendered_patch)

        real_scores = discriminator.forward(real_images)
        fake_scores = discriminator.forward(generated_patches)
        d_loss = compute_discriminator_loss(real_scores, fake_scores)
        update_weights(discriminator, d_loss, learning_rate)

        fake_scores_gen = discriminator.forward(generated_patches)
        g_loss = compute_generator_loss(fake_scores_gen)
        update_weights(generator, g_loss, learning_rate)

        receptive_field_size *= 0.99

def inference_graf(generator, pose, shape_code, appearance_code, patch_size, num_pixels):
    patch_pixels = define_patch_grid(patch_size, num_pixels)
    rendered_patch = []
    for pixel_offset in patch_pixels:
        ray = cast_ray(pose, pixel_offset)
        samples = sample_ray(ray, 128, 0.1, 10.0)
        densities = []
        colors = []
        distances = []
        for k in range(len(samples) - 1):
            densities.append(mlp_forward(samples[k]["position"], shape_code, appearance_code, generator.weights, generator.biases))
            colors.append(mlp_forward(samples[k]["position"], shape_code, appearance_code, generator.weights, generator.biases))
            distances.append(samples[k+1]["t"] - samples[k]["t"])
        rendered_patch.append(volume_render(densities, colors, distances))
    return rendered_patch
```

## Subtopics

### HybridNeRF Model Architecture
Combines voxel grids with neural radiance fields for efficiency. Choose when memory is constrained but high fidelity is needed. [HybridNeRF Model Architecture](4842-hybridnerf-model-architecture.md)

### MVS Network Query
Uses multi-view stereo features to initialize or condition the NeRF. Choose when sparse multi-view images are available. [MVS Network Query](7517-mvs-network-query.md)

### SqueezeNeRF
Compresses NeRF models for mobile deployment. Choose for real-time applications on edge devices. [SqueezeNeRF](8704-squeezenerf.md)

### VoxNeRF
Uses voxel grids for spatial hashing to accelerate NeRF queries. Choose for faster training/inference with large scenes. [VoxNeRF](8865-voxnerf.md)

### Deformation Field Integration for Telepresence
Adds time-varying deformation fields to NeRF. Choose for dynamic scenes. [Deformation Field Integration for Telepresence](10866-deformation-field-integration-for-telepresence.md)

### HDR-HexPlane
Uses hexagonal planes for high-dynamic-range rendering. Choose for scenes with extreme lighting variations. [HDR-HexPlane](13548-hdr-hexplane.md)

### Multi-scale PointNeRF++ Feature Aggregation
Aggregates features from multiple scales of point clouds. Choose for scenes with sparse geometry. [Multi-scale PointNeRF++ Feature Aggregation](13651-multi-scale-pointnerf-feature-aggregation.md)

### FourieRF
Uses Fourier features for frequency bias. Choose for high-frequency detail reconstruction. [FourieRF](15185-fourierf.md)

### Appearance Matching
Matches appearance across views using consistency losses. Choose for unposed datasets with consistent lighting. [Appearance Matching](15579-appearance-matching.md)

### Hash-grid anti-aliasing and regularization
Applies anti-aliasing to hash grids. Choose to reduce artifacts in high-resolution renders. [Hash-grid anti-aliasing and regularization](15594-hash-grid-anti-aliasing-and-regularization.md)

### NeRF-W
Handles unposed images with weather conditions. Choose for outdoor scenes with varying lighting. [NeRF-W](15666-nerf-w.md)

### UniSDF-based Latent NeRF
Uses signed distance fields for better geometry. Choose when accurate surface reconstruction is critical. [UniSDF-based Latent NeRF](17022-unisdf-based-latent-nerf.md)

### RawNeRF
Processes raw sensor data. Choose for high-fidelity rendering from unprocessed camera data. [RawNeRF](17244-rawnerf.md)

### Neural Duplex Radiance Field
Uses bidirectional rendering. Choose for view-dependent effects like specularities. [Neural Duplex Radiance Field](17643-neural-duplex-radiance-field.md)

### NeuRay Occlusion-Aware Parameterization
Handles occlusions explicitly. Choose for scenes with complex occlusions. [NeuRay Occlusion-Aware Parameterization](18695-neuray-occlusion-aware-parameterization.md)

### SHINE-Mapping
Maps lighting conditions. Choose for consistent lighting across views. [SHINE-Mapping](20278-shine-mapping.md)

### NeRF-LiDAR
Integrates LiDAR data. Choose when depth sensors are available. [NeRF-LiDAR](20283-nerf-lidar.md)

### DINER (Disorder-Invariant Implicit Neural Representation)
Invariant to point cloud disorder. Choose for unordered point cloud inputs. [DINER (Disorder-Invariant Implicit Neural Representation)](21718-diner-disorder-invariant-implicit-neural-representation.md)

## Operating characteristics

*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources, but the patch-based approach reduces memory usage compared to full-image prediction [S1].
*   **Measured Performance:** On CubeSat and SMOS satellite datasets, GRAF renders precise novel views with accurate details even when parts of the satellites are shadowed, whereas NeRF may struggle with dark scenes [S1]. NeRF generally renders more accurate images regarding material specularity and pose [S1]. GRAF's depth prediction captures shape accurately but the pose may be imprecise, shifting the depth map from ground truth [S1].
*   **Scaling:** The patch-based approach allows application to high-resolution images by avoiding memory-intensive full-image prediction [S1].
*   **Strengths:** Does not require camera pose information; disentangles shape and appearance; handles dark scenes better than NeRF; allows explicit camera control [S1].
*   **Limitations:** Prone to overfitting with high iteration counts (leading to floating artifacts in depth maps); early stopping leads to blurry RGB images; pose estimation is imprecise, shifting depth maps from ground truth, though shape capture remains plausible [S1].
*   **When to use:** When camera poses are unknown (unposed images); when dealing with dark or shadowed scenes; when disentangling shape and appearance is desired [S1].
*   **When not to use:** When high-fidelity material specularity is critical (NeRF may be better); when full-image rendering is necessary and memory is not constrained [S1].

## Background

Generative Radiance Fields (GRAF) was introduced by Schwarz et al. [S1]. It is based on the original NeRF paper and extends it with an adversarial training scheme [S1]. The method was applied to space situational awareness, specifically for identifying non-cooperative spacecraft from unposed images [S1]. The authors note that while other 3D-aware GANs use voxel grids, GRAF uses a NeRF as a 3D implicit representation, allowing it to learn without pose information [S1]. The method was evaluated on synthetic datasets of satellites generated using Unity [S1].

## Sources

[S1] Vision-based Neural Scene Representations for Spacecraft, https://doi.org/10.1109/cvprw53098.2021.00228, fetched 2026-10-26
