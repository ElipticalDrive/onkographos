# VolSDF Model Architecture

Also called: VolSDF Training, Volume Rendering of Neural Implicit Surfaces, Volume Rendering of Neural Implicit Surfaces training

## Overview
VolSDF is a volume rendering framework for neural implicit surfaces that represents volume density as a transformed version of the signed distance function (SDF) to the learned surface geometry, allowing disentanglement of geometry (density) and radiance field [S1, unofficial]. Unlike IDR, which requires object masks and sets pixel color based on a single intersection point leading to extraneous surface parts, VolSDF does not require masks and uses the volume rendering integral to avoid this issue [S1, unofficial]. It builds on [Model Architecture and Encoding](13550-model-architecture-and-encoding.md) by replacing the standard density prediction with an SDF-based density transformation.

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of images with camera parameters. For each pixel $p$, the input triplet is $(I_p, \mathbf{c}_p, \mathbf{v}_p)$, where $I_p \in \mathbb{R}^3$ is the intensity (RGB), $\mathbf{c}_p \in \mathbb{R}^3$ is the camera location, and $\mathbf{v}_p \in \mathbb{R}^3$ is the viewing direction (from camera to pixel) [S1, unofficial].
*   **Outputs:** Rendered RGB images and a reconstructed 3D surface geometry defined by the SDF.

**Data Structures**
The model consists of two Multi-Layer Perceptrons (MLPs) and two scalar parameters:
1.  **Geometry MLP ($f_\varphi$):** Approximates the SDF $d(\mathbf{x})$ and a global geometry feature vector $\mathbf{z}$.
    *   Input: Position $\mathbf{x} \in \mathbb{R}^3$.
    *   Output: Tuple $(d(\mathbf{x}), \mathbf{z}(\mathbf{x})) \in \mathbb{R}^{1+256}$. The feature vector $\mathbf{z}$ has dimension 256 [S1, unofficial].
    *   Parameters: $\varphi$.
2.  **Radiance Field MLP ($L_\psi$):** Represents the scene's radiance field.
    *   Input: Position $\mathbf{x}$, surface normal $\mathbf{n}$, viewing direction $\mathbf{v}$, and geometry feature $\mathbf{z}$.
    *   Output: RGB color $\mathbf{c} \in \mathbb{R}^3$.
    *   Parameters: $\psi$.
3.  **Scalar Parameters:** Two learnable scalars $\alpha, \beta \in \mathbb{R}$. In implementation, $\alpha = \beta^{-1}$ is enforced [S1, unofficial].
4.  **Total Parameters:** $\theta = (\varphi, \psi, \beta)$ [S1, unofficial].

**Positional Encoding**
Positional encoding [25] is exploited for the position $\mathbf{x}$ and view direction $\mathbf{v}$ in both the geometry and radiance field MLPs, following the shared principles of the parent article [S1, unofficial].

**Density Transformation**
The volume density $\sigma(\mathbf{x})$ is derived from the SDF $d(\mathbf{x})$ predicted by $f_\varphi$. The specific transformation formula is not explicitly detailed in the provided text, but it is defined as a "transformed version of the signed distance function" [S1, unofficial]. The scalar parameter $\beta$ (and $\alpha$) controls this transformation [S1, unofficial].

**Rendering**
Rendering follows the standard volume rendering integral using samples along the ray. The radiance $L_i$ at sample point $s_i$ is computed as:
$$ L_i = L_\psi(\mathbf{x}(s_i), \mathbf{n}(s_i), \mathbf{v}_p, \mathbf{z}(\mathbf{x}(s_i))) $$
where $\mathbf{n}(s_i)$ is the normal derived from the gradient of the SDF [S1, unofficial].

