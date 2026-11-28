# Neural Refractive Radiance Field (NeRRF)

Also called: IOR-Adaptive Refraction Ray Tracing, Index of Refraction (IOR) Adaptive Neural Refractive Radiance Field, Inverse Rendering with IOR Optimization, Mask-based Ray Tracing Acceleration

## Overview
Neural Refractive Radiance Field (NeRRF) is an inverse rendering method for transparent scenes that reconstructs surface geometry and a radiance field from multi-view images without prior knowledge of the object's Index of Refraction (IOR) [S1]. It solves the problem of incorrect radiance field learning and visual artifacts (such as holes and color errors) that occur when standard NeRF or SDF-based methods like NeuS attempt to model refraction with a fixed or unknown IOR [S1]. NeRRF distinguishes itself by automatically searching for the optimal IOR during training and correcting the calculation of refractive effects, specifically enabling accurate rendering of foreground darkening caused by refraction at object edges [S1].

## How it works
The method consists of three phases: surface reconstruction, background radiance field training with IOR optimization, and complete radiance field training.

### Inputs and Outputs
*   **Inputs:** A set of multi-view images of transparent objects, camera poses, and object masks (extracted via SAM) [S1].
*   **Outputs:** A reconstructed mesh (surface geometry), a trained radiance field, and the optimal Index of Refraction (IOR) [S1].

### Data Structures
1.  **Surface Representation:** A tetrahedral mesh represented by DMTet [S1].
2.  **Neural Network:** A multi-layer perceptron (MLP) taking position encoding and viewing direction as input, outputting volume density ($\sigma$) and color ($c$) [S1].
3.  **Position Encoding:** Six-channel Fourier frequency position encoding, gradually activated during training [S1].
4.  **Loss Function:** A composite loss including color loss for coarse and fine sampling [S1].

