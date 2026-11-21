# Global Scene Generator (GSN)


## Overview
Global Scene Generator (GSN) is a generative model that synthesizes unconstrained 3D scenes by decomposing them into local regions using a global latent code and a locally conditioned radiance field [S1]. It solves the scalability problem of single-network radiance fields (like standard NeRF or GRAF) by distributing scene representation across a 2D grid of local latent codes, allowing the model to handle large, densely populated environments [S1]. Unlike dictionary-based memory models that are limited to viewing angles present in training data, GSN learns a scene prior that enables extrapolation to unseen viewpoints [S1].

## How it works
**Inputs:**
*   Global latent code $z \sim p_z$ sampled from a prior distribution.
*   Camera pose $T = [R, t] \in SE(3)$ sampled from an empirical pose distribution $p_T$.
*   Camera intrinsics $K \in \mathbb{R}^{3 \times 3}$.

**Outputs:**
*   Rendered image $\hat{X} \in \mathbb{R}^{3 \times w \times h}$.

**Data Structures:**
1.  **Global Generator Output ($W$):** A 2D grid of local latent codes $W \in \mathbb{R}^{c \times s \times s}$. This acts as a "latent floorplan" for the scene.
2.  **Radiance Field Input:** A concatenated vector $[p_0, d] \in \mathbb{R}^5$ and a local latent code $w_{ij} \in \mathbb{R}^c$.
3.  **Radiance Field Architecture:** A network $f$ where the local latent code $w_{ij}$ modulates linear layers (similar to CIPS [S1]).

**Numbered Steps:**

1.  **Generate Local Latent Grid:** Sample $z \sim p_z$ and feed it to the global generator $g$ (implemented using StyleGAN2 architecture [S1]). The generator outputs the 2D grid of local latent codes $W \in \mathbb{R}^{c \times s \times s}$ [S1].

2.  **Determine Local Context:** For a 3D point $p$ in the global coordinate system, determine its projection onto the $zx$-plane (assuming the $y$-axis points up). This projection yields discrete grid indices $(i, j)$ [S1].

3.  **Sample Local Latent Code:** Sample the local latent code $w_{ij} \in \mathbb{R}^c$ from the grid $W$ at indices $(i, j)$ using bi-linear sampling [S1].

4.  **Transform Coordinates:** Transform the global point $p$ into a local point $p_0$ relative to the local coordinate system defined by $w_{ij}$. This decomposition prevents the radiance field $f$ from relying on absolute global coordinates, enabling spatial sharing of latent codes [S1].

5.  **Prepare Radiance Field Input:** Apply positional encoding to $p_0$ and concatenate it with the camera direction $d$ to form the input vector $[p_0, d] \in \mathbb{R}^5$ [S1].

6.  **Evaluate Radiance Field:** Evaluate the locally conditioned radiance field $f$ using inputs $[p_0, d]$ and modulation by $w_{ij}$. The network outputs occupancy $\sigma \in [0, \infty]$ and appearance $a \in \mathbb{R}^3$ [S1].

7.  **Volumetric Rendering:** Render the image by evaluating $f$ along rays $\mathbf{r}$. The color $c(\mathbf{r})$ for a ray is obtained by approximating the integral in Eq. 1 [S1]:

    $$
    c(\mathbf{r}, W) = \int_{u_n}^{u_f} T_{\mathbf{r}}(u) \sigma_{\mathbf{r}}(u, w_{ij}) a_{\mathbf{r}}(u, d, w_{ij}) \, du
    $$

    where the transmittance $T_{\mathbf{r}}(u)$ is defined as:

    $$
    T_{\mathbf{r}}(u) = \exp \left( - \int_{u_n}^{u} \sigma_{\mathbf{r}}(v, w_{ij}) \, dv \right)
    $$

    Here, $u_n$ and $u_f$ are the near and far clipping planes, and $w_{ij}$ is the local latent code associated with the spatial location of the point on the ray [S1].

8.  **Refinement:** The rendering step produces a feature map at a lower resolution. This feature map is upsampled using a small convolutional refinement network to achieve the final output resolution $\hat{X}$ [S1].

**Parameters:**
*   $c$: Dimension of the latent codes.
*   $s$: Spatial resolution of the grid $W$.
*   $T$: Camera extrinsic parameters.
*   $K$: Camera intrinsic parameters.

