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

2.  **Surface Reconstruction (DMTet):**
    *   Initialize the DMTet mesh.
    *   Train the surface geometry using the extracted masks and camera poses [S1].
    *   The network predicts the Signed Distance Field (SDF) values at tetrahedron vertices [S1].
    *   **Loss:** Includes a mask loss and an Eikonal loss. The Eikonal loss coefficient varies by object detail:
        *   High-frequency objects (e.g., rabbit, cow): 0.01 for the first two epochs, 0.001 for the last epoch [S1].
        *   Low-frequency objects (e.g., sphere): 0.1 for the first two epochs, 0.001 for the last epoch [S1].
    *   **Training Schedule:** 3 epochs, each traversing the dataset 50 times, batch size of 640 pixels sampled randomly [S1].
    *   **Optimizer:** Learning rate 0.01 [S1].

3.  **Background Radiance Field Training:**
    *   Train a radiance field for the background (environment) using the masks to separate foreground and background [S1].
    *   Assume ambient light is at infinite distance; color depends only on viewing direction [S1].
    *   Represent color and volume density as $F_{\sigma,c}(d) \rightarrow (\sigma, c)$ [S1].

4.  **IOR Optimization:**
    *   Define an initial IOR search interval of $[1, 2.5]$ with 10 search intervals [S1].
    *   For each epoch of background radiance field training:
        *   Render the foreground using the trained background radiance field and the current set of candidate IORs [S1].
        *   Perform pixel-by-pixel ray tracing to modify light paths based on the reconstructed surface [S1].
        *   Calculate the intersection of each ray with the transparent object.
        *   For rays **not** intersecting the object: render directly using the background radiance field [S1].
        *   For rays **intersecting** the object: calculate refraction. The object is assumed colorless; the color is determined by the refracted path through the background [S1].
        *   Compute the Peak Signal-to-Noise Ratio (PSNR) between the rendered foreground and the ground truth image for each candidate IOR [S1].
        *   Select the IOR that maximizes the PSNR as the optimal IOR [S1].
    *   This search is conducted every epoch during background radiance field training [S1].

