# HAME-NeRF

Also called: High Accuracy Mesh Extraction Leveraging Neural Radiance Fields

## Overview
HAME-NeRF is a two-stage framework for extracting high-fidelity, textured surface meshes from RGB images by defining volume density as a Secant Hyperbolic Function applied to a Signed Distance Function (SDF) [S1]. It builds on [NeuS](6060-neus.md) by introducing a spatially-varying sharpness parameter $\beta(\mathbf{x})$ learned via an MLP, which enhances flexibility for complex surfaces compared to methods using fixed $\beta$ values [S1]. Unlike [NeuS](6060-neus.md) which focuses on differentiable volume rendering for surface representation, HAME-NeRF explicitly optimizes a mesh through differentiable rendering and iterative refinement, achieving superior geometric accuracy and visual quality on synthetic benchmarks compared to [NeRF2Mesh](52-nerf2mesh.md) [S1].

## How it works
HAME-NeRF operates in two stages: initialization of geometry and appearance using a grid-based NeRF, followed by mesh refinement via differentiable rendering [S1].

### Inputs and Outputs
*   **Inputs:** A set of $N$ RGB images with known camera poses.
*   **Outputs:** A textured triangular mesh (vertices, faces, and diffuse/specular textures) and a learned scene representation.

### Data Structures
1.  **Geometry Representation:**
    *   Multi-resolution feature grid $E^{\text{geo}}$ storing spatial features for SDF computation [S1].
    *   A shallow MLP that maps features from $E^{\text{geo}}$ to the SDF value [S1].
2.  **Appearance Representation:**
    *   Multi-resolution feature grid $E^{\text{app}}$ storing spatial features for appearance [S1].
    *   MLP$_1$: Maps $E^{\text{app}}(\mathbf{x})$ to diffuse color $\mathbf{c}_d$ and specular feature $\mathbf{f}_s$ [S1].
    *   MLP$_2$: Maps $\mathbf{f}_s$ and view direction $\mathbf{d}$ to specular color $\mathbf{c}_s$ [S1].
    *   MLP$_\beta$: Maps spatial coordinates $\mathbf{x}$ to the sharpness parameter $\beta(\mathbf{x})$ [S1].
3.  **Mesh Data:**
    *   Vertices $\mathbf{V}$ with per-vertex offsets learned during refinement [S1].
    *   Faces $\mathbf{F}$ with adaptive resolution based on rendering error [S1].

### Formulas
The volume density $\sigma$ is derived from the SDF and the spatially varying parameter $\beta(\mathbf{x})$:
$$
\sigma(\text{SDF}, \beta) = \frac{1}{4\beta} \cdot \frac{1}{\cosh^2\left(\frac{\text{SDF}}{2\beta}\right)} \quad (2)
$$
where $\beta(\mathbf{x})$ is output by MLP$_\beta(\mathbf{x})$ [S1].

Appearance is decomposed into diffuse and specular components:
$$
\mathbf{c}_d, \mathbf{f}_s = \psi(\text{MLP}_1(E^{\text{app}}(\mathbf{x}))) \quad (4)
$$
$$
\mathbf{c}_s = \psi(\text{MLP}_2(\mathbf{f}_s, \mathbf{d})) \quad (4)
$$
$$
\mathbf{c} = \mathbf{c}_d + \mathbf{c}_s \quad (5)
$$
where $\psi$ denotes a sigmoid activation, $\mathbf{f}_s$ encodes specular information, and $\mathbf{d}$ is the view direction [S1].

### Parameters
*   $\lambda_{\text{elk}}$: Weight for the Eikonal Loss [S1].
*   $\lambda_\beta$: Weight for the Beta Regularization Loss [S1].
*   Typical values: Not given in the sources.

### Steps

#### Stage 1: Initialization
1.  **Train Grid-based NeRF:** Initialize a grid-based NeRF structure with an SDF component.
    *   Compute SDF using $E^{\text{geo}}$ and a shallow MLP [S1].
    *   Compute density $\sigma$ using Equation (2) [S1].
    *   Compute appearance using Equations (4) and (5) [S1].
2.  **Optimize:** Minimize the total loss (defined below) to converge the NeRF representation [S1].
3.  **Extract Coarse Mesh:** Apply Marching Cubes to the converged SDF field to generate an initial coarse mesh [S1].