**Training Procedure**
The model is trained using a batch size of 1024 pixels [S1, unofficial]. The loss function includes:
1.  **Photometric Loss:** Comparing rendered images to ground truth $I_p$.
2.  **SDF Loss ($\mathcal{L}_{\text{SDF}}$):** An Eikonal loss encouraging $d$ to approximate a signed distance function [S1, unofficial].
3.  **Sampling Strategy:** Samples $\mathbf{z}$ for the Eikonal loss are taken from a combination of a single random uniform space point and a single point from the set $\mathcal{S}$ (samples along the ray) for each pixel $p$ [S1, unofficial].
4.  **Hyperparameters:** The weight $\lambda$ for the SDF loss is set to 0.1 [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_volsdf(images, cameras, num_epochs=1000, batch_size=1024, lambda_sdf=0.1):
    # Initialize models
    f_phi = GeometryMLP() # Outputs (d, z)
    L_psi = RadianceMLP() # Outputs color
    beta = Parameter(torch.tensor(1.0)) # Scalar parameter
    alpha = 1.0 / beta # Enforce alpha = beta^-1
    
    optimizer = Adam([f_phi.parameters(), L_psi.parameters(), beta])
    
    for epoch in range(num_epochs):
        # Sample batch of pixels
        batch_pixels = sample_pixels(images, batch_size)
        
        total_loss = 0
        for pixel in batch_pixels:
            I_gt = pixel.intensity
            c_cam = pixel.camera_location
            v_view = pixel.viewing_direction
            
            # Generate ray and sample points
            ray_o, ray_d = generate_ray(c_cam, v_view)
            samples_x, samples_t = sample_along_ray(ray_o, ray_d)
            
            # Forward pass through Geometry MLP
            # Note: Positional encoding is applied to samples_x and v_view
            d_vals, z_vals = f_phi(samples_x)
            
            # Compute normals from SDF gradient (not shown in detail, [G])
            normals = compute_sdf_gradient(d_vals, samples_x)
            
            # Compute density from SDF using beta
            # Formula not explicitly given, but d is transformed [S1, unofficial]
            sigmas = transform_sdf_to_density(d_vals, beta)
            
            # Forward pass through Radiance MLP
            colors = L_psi(samples_x, normals, v_view, z_vals)
            
            # Volume rendering to get rendered color
            I_render = volume_render(samples_t, sigmas, colors)
            
            # Photometric loss
            loss_photo = mse(I_render, I_gt)
            
            # SDF Loss (Eikonal)
            # Requires combining random uniform point and ray sample
            random_point = sample_uniform_in_bounds()
            d_random, z_random = f_phi(random_point)
            grad_random = compute_sdf_gradient(d_random, random_point)
            loss_eikonal = mse(norm(grad_random), 1.0)
            
            # Combine losses
            loss = loss_photo + lambda_sdf * loss_eikonal
            
            total_loss += loss
            
        # Backpropagate
        optimizer.zero_grad()
        total_loss.backward()
        optimizer.step()
```

## Operating characteristics
*   **Complexity:** Time complexity is dominated by the two MLP evaluations per sample point along the ray. Memory complexity is proportional to the number of parameters in the two MLPs and the stored features.
*   **Performance:** On the DTU dataset, VolSDF is on par with IDR (which uses object masks) and outperforms NeRF and COLMAP in reconstruction accuracy. Rendering quality is comparable to NeRF [S1, unofficial]. On BlendedMVS, it provides more faithful results than IDR without requiring ground truth masks [S1, unofficial].
*   **Strengths:** Disentangles geometry and appearance; does not require object masks for training; avoids extraneous surface parts common in mask-based methods like IDR [S1, unofficial].
*   **Limitations:** Relies on the quality of the SDF approximation; performance depends on the choice of positional encoding [S1, unofficial].

## Background
VolSDF was introduced in the paper "Volume Rendering of Neural Implicit Surfaces" [S1, unofficial]. It addresses the limitations of previous implicit surface methods like IDR, which required segmentation masks and suffered from extraneous geometry. The method leverages the properties of Signed Distance Functions to regularize the neural network's output, ensuring the predicted density corresponds to a valid surface geometry. It is used for multi-view 3D surface reconstruction on datasets like DTU and BlendedMVS [S1, unofficial].

## Sources
[S1] Volume Rendering of Neural Implicit Surfaces, https://doi.org/10.48550/arxiv.2106.12052, fetched 2026-10-26 (unofficial)