5.  **Complete Radiance Field Training:**
    *   Train the complete radiance field (foreground and background) using the optimal IOR found in Step 4 [S1].
    *   **Loss Function:** The total loss $L$ is the sum of color losses for coarse sampling ($\hat{c}_c$) and fine sampling ($\hat{c}_f$) compared to ground truth color $c(r)$:
        $$ L = \sum_{r \in R} \left( \hat{c}_c(r) - c(r) \right)^2 + \left( \hat{c}_f(r) - c(r) \right)^2 $$
        where coefficients for both coarse and fine sampling loss terms are set to 1 [S1].
    *   **Training Schedule:**
        *   Blender dataset: 4 epochs, 50 traversals per epoch, batch size 640 [S1].
        *   Real dataset: 8 epochs, 100 traversals per epoch [S1].
    *   **Optimizer:** Adam with learning rate 0.01 [S1].
    *   **Final Step:** After background training, train the foreground radiance field for one additional epoch [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

# Constants
NUM_IOR_SEARCH_INTERVALS = 10
IOR_MIN = 1.0
IOR_MAX = 2.5
BATCH_SIZE = 640
LEARNING_RATE = 0.01

class NeRRF:
    def __init__(self, images, masks, camera_poses, dataset_type='blender'):
        self.images = images
        self.masks = masks
        self.camera_poses = camera_poses
        self.dataset_type = dataset_type
        
        # Initialize DMTet surface representation
        self.surface_model = DMTetModel()
        
        # Initialize Radiance Field MLP
        self.radiance_field = RadianceFieldMLP()
        
        # Optimizers
        self.surface_optimizer = optim.Adam(self.surface_model.parameters(), lr=0.01)
        self.radiance_optimizer = optim.Adam(self.radiance_field.parameters(), lr=0.01)
        
        # IOR Search Parameters
        self.ior_search_intervals = torch.linspace(IOR_MIN, IOR_MAX, NUM_IOR_SEARCH_INTERVALS)
        
    def train_surface(self, num_epochs=3, traversals_per_epoch=50):
        """Phase 1: Surface Reconstruction using DMTet"""
        for epoch in range(num_epochs):
            for _ in range(traversals_per_epoch):
                # Sample batch of pixels
                batch_pixels = sample_random_pixels(self.images, batch_size=BATCH_SIZE)
                batch_masks = sample_random_pixels(self.masks, batch_size=BATCH_SIZE)
                
                # Forward pass through DMTet
                sdf_values = self.surface_model(batch_pixels)
                
                # Calculate Loss
                mask_loss = calculate_mask_loss(sdf_values, batch_masks)
                
                # Eikonal loss coefficient depends on object detail and epoch
                if epoch < 2:
                    if self.dataset_type == 'high_freq': # e.g., rabbit, cow
                        eikonal_coeff = 0.01
                    else: # e.g., sphere
                        eikonal_coeff = 0.1
                else:
                    eikonal_coeff = 0.001
                
                eikonal_loss = calculate_eikonal_loss(sdf_values) * eikonal_coeff
                total_loss = mask_loss + eikonal_loss
                
                # Backward pass
                self.surface_optimizer.zero_grad()
                total_loss.backward()
                self.surface_optimizer.step()

    def train_radiance_field_with_ior_optimization(self, num_epochs=4, traversals_per_epoch=50):
        """Phase 2: Background Radiance Field Training and IOR Search"""
        optimal_ior = None
        max_psnr = -float('inf')
        
        for epoch in range(num_epochs):
            # IOR Search
            best_ior_epoch = None
            best_psnr_epoch = -float('inf')
            
            for ior in self.ior_search_intervals:
                psnr_score = self.evaluate_ior(ior)
                if psnr_score > best_psnr_epoch:
                    best_psnr_epoch = psnr_score
                    best_ior_epoch = ior
            
            if best_psnr_epoch > max_psnr:
                max_psnr = best_psnr_epoch
                optimal_ior = best_ior_epoch
            
            # Train Radiance Field with current best IOR (or average if not selected yet)
            # Note: The source says "conducted an IOR search for every epoch during the training 
            # of the background radiance field". It implies the radiance field is trained 
            # alongside or after the search. We train the radiance field here.
            
            for _ in range(traversals_per_epoch):
                batch_pixels = sample_random_pixels(self.images, batch_size=BATCH_SIZE)
                batch_masks = sample_random_pixels(self.masks, batch_size=BATCH_SIZE)
                
                # Render using current IOR (using optimal_ior if found, else current search point)
                # For training, we might use the current best or iterate. 
                # The source implies the search happens *during* training.
                # We assume we train the radiance field using the optimal_ior found so far.
                
                rendered_color = self.render_radiance_field(batch_pixels, optimal_ior if optimal_ior else self.ior_search_intervals[0])
                ground_truth_color = self.images[batch_pixels]
                
                # Loss Function (Equation 6)
                # L = sum( (c_c - c)^2 + (c_f - c)^2 )
                # Assuming coarse and fine sampling are handled in render_radiance_field
                loss = torch.mean((rendered_color - ground_truth_color)**2)
                
                self.radiance_optimizer.zero_grad()
                loss.backward()
                self.radiance_optimizer.step()
        
        # Final foreground radiance field training for one epoch
        self.train_foreground_radiance_field(optimal_ior, num_epochs=1, traversals_per_epoch=50)
        
        return optimal_ior

    def evaluate_ior(self, ior):
        """Evaluate PSNR for a specific IOR"""
        # Perform pixel-by-pixel ray tracing
        # 1. Calculate intersection of ray with transparent object
        # 2. If no intersection, render directly using background radiance field
        # 3. If intersection, modify light path based on surface and IOR
        
        total_psnr = 0
        num_pixels = len(self.images)
        
        for i, (image, mask, pose) in enumerate(zip(self.images, self.masks, self.camera_poses)):
            rays = generate_rays(pose, image.shape)
            rendered_image = torch.zeros_like(image)
            
            for j, ray in enumerate(rays):
                intersection = self.surface_model.get_intersection(ray)
                
                if intersection is None:
                    # No intersection: render background
                    color = self.radiance_field.render_background(ray.direction)
                else:
                    # Intersection: refract ray
                    refracted_ray = self.refract_ray(ray, intersection.normal, ior)
                    # Render along refracted ray
                    color = self.radiance_field.render_background(refracted_ray.direction)
                
                rendered_image[j] = color
            
            # Calculate PSNR
            psnr = calculate_psnr(rendered_image, image)
            total_psnr += psnr
            
        return total_psnr / num_pixels

    def train_foreground_radiance_field(self, ior, num_epochs=1, traversals_per_epoch=50):
        """Phase 3: Final Training with Optimal IOR"""
        for _ in range(num_epochs):
            for _ in range(traversals_per_epoch):
                batch_pixels = sample_random_pixels(self.images, batch_size=BATCH_SIZE)
                
                # Render with optimal IOR
                rendered_color = self.render_radiance_field(batch_pixels, ior)
                ground_truth_color = self.images[batch_pixels]
                
                loss = torch.mean((rendered_color - ground_truth_color)**2)
                
                self.radiance_optimizer.zero_grad()
                loss.backward()
                self.radiance_optimizer.step()

    def render_radiance_field(self, pixels, ior):
        """Render radiance field with coarse and fine sampling"""
        # Implementation of Equation 6 loss components
        # Returns rendered color
        pass

    def refract_ray(self, ray, normal, ior):
        """Refract ray based on Snell's Law"""
        # Implementation of refraction
        pass

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

## Operating characteristics
*   **Performance:** The mask-based ray-tracing acceleration algorithm reduces training time by approximately 60% and rendering time by 75% [S1].
*   **Hardware:** Implemented in PyTorch 1.10.0 on a single NVIDIA A40 GPU [S1].
*   **Strengths:** Automatically searches for the best IOR during training, ensuring accuracy without needing an accurate initial IOR value. It performs complete inverse rendering to obtain ambient light, geometry, and IOR, enabling downstream tasks like object and environment editing [S1].
*   **Limitations:** Novel view synthesis speeds are extremely slow compared to non-neural methods [S1]. It relies on accurate masks, which are extracted using SAM and may have edge deviations [S1].

## Background
*   **Origin:** The method is based on NeRRF [S1]. The specific IOR-adaptive variant was proposed in a 2025 paper published in *Electronics* [S1].
*   **History:** Standard NeRF and SDF-based methods like NeuS struggle with transparent objects due to refraction. NeRRF was introduced to handle refraction, but this work improves upon it by adding automatic IOR estimation and correcting refractive effect calculations [S1].
*   **Usage:** Used for surface reconstruction and radiance field reconstruction of transparent objects from multi-view images, enabling novel view synthesis and inverse rendering tasks [S1].

## Sources
[S1] An Index of Refraction Adaptive Neural Refractive Radiance Field for Transparent Scenes, https://doi.org/10.3390/electronics14061073, fetched 2026-10-29
