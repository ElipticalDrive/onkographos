# Discontinuity-aware Gaussian Splatting

Also called: DisC-GS

## Overview
Discontinuity-aware Gaussian Splatting (DisC-GS) is a framework that enables 3D Gaussian Splatting to represent and render sharp boundaries and discontinuities, addressing the limitation of standard Gaussian Splatting which tends to blur such features [S1]. It differs from sibling methods like [Planar Gaussian Splatting](16706-planar-gaussian-splatting.md) by explicitly modeling boundaries as cubic Bézier curves within the Gaussian's local subspace and applying a "pre-scissoring" operation during rendering, rather than relying solely on geometric constraints or mesh extraction [S1, 14095-mesh-extraction-and-representation.md].

## How it works
**Inputs:** A set of $N$ 3D Gaussians $\{G_i\}_{i=1}^N$, where each Gaussian $G_i$ is defined by mean $\boldsymbol{\mu}_i$, rotation $\mathbf{R}_i$, scale $\mathbf{S}_i$, opacity $\alpha_i$, and color $\mathbf{c}_i$. For 2D Gaussian Splatting, the last column of $\mathbf{S}_i$ is fixed to zero [S1]. The input also includes the camera projection matrix $\mathbf{P}$ and viewing transformation matrix $\mathbf{W}$, and a user-defined hyperparameter $M$ (number of Bézier curves per Gaussian) [S1].

**Outputs:** A rendered image $C(p)$ for each pixel $p$, where discontinuities are preserved [S1].

**Data Structures:**
Each Gaussian $G_i$ is augmented with an additional attribute $\mathbf{c}_{\text{curve}} \in \mathbb{R}^{4M \times 2}$ [S1].
*   **Layout:** $\mathbf{c}_{\text{curve}}$ stores $4M$ points in the 2D coordinate system defined by the Gaussian's center $\boldsymbol{\mu}_i$ (origin), rotation matrix column $\mathbf{r}_{1,i}$ (x-axis), and rotation matrix column $\mathbf{r}_{2,i}$ (y-axis) [S1].
*   **Interpretation:** These points serve as control points for $M$ cubic Bézier curves representing the boundaries of the Gaussian's projection [S1].

**Numbered Steps:**

1.  **Local to World Transformation:** For each Gaussian $G_i$, transform the control points from its local 2D subspace to the 3D world space. Let $\mathbf{c}_{\text{curve}}[i, :]$ denote the $i$-th point in the local attribute (for $i \in \{0, \dots, 4M-1\}$). The 3D control points $\mathbf{c}_{\text{3D\_curve}}[i]$ are computed as:
    $$ \mathbf{c}_{\text{3D\_curve}}[i] = \boldsymbol{\mu}_i + \mathbf{c}_{\text{curve}}[i, 0] \times \mathbf{r}_{1,i} + \mathbf{c}_{\text{curve}}[i, 1] \times \mathbf{r}_{2,i} $$
    Note: $\mathbf{r}_{1,i}$ and $\mathbf{r}_{2,i}$ are unit vectors, so normalization is omitted [S1].

2.  **Projection to Image Plane:** Project the 3D control points onto the image plane to obtain the 2D control points $\mathbf{c}_{\text{2D\_curve}}[i]$:
    $$ \mathbf{c}_{\text{2D\_curve}}[i] = \mathbf{P} \mathbf{W} \mathbf{c}_{\text{3D\_curve}}[i] $$
    where $\mathbf{P}$ is the projective transformation matrix and $\mathbf{W}$ is the viewing transformation matrix [S1].

3.  **Implicit Curve Representation:** For each of the $M$ cubic Bézier curves defined by four ordered control points $\boldsymbol{\omega}_0, \boldsymbol{\omega}_1, \boldsymbol{\omega}_2, \boldsymbol{\omega}_3$ (where $\boldsymbol{\omega}_k = \mathbf{c}_{\text{2D\_curve}}[4j+k]$ for curve index $j$), compute the implicit representation $B_{\text{imp}}(x, y)$:
    $$ B_{\text{imp}}(x, y) = \gamma_{xxx} x^3 + \gamma_{xxy} x^2 y + \gamma_{xyy} x y^2 + \gamma_{yyy} y^3 + \gamma_{xx} x^2 + \gamma_{xy} xy + \gamma_{yy} y^2 + \gamma_x x + \gamma_y y + \gamma_0 = 0 $$
    The coefficients $\gamma$ are obtained via $O(1)$ arithmetic operations on the control point coordinates [S1].