**Edge Cases and Pitfalls:**
*   **Invalid Camera Poses:** Sampling camera poses inside walls or solid objects is a risk. GSN handles this by performing stochastic weighted sampling over a set of candidate poses. The weights are derived from the occupancy $\sigma$ predicted by the generator at those locations, normalized via a softmin function to create a multinomial distribution [S1].
*   **Coordinate System Dependency:** If the global coordinate system is not decomposed into local coordinate systems for each $w_{ij}$, the radiance field $f$ will rely on absolute coordinates, breaking the spatial sharing property and preventing the model from generalizing across different grid positions [S1].
*   **Dimensionality:** The method uses a 2D grid for efficiency. While it can be extended to a 3D grid, the 2D approach is the standard implementation for computational efficiency [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class GlobalSceneGenerator:
    def __init__(self, latent_dim, grid_size, feature_dim):
        """
        latent_dim: dimension of global latent code z
        grid_size: s, spatial dimension of the local latent grid W
        feature_dim: c, dimension of local latent codes w_ij
        """
        super().__init__()
        # Global generator g using StyleGAN2-like architecture [S1]
        self.global_generator = StyleGAN2Generator(latent_dim, feature_dim, grid_size)
        
        # Locally conditioned radiance field f [S1]
        self.radiance_field = LocallyConditionedRadianceField(feature_dim)
        
        # Convolutional refinement network for upsampling [S1]
        self.refinement_net = ConvRefinementNetwork()

    def forward(self, z, camera_pose, camera_intrinsics, ray_directions):
        """
        z: Global latent code [B, latent_dim]
        camera_pose: Camera pose T = [R, t] [B, 3, 4] or [B, 6]
        camera_intrinsics: K [B, 3, 3]
        ray_directions: Directions for each pixel [B, H, W, 3]
        
        Returns:
        rendered_image: Final rendered image [B, 3, H_out, W_out]
        """
        # Step 1: Generate local latent grid W [S1]
        # W shape: [B, feature_dim, grid_size, grid_size]
        W = self.global_generator(z)
        
        # Step 2-6: Evaluate radiance field along rays
        # For each ray, we need to sample points and query f
        # This is typically done via volumetric rendering loop
        
        # Pre-compute ray origins and directions in global coordinates
        ray_origins = compute_ray_origins(camera_pose, camera_intrinsics)
        
        # Render feature map at lower resolution [S1]
        # The actual volumetric integration is handled by the renderer
        # which queries f with local conditioning
        feature_map = self.volumetric_render(
            W=W, 
            ray_origins=ray_origins, 
            ray_directions=ray_directions,
            radiance_field=self.radiance_field
        )
        
        # Step 8: Refine and upsample [S1]
        rendered_image = self.refinement_net(feature_map)
        
        return rendered_image

    def volumetric_render(self, W, ray_origins, ray_directions, radiance_field):
        """
        Implements the integration in Eq. 1 [S1]
        """
        # Sample points along rays
        # u_n to u_f are near/far planes
        points_global = sample_points_along_rays(ray_origins, ray_directions, u_n, u_f)
        
        # For each point, determine local context
        # Project point to zx-plane to get (i, j) [S1]
        # Note: Implementation details for projection depend on coordinate system convention
        # Assuming y is up, project to zx plane
        projected_points = points_global[:, [0, 2]] # x, z
        
        # Map projected points to grid indices (i, j)
        # This requires normalizing coordinates to grid space
        grid_indices = map_to_grid_indices(projected_points, grid_size=W.shape[-1])
        
        # Sample local latent codes w_ij using bi-linear sampling [S1]
        # W shape: [B, C, S, S]
        # grid_indices shape: [B, N, 2] where N is number of samples
        w_ij = bilinear_sample_grid(W, grid_indices)
        
        # Transform global points to local coordinates p0 [S1]
        # This involves subtracting the local origin and rotating
        # The local origin and rotation are implicitly defined by the grid position
        # For simplicity, assuming a fixed local frame per grid cell
        points_local = transform_to_local_coordinates(points_global, grid_indices)
        
        # Apply positional encoding to p0 [S1]
        points_local_encoded = positional_encoding(points_local)
        
        # Get camera directions d for each sample
        # Directions are constant along the ray
        directions = ray_directions[grid_indices[:, 0, 0]] # Broadcast direction to samples
        
        # Concatenate [p0, d] [S1]
        # p0 is encoded, so dimension is higher, plus d (3)
        # The paper states input is R^5, implying p0 is 2D or encoded to 2D?
        # Standard NeRF uses 3D pos + 3D dir = 6D. 
        # Paper says [p0, d] in R^5. This implies p0 might be 2D (x,z) or encoded differently.
        # Given "positional encoding" is applied, and input to f is R^5, 
        # it is likely p0 is the 2D local coordinate (x,z) in the local frame, 
        # and d is 3D direction. Or p0 is 3D but d is 2D? 
        # Re-reading: "p0 which results from expressing p relative to the local coordinate system... 
        # and applying positional encoding". 
        # If input is R^5, and d is 3D, p0 must be 2D. 
        # Since projection is on zx-plane, local coordinates might be (x_local, z_local).
        
        # Let's assume p0 is 2D local coordinate (x, z) in the local frame
        # and d is 3D direction.
        # However, standard volumetric rendering requires 3D position for density.
        # The paper says "f : R^5 x R^c -> [0,1] x R^3".
        # It is most likely that p0 is the 2D coordinate in the local plane, 
        # and the depth is handled by the ray parameter u? 
        # No, radiance fields usually take 3D position.
        # Let's stick to the text: "input [p0, d] in R^5".
        # If d is 3D, p0 is 2D.
        
        # Construct input tensor
        # p0_local_2d = points_local[:, [0, 2]] # x, z in local frame
        # d = ray_directions
        # input_to_f = torch.cat([p0_local_2d, d], dim=-1) # [B, N, 5]
        
        # Evaluate f
        # sigma, a = radiance_field(input_to_f, w_ij)
        
        # Compute transmittance and color using Eq. 1 [S1]
        # T_i = exp(-sum(sigma_j * delta_j))
        # C = sum(T_i * (1 - exp(-sigma_i * delta_i)) * a_i)
        
        # Return feature map
        pass

class LocallyConditionedRadianceField(nn.Module):
    def __init__(self, latent_dim):
        super().__init__()
        # Modulated linear layers similar to CIPS [S1]
        self.modulation = ModulatedLinear(latent_dim)
        
    def forward(self, x, w_ij):
        """
        x: [B, N, 5] -> [p0, d]
        w_ij: [B, N, c]
        """
        # Modulate weights of linear layers using w_ij [S1]
        # This is similar to AdaIN or StyleGAN modulation
        out = self.modulation(x, w_ij)
        return out

class StyleGAN2Generator(nn.Module):
    def __init__(self, latent_dim, feature_dim, grid_size):
        super().__init__()
        # Mapping network, modulated conv blocks, learned constant [S1]
        pass

class ConvRefinementNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        # Small convolutional network for upsampling [S1]
        pass
```

## Operating characteristics
*   **Complexity:** The method uses a 2D grid for computational efficiency, though it can be extended to 3D [S1]. The memory usage is determined by the size of the grid $W$ ($c \times s \times s$) and the radiance field network parameters.
*   **Strengths:** It scales effectively to large, densely populated scenes where single-network radiance fields show diminishing returns [S1]. It allows for explicit control over viewpoint and field of view via camera parameters [S1].
*   **Limitations:** The 2D grid assumption may not capture vertical variations as well as a 3D grid, although the paper notes the framework can be extended [S1]. Sampling invalid camera poses (e.g., inside walls) requires a specific stochastic weighted sampling strategy to mitigate [S1].
*   **When to use:** Use GSN when generating unconstrained, large-scale scenes with diverse viewpoints, particularly when single-network approaches fail to scale [S1].
*   **When not to use:** Avoid if the scene is small and simple, where a single radiance field (like NeRF or GRAF) would be more efficient and sufficient [S1].

## Background
GSN was introduced in the paper "Unconstrained Scene Generation with Locally Conditioned Radiance Fields" [S1]. It builds upon the concept of locally conditioned radiance fields [34] and StyleGAN2 [25]. The method addresses the limitations of 1D global latent codes used in models like GRAF [49] and $\pi$-GAN [3], which do not scale well to large scenes [S1]. It also contrasts with dictionary-based memory models that rely on stored observations and cannot extrapolate to unseen viewpoints [S1].

## Sources
[S1] Unconstrained Scene Generation with Locally Conditioned Radiance Fields, https://doi.org/10.1109/iccv48922.2021.01404, fetched 2026-10-26
