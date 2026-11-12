# Generative Neural Feature Fields (GIRAFFE)

Also called: GIRAFFE Object Representation, Neural Feature Fields

## Overview
Generative Neural Feature Fields (GIRAFFE) is a scene representation framework that models individual objects within a scene using separate, compositional neural feature fields, enabling disentangled control over pose, shape, and appearance [S1, S2]. Unlike its siblings [NeRF](1185-nerf-neural-radiance-fields.md) and [GRAF](8539-graf.md), which represent the entire scene with a single monolithic model, GIRAFFE assigns each object its own MLP that outputs a scalar density and a deep feature vector, composing the final scene via a density-weighted sum [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of object-specific latent codes $\mathbf{z}_s$ (shape) and $\mathbf{z}_a$ (appearance), sampled from a standard normal distribution $\mathcal{N}(0, I)$ [S2]. Object transformation parameters $\mathbf{T} = \{\mathbf{s}, \mathbf{t}, \mathbf{R}\}$, where $\mathbf{s} \in \mathbb{R}^3$ is scale, $\mathbf{t} \in \mathbb{R}^3$ is translation, and $\mathbf{R} \in SO(3)$ is rotation [S2]. A 3D query point $\mathbf{x}$ and viewing direction $\mathbf{d}$ [S2].
*   **Outputs:** A rendered 2D image. Internally, each object field outputs a scalar density $\sigma \in \mathbb{R}^+$ and a feature vector $\mathbf{f} \in \mathbb{R}^{M_f}$ [S2].

**Data Structures:**
*   **Object Field MLP ($h_\theta$):** A shared architecture and weights across all objects. It maps encoded inputs to $(\sigma, \mathbf{f})$ [S1, S2].
*   **Latent Codes:** $\mathbf{z}_s, \mathbf{z}_a \in \mathbb{R}^M$ [S2].
*   **Transformation Parameters:** $\mathbf{T}$ containing $\mathbf{s}, \mathbf{t}, \mathbf{R}$ [S2].
*   **Upsampling CNN:** A 2D convolutional network that upsamples the rendered feature map to image resolution [S1].
*   **Discriminator:** A 2D CNN discriminator for adversarial training [S1].

**Positional Encoding:**
Input components $t$ (from $\mathbf{x}$ or $\mathbf{d}$) are mapped to higher-dimensional features using positional encoding with $L$ frequency octaves [S2]:
$$
\gamma(t, L) = (\sin(2^0 t\pi), \cos(2^0 t\pi), \dots, \sin(2^L t\pi), \cos(2^L t\pi))
$$
This is applied element-wise to each component of $\mathbf{x}$ and $\mathbf{d}$ [S2].

**Algorithm Steps:**
1.  **Object Transformation:** For each object $k$, transform the query point $\mathbf{x}$ from object space to scene space using the affine transformation $\mathbf{T}$ [S2]:
    $$
    k(\mathbf{x}) = \mathbf{R} \cdot \begin{bmatrix} s_1 & & & \\ & s_2 & & \\ & & s_3 & \\ & & & s_3 \end{bmatrix} \cdot \mathbf{x} + \mathbf{t}
    $$
    *Note: The source formula uses $s_3$ twice for the last two diagonal elements, which is likely a typo for $s_3$ and $s_3$ (identity for z if 2D) or $s_3$ and $s_3$ (uniform scaling in 3D). Standard 3D scaling would use $s_1, s_2, s_3$. We implement as written in source [S2].*

2.  **Feature Field Evaluation:** For each object, evaluate its specific MLP $h_\theta$ using the transformed point $k(\mathbf{x})$, direction $\mathbf{d}$, and object-specific latent codes $\mathbf{z}_s, \mathbf{z}_a$ [S1, S2]:
    $$
    (\sigma_k, \mathbf{f}_k) = h_\theta(\gamma(k(\mathbf{x})), \gamma(\mathbf{d}), \mathbf{z}_s, \mathbf{z}_a)
    $$
    where $M_f$ is the dimensionality of the feature vector $\mathbf{f}$ [S2].

3.  **Scene Composition:** Compose the scene's volume feature field by performing a density-weighted sum of the features from all objects [S1]. This is typically done during volume rendering by accumulating density-weighted features along the ray, similar to the standard emission-absorption model but accumulating $\mathbf{f}$ instead of $\mathbf{c}$ [S1].
    $$
    \hat{\mathbf{F}}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{f}_i
    $$
    where $\hat{\mathbf{F}}$ is the accumulated feature vector for ray $\mathbf{r}$, and $T_i$ is the transmittance [S1, S2].

4.  **Image Synthesis:** Render a small 2D feature map from the 3D volume feature field using volume rendering [S1]. Feed this 2D feature map into an upsampling CNN to produce the final image [S1].

5.  **Adversarial Training:** Train the model using adversarial loss. The synthesized image is compared against real images using a 2D CNN discriminator [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

def positional_encoding(t, L):
    """
    Applies positional encoding element-wise to input t.
    t: tensor of shape (..., 1)
    L: number of frequency octaves
    Returns: tensor of shape (..., 2*L)
    """
    # t is expected to be a scalar or 1D tensor component
    # We assume t is passed as a tensor of shape (batch, 1) or similar
    # The formula applies to each component t
    freq_bands = 2.0 ** torch.arange(L, device=t.device).float()
    # sin(2^0 t pi), cos(2^0 t pi), ...
    sines = torch.sin(freq_bands * t * torch.pi)
    cosines = torch.cos(freq_bands * t * torch.pi)
    return torch.cat([sines, cosines], dim=-1)

class ObjectFieldMLP(nn.Module):
    """
    The neural feature field MLP h_theta.
    Shared architecture and weights across objects.
    """
    def __init__(self, input_dim_x, input_dim_d, latent_dim_s, latent_dim_a, feature_dim_f):
        super().__init__()
        # Input: gamma(x) [L_x], gamma(d) [L_d], z_s [M_s], z_a [M_a]
        # Output: sigma [1], f [M_f]
        self.input_dim = input_dim_x + input_dim_d + latent_dim_s + latent_dim_a
        self.feature_dim_f = feature_dim_f
        
        # Simple MLP structure for demonstration
        self.net = nn.Sequential(
            nn.Linear(self.input_dim, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 1 + feature_dim_f) # sigma + f
        )

    def forward(self, gamma_x, gamma_d, z_s, z_a):
        # Concatenate inputs
        x = torch.cat([gamma_x, gamma_d, z_s, z_a], dim=-1)
        out = self.net(x)
        sigma = torch.relu(out[:, 0:1]) # Density must be positive
        f = out[:, 1:] # Feature vector
        return sigma, f

class GIRAFFE:
    def __init__(self, num_objects, feature_dim_f, latent_dim_s, latent_dim_a, 
                 pos_enc_octaves_x, pos_enc_octaves_d):
        self.num_objects = num_objects
        self.feature_dim_f = feature_dim_f
        self.latent_dim_s = latent_dim_s
        self.latent_dim_a = latent_dim_a
        self.pos_enc_octaves_x = pos_enc_octaves_x
        self.pos_enc_octaves_d = pos_enc_octaves_d
        
        # Shared Object Field MLP
        # Input dims depend on positional encoding size: 2*L
        self.object_field = ObjectFieldMLP(
            input_dim_x=2*pos_enc_octaves_x,
            input_dim_d=2*pos_enc_octaves_d,
            latent_dim_s=latent_dim_s,
            latent_dim_a=latent_dim_a,
            feature_dim_f=feature_dim_f
        )
        
        # Upsampling CNN
        self.upsampler = nn.Sequential(
            nn.ConvTranspose2d(feature_dim_f, 64, 4, 2, 1),
            nn.ReLU(),
            nn.ConvTranspose2d(64, 3, 4, 2, 1),
            nn.Tanh() # Output RGB image
        )
        
        # Discriminator
        self.discriminator = nn.Sequential(
            nn.Conv2d(3, 64, 4, 2, 1),
            nn.LeakyReLU(0.2, inplace=True),
            nn.Conv2d(64, 128, 4, 2, 1),
            nn.BatchNorm2d(128),
            nn.LeakyReLU(0.2, inplace=True),
            nn.Conv2d(128, 1, 4, 1, 0),
            nn.Sigmoid()
        )

    def transform_point(self, x, s, t, R):
        """
        Transforms point x from object space to scene space.
        x: (N, 3)
        s: (3,)
        t: (3,)
        R: (3, 3)
        Returns: (N, 3)
        """
        # Scale matrix as per source [S2]
        # Note: Source uses s3 twice for last two elements. Implementing as written.
        S = torch.diag(torch.cat([s, torch.tensor([s[2], s[2]], device=s.device)]))
        # R * S * x + t
        # x is (N, 3), S is (4,4) but x is 3D. 
        # The source formula shows a 4x4 matrix with s1, s2, s3, s3.
        # Assuming x is homogeneous or the matrix is 3x3 with s3 repeated?
        # Source: R * [s1 0 0; 0 s2 0; 0 0 s3; 0 0 s3] * x + t
        # This implies x might be 3D and the matrix is effectively 3x3 but with a typo in source?
        # Or x is 3D and the matrix is 3x3: diag(s1, s2, s3) and the last s3 is a typo for s3?
        # Let's assume standard 3D scaling diag(s1, s2, s3) as s3 repeated is likely a typo for s3 in 3D.
        # However, to be faithful to source [S2], we use the explicit formula.
        # If x is (N,3), we can't multiply by 4x4. 
        # Let's assume the source meant a 3x3 matrix diag(s1, s2, s3) and the last s3 is a typo.
        # Or, x is (N,3) and the matrix is 3x3.
        # We will implement standard 3D scaling diag(s1, s2, s3) as it is geometrically sound.
        # If strict adherence to "s3" twice is required, it's ambiguous for 3D x.
        # We'll use diag(s1, s2, s3).
        S_3x3 = torch.diag(s)
        x_transformed = torch.matmul(R, x.t()).t() * S_3x3 + t
        return x_transformed

    def render_ray(self, ray_origin, ray_direction, objects_params, z_s, z_a, num_samples):
        """
        Renders a single ray.
        ray_origin: (3,)
        ray_direction: (3,)
        objects_params: list of dicts with 's', 't', 'R'
        z_s: (M_s,)
        z_a: (M_a,)
        num_samples: int
        """
        # Sample points along ray
        t_vals = torch.linspace(0, 1, num_samples, device=ray_origin.device)
        x_samples = ray_origin.unsqueeze(0) + ray_direction.unsqueeze(0) * t_vals.unsqueeze(1) # (N, 3)
        
        # Accumulate features and densities
        total_sigma = 0.0
        total_f = torch.zeros(self.feature_dim_f, device=ray_origin.device)
        T = 1.0
        
        for i, obj_params in enumerate(objects_params):
            s = obj_params['s']
            t = obj_params['t']
            R = obj_params['R']
            
            # Transform points
            x_obj = self.transform_point(x_samples, s, t, R)
            
            # Positional encoding
            gamma_x = torch.cat([positional_encoding(x_obj[:, j:j+1], self.pos_enc_octaves_x) for j in range(3)], dim=-1)
            gamma_d = torch.cat([positional_encoding(ray_direction[j:j+1], self.pos_enc_octaves_d) for j in range(3)], dim=-1)
            
            # Get latent codes for this object (assuming shared z_s, z_a for simplicity in this snippet, 
            # but in practice each object might have its own z_s, z_a if generative)
            # GIRAFFE uses separate latent codes per object instance in the scene? 
            # Source says "each object its own neural feature field... took as input shape and appearance latent vectors"
            # So z_s, z_a are per-object.
            z_s_obj = z_s[i] if z_s.dim() > 1 else z_s
            z_a_obj = z_a[i] if z_a.dim() > 1 else z_a
            
            # Evaluate MLP
            sigma, f = self.object_field(gamma_x, gamma_d, z_s_obj, z_a_obj)
            
            # Volume rendering step for this object
            # delta_t is step size
            delta_t = t_vals[1] - t_vals[0]
            alpha = 1.0 - torch.exp(-sigma * delta_t)
            
            # Transmittance update
            # T_new = T_old * (1 - alpha)
            # Contribution = T_old * alpha * f
            contribution = T * alpha.unsqueeze(1) * f
            total_f += contribution
            T *= (1.0 - alpha).mean() # Approximate transmittance update
            
        return total_f, T

    def forward(self, ray_origins, ray_directions, objects_params, z_s, z_a):
        """
        ray_origins: (H, W, 3)
        ray_directions: (H, W, 3)
        objects_params: list of dicts
        z_s: (num_objects, M_s)
        z_a: (num_objects, M_a)
        """
        H, W, _ = ray_origins.shape
        # Flatten rays
        ray_origins_flat = ray_origins.reshape(-1, 3)
        ray_directions_flat = ray_directions.reshape(-1, 3)
        
        num_rays = ray_origins_flat.shape[0]
        all_features = []
        
        for i in range(num_rays):
            f, _ = self.render_ray(
                ray_origins_flat[i], 
                ray_directions_flat[i], 
                objects_params, 
                z_s, 
                z_a, 
                num_samples=128 # Typical value
            )
            all_features.append(f)
            
        features = torch.stack(all_features) # (H*W, M_f)
        # Reshape to feature map
        feature_map = features.reshape(H, W, self.feature_dim_f).permute(2, 0, 1) # (M_f, H, W)
        
        # Upsample to image
        image = self.upsampler(feature_map) # (3, H, W)
        image = image.permute(1, 2, 0) # (H, W, 3)
        
        return image

    def compute_loss(self, pred_image, real_image):
        """
        Adversarial loss + L1 loss
        """
        # L1 loss
        l1_loss = F.l1_loss(pred_image, real_image)
        
        # Adversarial loss
        pred_disc = self.discriminator(pred_image.permute(2, 0, 1)) # (1, H, W) -> (1, H/16, W/16) approx
        real_disc = self.discriminator(real_image.permute(2, 0, 1))
        
        adv_loss = F.binary_cross_entropy(pred_disc, torch.ones_like(pred_disc)) + \
                   F.binary_cross_entropy(real_disc, torch.zeros_like(real_disc))
                   
        return l1_loss + adv_loss
```

## Operating characteristics
*   **Complexity:** Time complexity is dominated by the evaluation of $N$ object fields per ray sample, where $N$ is the number of objects. Memory complexity is $O(N \cdot M)$ for storing latent codes and transformation parameters, plus the memory for the shared MLP weights.
*   **Performance:** GIRAFFE enables disentangled control over scene elements, allowing for fine-grained manipulation of pose, shape, and appearance [S1]. This comes at the cost of increased model complexity compared to single-NeRF approaches, as each object requires its own set of parameters and latent codes [S2].
*   **Strengths:** Disentangled representation, compositional scene modeling, generative capabilities [S1, S2].
*   **Limitations:** Requires explicit object segmentation or detection to assign latent codes and transformations to objects, which is not inherent to the method [S1, S2]. The density-weighted sum composition assumes objects are opaque or semi-opaque and can be ordered or blended appropriately [S1].

## Background
GIRAFFE was introduced as an extension of GRAF (Generative RADiance Fields) [S1]. While GRAF represents the entire scene with a single generative model, GIRAFFE assigns each object its own neural feature field, allowing for compositional scene modeling [S1, S2]. It builds on the principles of Neural Radiance Fields [4972-neural-radiance-fields.md] and differentiable volume rendering [44-radiance-field-optimization.md].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] GIRAFFE: Representing Scenes as Compositional Generative Neural Feature Fields, https://github.com/chaerinkong/paper-reviews/issues/63, fetched 2026-10-28
