# MLI-NeRF

Also called: Intrinsic-Aware NeRF (Stage 2), Multi-Light Intrinsic-Aware Neural Radiance Fields, Stage 2 Training

## Overview
MLI-NeRF is a two-stage optimization procedure that learns an intrinsic-aware Neural Radiance Field (NeRF) capable of novel view synthesis, relighting, and intrinsic decomposition (separating reflectance and shading) without relying on statistical priors or ground-truth intrinsic labels [S1]. It differs from standard NeRF variants by explicitly modeling multiple light sources to generate physics-based pseudo-labels, and it differs from prior intrinsic NeRF methods (like [IntrinsicNeRF](10253-nerv-training.md)) by leveraging 3D geometric constraints and multi-light visibility rather than chromaticity or semantic statistical constraints [S1]. The method solves the ill-posed nature of inverse rendering by decoupling the geometry learning (Stage 1) from the intrinsic component separation (Stage 2), using pseudo-labels derived from the Stage 1 model to guide the decomposition [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of images captured from various camera positions and multiple light source positions [S1].
*   **Outputs:** A trained model capable of rendering novel views under arbitrary lighting, and decomposing scenes into reflectance ($R$), shading ($S$), and residual ($Re$) components [S1].

**Data Structures and Network Layout:**
The method builds upon the Neuralangelo architecture, which uses 3D hashing encoding and Signed Distance Fields (SDF) [S1].
1.  **Geometry Network ($f$):** Predicts the SDF value $sdf = f(x)$ for a spatial position $x$ [S1]. It also provides features ($feat$) and normals ($n$) derived from the gradient of the SDF [S1].
2.  **Color Network ($MLP_{color}$):** Used in Stage 1. Inputs: spatial position $x$, normal $n$, features $feat$, view direction $d$, and light position $l$. Output: color $c$ [S1].
3.  **Reflectance Network ($MLP_{reflectance}$):** Used in Stage 2. Inputs: spatial position $x$, normal $n$, features $feat$. It does not take light position or view direction as input [S1]. Output: reflectance $r$ [S1].
4.  **Shading Network ($MLP_{shading}$):** Used in Stage 2. Inputs: spatial position $x$, normal $n$, features $feat$, and light position $l$. It does not take view direction as input due to the Lambertian assumption [S1]. Output: shading $s$ [S1].
5.  **Residual ($Re$):** Accounts for specularities and undirected light, calculated as $Re = I - R \otimes S$ [S1].

**Numbered Steps:**

**Stage 1: Neural Multi-Light Implicit Representation**
1.  Train the geometry network $f$ and color network $MLP_{color}$ using images from multiple camera angles and light positions [S1].
2.  Optimize the Stage 1 loss $L_{S1}$:
    $$ L_{S1} = w_{rgb} L_{rgb} + w_{eik} L_{eik} + w_{curv} L_{curv} $$
    Where $L_{rgb}$ is the rendered image loss, $L_{eik}$ is the Eikonal loss, and $L_{curv}$ is the curvature loss [S1].
3.  **Post-processing (Pseudo Label Generation):**
    *   **Step A (Normals and Visibility):** Derive normals $n$ from the gradient of the SDF network. Use geometry network depth and sphere tracing to estimate intersection points. Compute light visibility $V$ (whether a point is directly illuminated) by sphere tracing from the light position to the intersection point [S1].
    *   **Step B (Pseudo Shading):** Compute pseudo-shading $S^*$ using the Lambertian cosine law and visibility, adjusted for gamma correction:
        $$ S^* = (\max(N \cdot L, 0) \otimes V)^\gamma $$
        Where $N$ is the normal ray, $L$ is the light ray, $\otimes$ is pixel-wise product, and $\gamma$ is the gamma correction exponent [S1].
    *   **Step C (Pseudo Reflectance):** Infer high-fidelity pseudo-reflectance $R^*$ using multiple lighting conditions. For each pixel, use the equation:
        $$ R^* = I \oslash S^* $$
        Where $I$ is the input image and $\oslash$ is pixel-wise division. This ignores the residual component $Re$ initially, assuming diffuse dominance [S1].

**Stage 2: Intrinsic-Aware NeRF**
1.  Freeze the geometry network $f$ [S1].
2.  Initialize and train the new modules $MLP_{reflectance}$ and $MLP_{shading}$ [S1].
3.  Render predicted reflectance $\hat{R}$ and shading $\hat{S}$ using the new networks [S1].
4.  Compute the residual $\hat{Re}$ from the rendered image $\hat{I}$ and the predicted components:
    $$ \hat{I} = \hat{R} \otimes \hat{S} + \hat{Re} $$
5.  Optimize the Stage 2 loss $L_{S2}$:
    $$ L_{S2} = w_{rgb} L_{rgb} + w_{intrinsic} L_{intrinsic} + w_{reg} L_{reg} $$
    *   **Intrinsic Loss ($L_{intrinsic}$):** Guides the separation using pseudo-labels $R^*$ and $S^*$:
        $$ L_{intrinsic} = W_R \cdot \|\hat{R} - R^*\|_1 + W_S \cdot \|\hat{S} - S^*\|_1 $$
        Where $W_R$ and $W_S$ are weight maps derived during pseudo-label generation [S1].
    *   **Regularization Loss ($L_{reg}$):** Prevents convergence to local minima (e.g., $R=0, S=0, Re=I$):
        $$ L_{reg} = \|\hat{Re}\|_1 $$
    *   **Weights:** Typical values are $w_{intrinsic} = 1.0$, $w_{reg} = 1.0$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def normalize(v):
    return v / np.linalg.norm(v)

def compute_gradient(network, points):
    # [G] Standard automatic differentiation
    return network.gradient(points)

def sphere_trace_visibility(geometry_net, origin, light_pos, max_steps=64):
    # [G] Standard sphere tracing for SDF visibility
    t = 0.0
    for _ in range(max_steps):
        pos = origin + t * normalize(light_pos - origin)
        sdf_val = geometry_net(pos)
        if sdf_val < 1e-4:
            return 1.0
        if t > 100.0:
            break
        t += sdf_val
    return 0.0

def volume_render(positions, colors, alphas, rays):
    # [G] Standard alpha compositing along rays
    T = 1.0
    rendered = np.zeros_like(colors)
    for i in range(len(positions)):
        alpha = alphas[i]
        rendered += T * alpha * colors[i]
        T *= (1.0 - alpha)
    return rendered

def train_stage_1(geometry_net, color_net, dataset, lr=1e-4, w_rgb=1.0, w_eik=0.1, w_curv=5e-4):
    optimizer_params = list(geometry_net.parameters()) + list(color_net.parameters())
    for epoch in range(num_epochs):
        for image, pose, light_pos in dataset:
            rays = sample_rays(pose)
            positions, directions = rays
            sdf = geometry_net(positions)
            feat = geometry_net.get_features(positions)
            normals = normalize(compute_gradient(geometry_net, positions))
            colors = color_net(positions, normals, feat, directions, light_pos)
            rendered_image = volume_render(positions, colors, alphas, rays)
            loss_rgb = np.mean((rendered_image - image) ** 2)
            loss_eik = np.mean((np.linalg.norm(compute_gradient(geometry_net, positions)) - 1.0) ** 2)
            loss_curv = geometry_net.curvature(positions)
            total_loss = w_rgb * loss_rgb + w_eik * loss_eik + w_curv * loss_curv
            for p in optimizer_params:
                p -= lr * p.grad
    return geometry_net, color_net

def generate_pseudo_labels(geometry_net, dataset, gamma=2.2):
    pseudo_shadings = []
    pseudo_reflectances = []
    for image, pose, light_pos in dataset:
        rays = sample_rays(pose)
        positions, directions = rays
        normals = normalize(compute_gradient(geometry_net, positions))
        visibility = sphere_trace_visibility(geometry_net, positions, light_pos)
        light_dir = normalize(light_pos - positions)
        cosine_term = np.maximum(np.sum(normals * light_dir, axis=-1), 0.0)
        pseudo_shading = (cosine_term * visibility) ** gamma
        pseudo_reflectance = image / pseudo_shading
        pseudo_shadings.append(pseudo_shading)
        pseudo_reflectances.append(pseudo_reflectance)
    return pseudo_shadings, pseudo_reflectances

def train_stage_2(geometry_net, reflectance_net, shading_net, 
                  pseudo_shadings, pseudo_reflectances, dataset, 
                  lr=1e-4, w_rgb=1.0, w_intrinsic=1.0, w_reg=1.0):
    for param in geometry_net.parameters():
        param.requires_grad = False
    optimizer_params = list(reflectance_net.parameters()) + list(shading_net.parameters())
    for epoch in range(num_epochs):
        for i, (image, pose, light_pos) in enumerate(dataset):
            rays = sample_rays(pose)
            positions, directions = rays
            feat = geometry_net.get_features(positions)
            normals = normalize(compute_gradient(geometry_net, positions))
            pred_reflectance = reflectance_net(positions, normals, feat)
            pred_shading = shading_net(positions, normals, feat, light_pos)
            rendered_image = volume_render(positions, pred_reflectance * pred_shading, alphas, rays)
            pred_residual = rendered_image - (pred_reflectance * pred_shading)
            loss_rgb = np.mean((rendered_image - image) ** 2)
            loss_intrinsic = np.mean(W_R * np.abs(pred_reflectance - pseudo_reflectances[i])) + \
                             np.mean(W_S * np.abs(pred_shading - pseudo_shadings[i]))
            loss_reg = np.mean(np.abs(pred_residual))
            total_loss = w_rgb * loss_rgb + w_intrinsic * loss_intrinsic + w_reg * loss_reg
            for p in optimizer_params:
                p -= lr * p.grad
    return reflectance_net, shading_net
```

**Parameters:**
*   $\gamma$: Gamma correction exponent used to adapt pseudo-shading to nonlinear digital image representation [S1].
*   $w_{eik}$: Eikonal loss weight (0.1) [S1].
*   $w_{curv}$: Curvature loss weight ($5 \times 10^{-4}$) [S1].
*   $w_{intrinsic}$: Intrinsic loss weight (1.0) [S1].
*   $w_{reg}$: Regularization loss weight (1.0) [S1].
*   $w_{rgb}$: RGB loss weight (Not explicitly isolated in Stage 2 snippet, but present in $L_{S2}$) [S1].

**Edge Cases and Pitfalls:**
*   **Local Minima:** The optimization may converge to trivial solutions where $R=0, S=0, Re=I$. This is mitigated by the $L_{reg}$ term which penalizes the residual norm [S1].
*   **Ill-Posed Inverse Rendering:** Traditional inverse rendering is computationally intensive and ambiguous. MLI-NeRF avoids this by using intrinsic decomposition with pseudo-labels, limiting the scope to object-specific or well-lit scenes where diffuse dominance holds [S1].
*   **Specularities:** The pseudo-reflectance generation ignores the residual component, which entangles specularities and undirected light. This may lead to inaccuracies in highly specular regions [S1].

## Subtopics

### [Physics-based Pseudo Label Generation](12712-physics-based-pseudo-label-generation.md)
A pipeline to generate high-fidelity pseudo reflectance and shading labels from a trained multi-light NeRF model to guide intrinsic decomposition training in Stage 2. Unlike previous statistical priors, this physics-based pipeline provides a more reliable basis for disentanglement by leveraging explicit geometric and illumination models. It serves as the bridge between the implicit representation learned in Stage 1 and the intrinsic-aware NeRF in Stage 2.

## Operating characteristics
*   **Time Complexity:** Stage 1 training time follows Neuralangelo [24]. Stage 2 training takes approximately 19 hours on a single Nvidia RTX 3090 for the tested datasets [S1].
*   **Memory Complexity:** Uses 3D hashing encoding and SDF storage similar to Neuralangelo. The addition of two MLPs (reflectance and shading) increases parameter count slightly, but the geometry network is frozen in Stage 2 [S1].
*   **Strengths:** Does not require ground-truth intrinsic labels; leverages 3D geometric constraints for robustness; enables relighting and intrinsic editing [S1].
*   **Limitations:** Computationally intensive due to multi-light rendering and sphere tracing; limited to scenes where diffuse reflection dominates (Lambertian assumption); struggles with highly specular or complex non-diffuse effects without residual compensation [S1].
*   **When to use:** When intrinsic decomposition and relighting are required without ground-truth labels, and the scene has controllable or known multiple light sources [S1].
*   **When not to use:** For real-time applications (training is slow); for scenes with strong specularities where the Lambertian assumption fails significantly; when ground-truth intrinsic data is available (other methods may be more direct) [S1].

## Background
MLI-NeRF was proposed to address the limitations of existing intrinsic decomposition methods that rely on statistical priors (like chromaticity) and struggle with real-world data [S1]. It builds on the idea that varying light source positions, similar to how mineralogists illuminate specimens, reveals intrinsic scene details [S1]. The method integrates multiple light information into Neural Radiance Fields, generating pseudo-labels to guide intrinsic decomposition without statistical constraints [S1]. It was validated on synthetic (Blender-based) and real-world datasets, outperforming methods like IntrinsicNeRF, InvRender, and TensoIR in intrinsic decomposition and relighting tasks [S1].

## Sources
[S1] MLI-NeRF: Multi-Light Intrinsic-Aware Neural Radiance Fields, https://arxiv.org/abs/2411.17235v1, fetched 2026-10-26