4.  **Indicator Function Construction:** Define the single-curve indicator function $g_{\text{sc}}$ for a pixel $p=(x_p, y_p)$:
    $$ g_{\text{sc}}(\boldsymbol{\omega}_0, \boldsymbol{\omega}_1, \boldsymbol{\omega}_2, \boldsymbol{\omega}_3; p) = \begin{cases} 1 & \text{if } B_{\text{imp}}(x_p, y_p) > 0 \\ 0 & \text{otherwise} \end{cases} $$
    For $M > 1$ curves, the combined indicator function $g(p)$ is the product of individual indicators:
    $$ g(p) = \prod_{j=0}^{M-1} g_{\text{sc}}(\mathbf{c}_{\text{2D\_curve}}[4j], \mathbf{c}_{\text{2D\_curve}}[4j+1], \mathbf{c}_{\text{2D\_curve}}[4j+2], \mathbf{c}_{\text{2D\_curve}}[4j+3]; p) $$
    A pixel is "scissored out" ($g(p)=0$) if it is outside at least one of the $M$ curves [S1].

5.  **Discontinuity-Aware Rendering:** Compute the color $C(p)$ using a modified $\alpha$-blending function:
    $$ C(p) = \sum_{i=1}^{N} \mathbf{c}_i \beta_i \prod_{j=1}^{i-1} (1 - \beta_j) $$
    where the effective opacity $\beta_i$ for the $i$-th Gaussian is:
    $$ \beta_i = \alpha_i g_i(p) \exp\left(-\frac{1}{2} (\mathbf{p} - \boldsymbol{\mu}_{i,2D})^\top (\boldsymbol{\Sigma}_{i,2D})^{-1} (\mathbf{p} - \boldsymbol{\mu}_{i,2D})\right) $$
    Here, $\boldsymbol{\mu}_{i,2D}$ and $\boldsymbol{\Sigma}_{i,2D}$ are the projected mean and covariance of the $i$-th Gaussian, and $g_i(p)$ is the indicator function for the $i$-th Gaussian [S1].

6.  **Gradient Approximation:** During backpropagation, use a Bézier-boundary gradient approximation strategy to enable gradients to pass through the discontinuous indicator function $g(p)$, maintaining differentiability for learning $\mathbf{c}_{\text{curve}}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def cubic_bezier_implicit_coeffs(w0, w1, w2, w3):
    """
    Computes the coefficients gamma for the implicit representation of a cubic Bezier curve.
    w0, w1, w2, w3: numpy arrays of shape (2,) representing control points.
    Returns: dict of gamma coefficients.
    Note: Implementation details for O(1) arithmetic operations are omitted as per source [S1] 
    which refers to supplementary material for exact derivation.
    """
    # Placeholder for the specific algebraic derivation provided in [S1] supplementary
    # This function must return gamma_xxx, gamma_xxy, etc.
    raise NotImplementedError("See [S1] supplementary for exact O(1) arithmetic derivation")

def bezier_indicator(p, coeffs):
    """
    Evaluates the implicit function B_imp(x, y) for a pixel p.
    p: numpy array of shape (2,) [x, y]
    coeffs: dict of gamma coefficients
    Returns: float value of B_imp(x, y)
    """
    x, y = p
    gamma = coeffs
    val = (gamma['gamma_xxx'] * x**3 +
           gamma['gamma_xxy'] * x**2 * y +
           gamma['gamma_xyy'] * x * y**2 +
           gamma['gamma_yyy'] * y**3 +
           gamma['gamma_xx'] * x**2 +
           gamma['gamma_xy'] * x * y +
           gamma['gamma_yy'] * y**2 +
           gamma['gamma_x'] * x +
           gamma['gamma_y'] * y +
           gamma['gamma_0'])
    return val

