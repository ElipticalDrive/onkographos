# IDR (Implicit Differentiable Renderer)
Also called: Implicit Differentiable Renderer

## Overview
IDR is a method for learning a neural Signed Distance Function (SDF) and camera parameters simultaneously from uncalibrated images [S2, unofficial]. It disentangles geometry and appearance by combining a DVR-like implicit surface MLP with a NeRF-like view-dependent branch [S1, unofficial]. Unlike standard NeRF which models volume density, IDR represents geometry as the zero-level set of an MLP and uses a neural renderer to approximate light reflection [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of uncalibrated RGB images with approximate camera intrinsics and initial camera extrinsics [S2, unofficial].
*   **Outputs:** A trained neural SDF $\phi(\mathbf{x})$, a neural renderer predicting color $\mathbf{c}$, and optimized camera poses [S2, unofficial].

**Data Structures:**
1.  **SDF Network:** An MLP mapping $\mathbf{x} \in \mathbb{R}^3 \to \mathbb{R}$ representing the signed distance field [S2, unofficial].
2.  **Color Network:** An MLP taking encoded position, view direction, and surface normal as input to predict view-dependent color $\mathbf{c} \in \mathbb{R}^3$ [S1, unofficial].
3.  **Camera Parameters:** Extrinsic matrix $[R|t]$ and intrinsic matrix $K$, treated as trainable variables [S2, unofficial].

**Method Steps:**
1.  **Ray Casting:** For each pixel, compute the ray $r(\lambda) = o + \lambda d$ using current camera parameters [G].
2.  **SDF Sampling:** Sample points $\mathbf{x}$ along the ray and query the SDF network to obtain $\phi(\mathbf{x})$ [S2, unofficial].
3.  **Surface Intersection:** Identify the zero-crossing of $\phi(\mathbf{x})$ to approximate the surface location [S2, unofficial]. The geometry is represented as the zero-level set of an MLP [S2, unofficial].
4.  **Feature Extraction:** Compute the surface normal $\mathbf{n} = \nabla \phi(\mathbf{x}) / \|\nabla \phi(\mathbf{x})\|$ via automatic differentiation of the SDF network [G]. Apply positional encoding $\gamma$ to the position $\mathbf{x}$ and view direction $\mathbf{d}$ [G].
5.  **Color Prediction:** Concatenate the encoded position, view direction $\mathbf{d}$, and normal $\mathbf{n}$, then pass through the color network to predict the view-varying output color $\mathbf{c}$ [S1, unofficial]. Including the normal vector helps disentangle geometry and appearance [S1, unofficial].
6.  **Rendering:** Integrate the predicted color along the ray to produce the final pixel color [S2, unofficial].
7.  **Optimization:** Minimize the photometric loss between rendered images and input images [S2, unofficial]. Jointly optimize the network weights and camera extrinsics $[R|t]$ to recover from small miscalibration errors [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class MLP:
    def __init__(self, layer_sizes):
        self.weights = [random_normal((layer_sizes[i], layer_sizes[i+1])) for i in range(len(layer_sizes)-1)]
    def forward(self, x):
        for i, w in enumerate(self.weights):
            x = x @ w
            if i < len(self.weights) - 1:
                x = max(0, x) # ReLU
        return x

def positional_encoding(x, min_deg=0, max_deg=10):
    scales = [2**i for i in range(min_deg, max_deg)]
    xb = reshape(x * scales, (x.shape[0], -1))
    return concat(sin(xb), cos(xb), dim=-1)

def cast_rays(extrinsics, intrinsics, image_shape):
    H, W = image_shape[:2]
    u, v = meshgrid(arange(W), arange(H), indexing='xy')
    cam_coords = stack([u, v, ones_like(u)], dim=-1).float()
    inv_k = inverse(intrinsics)
    ray_dirs = (inv_k @ cam_coords.view(-1, 3).T).T
    ray_dirs = ray_dirs / norm(ray_dirs, dim=-1, keepdim=True)
    extr_inv = inverse(extrinsics)
    origins = extr_inv[:3, 3]
    return origins, ray_dirs

def sample_along_rays(origins, dirs, num_samples=128):
    t_vals = linspace(0.0, 1.0, num_samples)
    points = origins[:, None, :] + dirs[:, None, :] * t_vals[None, :, None]
    return points.view(-1, 3)

def integrate(colors, points, dirs):
    return colors.mean(dim=0).view(1, 1, 3)

# Initialization
sdf_net = MLP([3, 256, 1])
color_net = MLP([3 + 3 + 3 + 256, 256, 3])
camera_extrinsics = identity(4) # Trainable
camera_intrinsics = identity(3) # Fixed or trainable
optimizer = Adam([sdf_net.parameters(), color_net.parameters(), camera_extrinsics], lr=1e-4)

def render_image(camera_extrinsics, camera_intrinsics, training_images):
    loss = 0.0
    for image in training_images:
        origins, dirs = cast_rays(camera_extrinsics, camera_intrinsics, image.shape)
        points = sample_along_rays(origins, dirs)
        sdfs = sdf_net(points)
        # Compute normals via autograd [G]
        points.requires_grad_(True)
        grad = autograd_grad(sdfs.sum(), points, create_graph=True)[0]
        normals = grad / norm(grad, dim=-1, keepdim=True)
        # Encode inputs
        x_enc = positional_encoding(points)
        d_enc = positional_encoding(dirs)
        # Predict color
        colors = color_net(concat([x_enc, d_enc, normals], dim=-1))
        # Render
        rendered = integrate(colors, points, dirs)
        loss += mse_loss(rendered, image)
    return loss

# Training Loop
num_epochs = 1000
for epoch in range(num_epochs):
    loss = render_image(camera_extrinsics, camera_intrinsics, training_images)
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
```

## Operating characteristics
*   **Strengths:** Disentangles geometry and appearance effectively by using surface normals as input to the color branch [S1, unofficial]. Recovers from small camera miscalibration errors by jointly optimizing poses [S1, unofficial].
*   **Limitations:** Relies on a neural renderer that implicitly models lighting, which may limit explicit material control [S2, unofficial].
*   **Performance:** Not given in the sources.

## Background
IDR was introduced by Yariv et al. [YKM* 20] as one of the first geometric Implicit Neural Representations (INR) approaches to learn a neural SDF directly from images [S2, unofficial]. It builds on the concept of differentiable rendering to optimize the SDF and camera parameters simultaneously [S2, unofficial]. It is cited as influencing many other works in surface representation [S2, unofficial].

## Sources
[S1] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S2] Neural Volume Rendering: NeRF And Beyond, https://arxiv.org/abs/2101.05204v2, fetched 2026-10-26 (unofficial)
[S3] Geometric implicit neural representations for signed distance functions, https://doi.org/10.1016/j.cag.2024.104085, fetched 2026-10-26