### Numbered Steps
1.  **Data Preprocessing:** Extract object masks from input images using a pre-trained Segment Anything Model (SAM) [S1]. Filter masks to select the top-3 matching masks for each image [S1].
2.  **Surface Reconstruction (DMTet):** Initialize the DMTet mesh and train surface geometry using the extracted masks and camera poses [S1]. The network predicts Signed Distance Field (SDF) values at tetrahedron vertices [S1]. Training uses a mask loss and an Eikonal loss. The Eikonal loss coefficient varies by object detail: high-frequency objects (e.g., rabbit, cow) use 0.01 for the first two epochs and 0.001 for the last epoch; low-frequency objects (e.g., sphere) use 0.1 for the first two epochs and 0.001 for the last epoch [S1]. The schedule is 3 epochs, each traversing the dataset 50 times with a batch size of 640 pixels sampled randomly [S1]. The learning rate is 0.01 [S1].
3.  **Background Radiance Field Training:** Train a radiance field for the background using the masks to separate foreground and background [S1]. Assume ambient light is at infinite distance; color depends only on viewing direction [S1]. Represent color and volume density as $F_{\sigma,c}(d) \rightarrow (\sigma, c)$ [S1].
4.  **IOR Optimization:** Define an initial IOR search interval of $[1, 2.5]$ with 10 search intervals [S1]. For each epoch of background radiance field training, render the foreground using the trained background radiance field and the current set of candidate IORs [S1]. Perform pixel-by-pixel ray tracing to modify light paths based on the reconstructed surface [S1]. Calculate the intersection of each ray with the transparent object. For rays not intersecting the object, render directly using the background radiance field [S1]. For rays intersecting the object, calculate refraction; the object is assumed colorless, so the color is determined by the refracted path through the background [S1]. Compute the Peak Signal-to-Noise Ratio (PSNR) between the rendered foreground and the ground truth image for each candidate IOR [S1]. Select the IOR that maximizes the PSNR as the optimal IOR [S1]. This search is conducted every epoch during background radiance field training [S1].
5.  **Complete Radiance Field Training:** Train the complete radiance field (foreground and background) using the optimal IOR found in Step 4 [S1]. The total loss $L$ is the sum of color losses for coarse sampling ($\hat{c}_c$) and fine sampling ($\hat{c}_f$) compared to ground truth color $c(r)$:
    $$ L = \sum_{r \in R} \left( \hat{c}_c(r) - c(r) \right)^2 + \left( \hat{c}_f(r) - c(r) \right)^2 $$
    where coefficients for both coarse and fine sampling loss terms are set to 1 [S1]. Training schedules are: Blender dataset uses 4 epochs, 50 traversals per epoch, batch size 640; Real dataset uses 8 epochs, 100 traversals per epoch [S1]. The optimizer is Adam with learning rate 0.01 [S1]. After background training, the foreground radiance field is trained for one additional epoch [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class DMTetSurface(nn.Module):
    """Predicts Signed Distance Field (SDF) values at tetrahedron vertices [S1]."""
    def __init__(self):
        super().__init__()
        # Six-channel Fourier frequency position encoding, gradually activated [S1]
        self.encoder = nn.Sequential(
            nn.Linear(3, 64),
            nn.ReLU(),
            nn.Linear(64, 1)
        )
    def forward(self, points):
        return self.encoder(points)

class RadianceFieldMLP(nn.Module):
    """Predicts volume density (sigma) and color (c) given position and viewing direction [S1]."""
    def __init__(self):
        super().__init__()
        self.sigma_net = nn.Sequential(nn.Linear(3, 64), nn.ReLU(), nn.Linear(64, 1))
        self.color_net = nn.Sequential(nn.Linear(3 + 64, 64), nn.ReLU(), nn.Linear(64, 3))
    def forward(self, positions, directions):
        sigma = self.sigma_net(positions)
        h = torch.cat([directions, sigma], dim=-1)
        color = self.color_net(h)
        return sigma, color

def refract_ray(ray_direction, normal, ior):
    """Refracts ray according to Snell's law [S2]."""
    eta = 1.0 / ior
    cos_theta_i = -torch.dot(ray_direction, normal)
    sin_theta_i_sq = 1.0 - cos_theta_i**2
    if sin_theta_i_sq > eta**2:
        return ray_direction
    sin_theta_t_sq = eta**2 * sin_theta_i_sq
    cos_theta_t = torch.sqrt(1.0 - sin_theta_t_sq)
    refracted = eta * ray_direction + (eta * cos_theta_i - cos_theta_t) * normal
    return refracted / torch.norm(refracted)

def volume_rendering(ray_origin, ray_direction, sigma, color, num_samples):
    """Standard volumetric rendering corresponding to Equation (5) in the source [S1]."""
    # Standard alpha compositing loop over samples along ray
    # Returns rendered color c(r)
    pass # Standard volume rendering formula [S1]

def train_neRRF(images, masks, camera_poses, dataset_type='blender'):
    # Initialize models
    surface_model = DMTetSurface()
    radiance_field = RadianceFieldMLP()
    surface_optimizer = optim.Adam(surface_model.parameters(), lr=0.01)
    radiance_optimizer = optim.Adam(radiance_field.parameters(), lr=0.01)
    
    # IOR Search Parameters [S1]
    ior_search_intervals = torch.linspace(1.0, 2.5, 10)
    
    # Phase 1: Surface Reconstruction [S1]
    for epoch in range(3):
        for _ in range(50):
            batch_pixels = sample_random_pixels(images, batch_size=640)
            batch_masks = sample_random_pixels(masks, batch_size=640)
            sdf_values = surface_model(batch_pixels)
            mask_loss = calculate_mask_loss(sdf_values, batch_masks)
            
            # Eikonal loss coefficient [S1]
            if epoch < 2:
                eikonal_coeff = 0.01 if dataset_type == 'high_freq' else 0.1
            else:
                eikonal_coeff = 0.001
            eikonal_loss = calculate_eikonal_loss(sdf_values) * eikonal_coeff
            total_loss = mask_loss + eikonal_loss
            
            surface_optimizer.zero_grad()
            total_loss.backward()
            surface_optimizer.step()
            
    # Phase 2: Background Radiance Field Training & IOR Optimization [S1]
    optimal_ior = None
    max_psnr = -float('inf')
    
    num_epochs = 4 if dataset_type == 'blender' else 8
    traversals = 50 if dataset_type == 'blender' else 100
    
    for epoch in range(num_epochs):
        # IOR Search every epoch [S1]
        best_ior_epoch = None
        best_psnr_epoch = -float('inf')
        
        for ior in ior_search_intervals:
            psnr_score = evaluate_ior(radiance_field, surface_model, ior, images, masks, camera_poses)
            if psnr_score > best_psnr_epoch:
                best_psnr_epoch = psnr_score
                best_ior_epoch = ior
                
        if best_psnr_epoch > max_psnr:
            max_psnr = best_psnr_epoch
            optimal_ior = best_ior_epoch
            
        # Train Radiance Field [S1]
        for _ in range(traversals):
            batch_pixels = sample_random_pixels(images, batch_size=640)
            rendered_color = render_radiance_field(radiance_field, surface_model, batch_pixels, optimal_ior if optimal_ior else ior_search_intervals[0])
            ground_truth_color = images[batch_pixels]
            
            # Loss Function (Equation 6) [S1]
            loss = torch.mean((rendered_color - ground_truth_color)**2)
            radiance_optimizer.zero_grad()
            loss.backward()
            radiance_optimizer.step()
            
    # Phase 3: Final Foreground Radiance Field Training [S1]
    for _ in range(1):
        for _ in range(50):
            batch_pixels = sample_random_pixels(images, batch_size=640)
            rendered_color = render_radiance_field(radiance_field, surface_model, batch_pixels, optimal_ior)
            ground_truth_color = images[batch_pixels]
            loss = torch.mean((rendered_color - ground_truth_color)**2)
            radiance_optimizer.zero_grad()
            loss.backward()
            radiance_optimizer.step()
            
    return optimal_ior

def evaluate_ior(radiance_field, surface_model, ior, images, masks, camera_poses):
    """Evaluate PSNR for a specific IOR using pixel-by-pixel ray tracing [S1]."""
    total_psnr = 0
    num_pixels = len(images)
    
    for image, mask, pose in zip(images, masks, camera_poses):
        rays = generate_rays(pose, image.shape)
        rendered_image = torch.zeros_like(image)
        
        for j, ray in enumerate(rays):
            intersection = surface_model.get_intersection(ray)
            if intersection is None:
                # No intersection: render directly using background radiance field [S1]
                color = radiance_field.render_background(ray.direction)
            else:
                # Intersection: refract ray based on surface [S1]
                refracted_ray = refract_ray(ray.direction, intersection.normal, ior)
                color = radiance_field.render_background(refracted_ray.direction)
            rendered_image[j] = color
            
        total_psnr += calculate_psnr(rendered_image, image)
        
    return total_psnr / num_pixels

def render_radiance_field(radiance_field, surface_model, pixels, ior):
    """Render radiance field with coarse and fine sampling [S1]."""
    # Returns rendered color using Equation (6) loss components
    pass # Standard volume rendering with coarse/fine sampling [S1]

def sample_random_pixels(images, batch_size):
    # Sample random pixel coordinates
    pass

def calculate_mask_loss(sdf_values, masks):
    # Calculate loss between predicted SDF and mask
    pass

def calculate_eikonal_loss(sdf_values):
    # Calculate Eikonal loss |grad(SDF)| - 1
    pass

def calculate_psnr(rendered, ground_truth):
    # Calculate PSNR
    pass

def generate_rays(pose, shape):
    # Generate rays from camera pose
    pass
```

### Parameters
*   **Position Encoding:** Six-channel Fourier frequency, gradually activated [S1].
*   **Surface Reconstruction Learning Rate:** 0.01 [S1].
*   **Surface Reconstruction Epochs:** 3 [S1].
*   **Surface Reconstruction Traversals:** 50 per epoch [S1].
*   **Surface Reconstruction Batch Size:** 640 pixels [S1].
*   **Eikonal Loss Coefficient (High-Freq):** 0.01 (epochs 1-2), 0.001 (epoch 3) [S1].
*   **Eikonal Loss Coefficient (Low-Freq):** 0.1 (epochs 1-2), 0.001 (epoch 3) [S1].
*   **Radiance Field Learning Rate:** 0.01 [S1].
*   **Radiance Field Loss Coefficients:** 1 for coarse sampling, 1 for fine sampling [S1].
*   **Blender Dataset Radiance Field Epochs:** 4 [S1].
*   **Blender Dataset Radiance Field Traversals:** 50 per epoch [S1].
*   **Blender Dataset Radiance Field Batch Size:** 640 pixels [S1].
*   **Real Dataset Radiance Field Epochs:** 8 [S1].
*   **Real Dataset Radiance Field Traversals:** 100 per epoch [S1].
*   **IOR Search Interval:** $[1, 2.5]$ [S1].
*   **IOR Search Intervals Count:** 10 [S1].
*   **IOR Search Frequency:** Every epoch during background radiance field training [S1].

### Edge Cases and Pitfalls
*   **NeuS Failure:** Standard SDF-based methods like NeuS fail on glass datasets because refraction causes the network to learn the wrong Signed Distance Field (SDF), resulting in holes and color errors on the object's surface [S1].
*   **Mask Deviation:** There may be a deviation of several pixels at the edges of the extracted mask, but reliable reconstruction results are still obtained [S1].
*   **Refraction Artifacts:** Standard NeRF suffers from artifacts and wrong radiance field learning in transparent scenes. NeRRF corrects inaccurate calculation of refractive effects, specifically enabling more accurate rendering of foreground darkening caused by refraction at edges [S1].

## Subtopics

### NeRFrac
NeRFrac tackles NeRF's constraints in handling refractive surfaces, like subaquatic environments, through the integration of a multilayer perceptron (MLP)-based refractive field [S2]. It calculates the gap between a ray and its refracted counterpart, calculates the refracted ray in line with Snell's law, and ensures uniformity across multiple views [S2]. The method successfully generates scenes devoid of refraction impacts by eliminating the acquired refraction field, showcasing its proficiency in creating and reconstructing underwater scenes [S2].

## Operating characteristics
*   **Performance:** The mask-based ray-tracing acceleration algorithm reduces training time by approximately 60% and rendering time by 75% [S1].
*   **Hardware:** Implemented in PyTorch 1.10.0 on a single NVIDIA A40 GPU [S1].
*   **Strengths:** Automatically searches for the best IOR during training, ensuring accuracy without needing an accurate initial IOR value. It performs complete inverse rendering to obtain ambient light, geometry, and IOR, enabling downstream tasks like object and environment editing [S1].
*   **Limitations:** Novel view synthesis speeds are extremely slow compared to non-neural methods [S1]. It relies on accurate masks, which are extracted using SAM and may have edge deviations [S1].

## Background
*   **Origin:** The method is based on NeRF [S1]. The specific IOR-adaptive variant was proposed in a 2025 paper published in *Electronics* [S1].
*   **History:** Standard NeRF and SDF-based methods like NeuS struggle with transparent objects due to refraction. NeRRF was introduced to handle refraction, but this work improves upon it by adding automatic IOR estimation and correcting refractive effect calculations [S1].
*   **Usage:** Used for surface reconstruction and radiance field reconstruction of transparent objects from multi-view images, enabling novel view synthesis and inverse rendering tasks [S1].

## Sources
[S1] An Index of Refraction Adaptive Neural Refractive Radiance Field for Transparent Scenes, https://doi.org/10.3390/electronics14061073, fetched 2026-10-29
[S2] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