def pre_scissor_render(gaussians, camera_matrix_P, camera_matrix_W, M):
    """
    Renders an image using Discontinuity-aware Gaussian Splatting.
    gaussians: list of N Gaussian objects. Each has:
        mu: (3,) mean
        R: (3,3) rotation matrix
        alpha: scalar opacity
        color: (3,) color
        c_curve: (4M, 2) control points in local 2D space
    camera_matrix_P: (4,4) projection matrix
    camera_matrix_W: (4,4) viewing matrix
    M: int, number of Bezier curves per Gaussian
    Returns: rendered_image (H, W, 3)
    """
    # Assume image dimensions H, W are known or derived from camera frustum
    # For simplicity, we iterate pixels. In practice, rasterization is used.
    
    # Precompute projected Gaussians
    projected_gaussians = []
    for g in gaussians:
        # Project mean
        mu_2d = camera_matrix_P @ camera_matrix_W @ np.append(g.mu, 1)
        mu_2d = mu_2d[:2] / mu_2d[2] # Perspective divide
        
        # Project covariance (simplified Jacobian J omitted for brevity, assume standard 3DGS projection)
        # Sigma_2D = J @ W @ Sigma_3D @ W.T @ J.T
        # This step relies on standard 3DGS projection logic [S1]
        projected_gaussians.append({
            'mu_2d': mu_2d,
            'alpha': g.alpha,
            'color': g.color,
            'c_curve': g.c_curve
        })
        
    # Initialize image
    # Note: Full rasterization loop is complex; this pseudocode outlines the core DisC-GS logic
    # for a single pixel p.
    
    def render_pixel(p):
        color_sum = np.zeros(3)
        alpha_sum = 0.0
        
        # Sort Gaussians by depth (standard 3DGS step)
        # ... sorting logic ...
        
        for pg in projected_gaussians:
            # 1. Transform local control points to 3D
            # r1, r2 are first two columns of R
            r1 = pg['R'][:, 0]
            r2 = pg['R'][:, 1]
            
            c_3d_curve = np.zeros((4*M, 3))
            for i in range(4*M):
                c_3d_curve[i] = pg['mu'] + pg['c_curve'][i, 0] * r1 + pg['c_curve'][i, 1] * r2
                
            # 2. Project control points to 2D
            c_2d_curve = np.zeros((4*M, 2))
            for i in range(4*M):
                pt_3d_hom = np.append(c_3d_curve[i], 1)
                pt_2d_hom = camera_matrix_P @ camera_matrix_W @ pt_3d_hom
                c_2d_curve[i] = pt_2d_hom[:2] / pt_2d_hom[2]
                
            # 3. Compute Indicator Function g(p)
            g_val = 1.0
            for j in range(M):
                w0 = c_2d_curve[4*j]
                w1 = c_2d_curve[4*j+1]
                w2 = c_2d_curve[4*j+2]
                w3 = c_2d_curve[4*j+3]
                
                coeffs = cubic_bezier_implicit_coeffs(w0, w1, w2, w3)
                b_imp = bezier_indicator(p, coeffs)
                
                if b_imp <= 0:
                    g_val = 0.0
                    break
            
            if g_val == 0:
                continue
                
            # 4. Compute Beta
            # Sigma_2D calculation omitted, assume standard 3DGS projection
            # dist_sq = (p - mu_2d).T @ inv(Sigma_2D) @ (p - mu_2d)
            # beta = alpha * g_val * exp(-0.5 * dist_sq)
            
            # Simplified beta for pseudocode structure
            beta = pg['alpha'] * g_val * 1.0 # Placeholder for Gaussian term
            
            # 5. Alpha Blending
            color_sum += pg['color'] * beta * (1 - alpha_sum)
            alpha_sum += beta * (1 - alpha_sum)
            
        return color_sum

    # Iterate over all pixels
    # rendered_image = np.zeros((H, W, 3))
    # for y in range(H):
    #     for x in range(W):
    #         p = np.array([x, y])
    #         rendered_image[y, x] = render_pixel(p)
        
    return None # Placeholder
```

**Parameters:**
*   $M$: User-defined hyperparameter representing the number of Bézier curves per Gaussian distribution [S1].

**Edge Cases and Pitfalls:**
*   **Non-differentiability:** The indicator function $g(p)$ is piecewise constant, making the forward rendering process non-differentiable. Naive integration breaks backpropagation [S1].
*   **Zero Gradients:** Even in differentiable segments, the gradient of $g(\cdot)$ with respect to its inputs is zero, which fails to guide the learning of $\mathbf{c}_{\text{curve}}$ without the Bézier-boundary gradient approximation strategy [S1].
*   **Viewpoint Inconsistency:** Using 3D Gaussians can result in viewpoint inconsistency. DisC-GS fixes the last column of the scale matrix $\mathbf{S}$ to be a zero vector for 2D Gaussians to mitigate this [S1].

## Operating characteristics
*   **Time Complexity:** The implicitization of cubic Bézier curves and evaluation of the indicator function is $O(1)$ per curve per pixel [S1]. With $M$ curves per Gaussian and $N$ Gaussians, the overhead per pixel is proportional to $N \times M$.
*   **Memory Complexity:** Each Gaussian stores an additional attribute $\mathbf{c}_{\text{curve}} \in \mathbb{R}^{4M \times 2}$, increasing memory usage by $8M$ floats per Gaussian [S1].
*   **Strengths:** Enables accurate rendering of sharp boundaries and discontinuities, which standard Gaussian Splatting blurs [S1].
*   **Limitations:** Requires careful handling of differentiability via the gradient approximation strategy [S1]. Performance depends on the choice of $M$ [S1].

## Background
Discontinuity-aware Gaussian Splatting (DisC-GS) was proposed to address the key intrinsic limitation of Gaussian Splatting: its inability to render discontinuities and boundaries accurately [S1]. It is the first effort to enable Gaussian Splatting to represent and render boundaries properly in its image rendering pipeline [S1]. The method builds on the standard 3D Gaussian Splatting representation and incorporates insights from cubic Bézier curves [S1].

## Sources
[S1] DisC-GS: Discontinuity-aware Gaussian Splatting, https://arxiv.org/abs/2405.15196v2, fetched 2026-10-26