#### Stage 2: Refinement
1.  **Differentiable Rendering:** Use nvdiffrast to render the current mesh and compute the photometric rendering loss against input images [S1].
2.  **Vertex Optimization:** Optimize vertex positions by learning per-vertex offsets using gradients from the pixel-wise rendering loss [S1].
3.  **Adaptive Mesh Resolution:**
    *   Identify faces with high rendering errors using k-means clustering [S1].
    *   Subdivide high-error faces to increase resolution [S1].
    *   Simplify low-error regions to decrease resolution [S1].
4.  **Iterate:** Repeat steps 1-3 until convergence [S1].
5.  **Unbounded Scene Handling:** For unbounded scenes, partition the environment into regions of progressively lower resolution away from the center and apply regularization terms for smooth surfaces and bounded vertex displacements [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch
import torch.nn as nn
from torch.autograd import grad

class HAME_NeRF:
    def __init__(self, config):
        # Geometry
        self.geo_grid = MultiResolutionGrid(config.geo_config)
        self.geo_mlp = ShallowMLP(config.geo_mlp_config)
        
        # Appearance
        self.app_grid = MultiResolutionGrid(config.app_config)
        self.mlp_1 = MLP(config.mlp1_config) # Outputs c_d, f_s
        self.mlp_2 = MLP(config.mlp2_config) # Outputs c_s
        self.mlp_beta = MLP(config.mlp_beta_config) # Outputs beta
        
        # Mesh
        self.mesh = None
        self.vertex_offsets = None # Initialized to zeros
        
        # Loss weights
        self.lambda_elk = config.lambda_elk
        self.lambda_beta = config.lambda_beta

    def compute_sdf(self, x):
        features = self.geo_grid.query(x)
        sdf = self.geo_mlp(features)
        return sdf

    def compute_beta(self, x):
        return self.mlp_beta(x)

    def compute_density(self, sdf, beta):
        # sigma(SDF, beta) = (1 / 4*beta) * (1 / cosh^2(SDF / 2*beta))
        arg = sdf / (2 * beta)
        cosh_arg = torch.cosh(arg)
        density = (1.0 / (4 * beta)) * (1.0 / (cosh_arg ** 2))
        return density

    def compute_appearance(self, x, d):
        features = self.app_grid.query(x)
        c_d, f_s = self.mlp_1(features)
        c_s = self.mlp_2(torch.cat([f_s, d], dim=-1))
        # Apply sigmoid activation psi
        c_d = torch.sigmoid(c_d)
        c_s = torch.sigmoid(c_s)
        c = c_d + c_s
        return c

    def render_mesh(self, mesh, camera_params):
        # Use nvdiffrast to render the mesh
        # Returns rendered color c_hat and depth/normal if needed
        c_hat = nvdiffrast_render(mesh, camera_params)
        return c_hat

    def compute_losses(self, x_samples, d_samples, gt_colors, mesh):
        # 1. Photometric Rendering Loss
        # Render current mesh
        c_hat = self.render_mesh(mesh, camera_params)
        # gt_colors are the ground truth pixel colors
        L_render = torch.mean((c_hat - gt_colors) ** 2)
        
        # 2. Eikonal Loss
        # Sample points x near the surface
        x_samples.requires_grad_(True)
        sdf_val = self.compute_sdf(x_samples)
        grad_outputs = torch.ones_like(sdf_val)
        gradients = grad(
            sdf_val, x_samples, grad_outputs, create_graph=True, only_inputs=True
        )[0]
        grad_norm = torch.norm(gradients, dim=-1)
        L_eik = torch.mean((grad_norm - 1.0) ** 2)
        
        # 3. Beta Regularization Loss
        beta_val = self.compute_beta(x_samples)
        L_beta = self.lambda_beta * torch.mean(torch.abs(beta_val))
        
        L_total = L_render + self.lambda_elk * L_eik + L_beta
        
        return L_total, L_render, L_eik, L_beta

    def refine_mesh(self, images, poses, num_iterations):
        # Stage 1: Initialization
        # Train NeRF to convergence
        for i in range(num_init_iterations):
            # Sample rays, compute loss, backprop
            loss = self.compute_neural_loss(images, poses)
            loss.backward()
            optimizer.step()
            
        # Extract coarse mesh via Marching Cubes
        sdf_field = self.extract_sdf_field()
        self.mesh = marching_cubes(sdf_field)
        
        # Stage 2: Refinement
        for i in range(num_iterations):
            # Compute gradients for vertex offsets
            loss, _, _, _ = self.compute_losses(None, None, images, self.mesh)
            loss.backward()
            
            # Update vertex positions
            self.mesh.vertices += self.vertex_offsets.grad
            
            # Adaptive resolution
            errors = compute_rendering_errors(self.mesh, images, poses)
            high_error_faces = kmeans_cluster(errors, k=1)[0] # Indices of high error
            
            # Subdivide high error faces
            self.mesh.subdivide_faces(high_error_faces)
            
            # Simplify low error faces
            low_error_faces = kmeans_cluster(errors, k=1)[1] # Indices of low error
            self.mesh.simplify_faces(low_error_faces)
            
            optimizer.step()
```

### Edge Cases and Pitfalls
*   **Unbounded Scenes:** Requires partitioning the environment into regions of progressively lower resolution away from the center to maintain efficiency and accuracy [S1].
*   **Beta Instability:** Without regularization, $\beta(\mathbf{x})$ may vary unstably. The L1 penalty $\lambda_\beta |\beta(\mathbf{x})|$ is used to prevent this [S1].
*   **Precision:** The density formula involves division by $\beta$. If $\beta$ approaches zero, numerical instability may occur. The sources do not specify a lower bound for $\beta$, but the regularization encourages smooth transitions [S1].

## Subtopics

### NeuS
The parent method that introduced differentiable volume rendering for surface representation using SDFs. HAME-NeRF extends this by optimizing a mesh directly and learning spatially varying $\beta$. See [NeuS](6060-neus.md).

### NeRF2Mesh
A baseline method that applies Marching Cubes directly to a density field. HAME-NeRF improves upon this by using an SDF for mesh extraction and adaptive mesh refinement. See [NeRF2Mesh](52-nerf2mesh.md).

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Training involves two stages: NeRF initialization and mesh refinement. The refinement stage includes iterative k-means clustering and mesh subdivision, which may increase computational cost compared to static NeRF methods [S1].
*   **Memory Complexity:** Uses multi-resolution feature grids for geometry and appearance, which are memory-efficient compared to dense voxel grids. The mesh size adapts to scene complexity [S1].
*   **Measured Performance:**
    *   On the NeRF-Synthetic dataset, HAME-NeRF outperforms NeRF2Mesh in Chamfer Distance, Normal Consistency, PSNR, and visual quality [S1].
    *   On the Mip-NeRF 360 dataset, it achieves comparable, though slightly lower, performance than NeRF2Mesh [S1].
*   **Strengths:** High-fidelity surface extraction, spatially varying sharpness control, adaptive mesh resolution, and compatibility with real-time graphics via mesh output [S1].
*   **Limitations:** Slightly lower performance on complex real-world scenes (Mip-NeRF 360) compared to NeRF2Mesh. Requires careful tuning of loss weights $\lambda_{\text{elk}}$ and $\lambda_\beta$ [S1].
*   **When to use:** When high-quality, editable 3D meshes are required for real-time rendering or simulation, and the scene is well-lit or pre-baked lighting is acceptable [S1].
*   **When not to use:** For unbounded scenes with extreme dynamic range or complex lighting without pre-baking, as the method relies on baked lighting [S1].

## Background
HAME-NeRF was proposed to address the difficulty of extracting accurate and editable 3D meshes from NeRFs, which are inherently volumetric and incompatible with polygonal meshes needed for real-time graphics and simulation [S1]. The method builds on the NeRF2Mesh pipeline but introduces an SDF-based density modeling and a learned spatially varying $\beta$ parameter [S1]. It was evaluated on the NeRF-Synthetic and Mip-NeRF 360 datasets [S1].

## Sources
[S1] HAME-NeRF: High Accuracy Mesh Extraction Leveraging Neural Radiance Fields, https://zenodo.org/records/18469075, fetched 2026-10-26
