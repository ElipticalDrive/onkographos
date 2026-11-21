# Gaussian Splatting Volume Rendering

Also called: 3D Gaussian Splatting (3DGS) Rendering, Volume Splatting

## Overview
Gaussian Splatting Volume Rendering is a differentiable rendering technique that synthesizes images by projecting 3D Gaussian distributions onto the image plane and alpha-compositing them along viewing rays [S1][S2, unofficial]. It solves the problem of high-fidelity novel view synthesis by replacing traditional triangle primitives with Gaussian splats (probability distributions), allowing for efficient tile-based rasterization and differentiable optimization of scene parameters [S1][S2, unofficial]. Unlike NeRFs which rely on dense volumetric sampling, or traditional rasterizers using triangles, 3DGS projects 3D probability distributions into 2D ellipses, enabling smooth rendering with explicit geometric primitives [S2, unofficial].

## How it works
The method reconstructs a 3D scene from posed images by optimizing a set of 3D Gaussians [S2, unofficial]. The rendering pipeline consists of loading the scene data, projecting the Gaussians to screen space, sorting them by depth, and performing front-to-back alpha compositing [S1][S2, unofficial].

### Inputs and Outputs
*   **Input:** A set of $N$ 3D Gaussians defined by their parameters, and camera parameters (view and projection matrices).
*   **Output:** A 2D image (color and optionally depth/opacity) for the given camera pose.

### Data Structures
The scene is stored as an array of `GaussianSplat` objects [S2, unofficial]. Each splat contains the following fields [S2, unofficial]:

*   `centroid`: `vec3` (world-space position).
*   `opacity`: `float` (scalar transparency).
*   `sphericalHarmonics`: `array<float, 48>` (16 coefficients $\times$ 3 RGB channels) [S2, unofficial].
*   `scale`: `array<float, 3>` (log-space floats representing the diagonal of the scaling matrix $S$) [S2, unofficial].
*   `rotation`: `array<float, 4>` (quaternion representing the rotation matrix $R$) [S2, unofficial].

### Parameters
*   **SH_COUNT:** 16 (number of spherical harmonic coefficients per channel) [S2, unofficial].
*   **SH_CHANNEL_COUNT:** 3 (RGB) [S2, unofficial].
*   **SH_FLOAT_COUNT:** 48 (total coefficients per splat) [S2, unofficial].
*   **Number of Gaussians:** Approximately 200,000–500,000 for complex scenes [S1].

### Formulas
1.  **3D Gaussian PDF:**
    $$p(\mathbf{x}) = \frac{1}{\sqrt{(2\pi)^d |\Sigma|}} \exp\left(-\frac{1}{2} (\mathbf{x}-\boldsymbol{\mu})^T \Sigma^{-1} (\mathbf{x}-\boldsymbol{\mu})\right)$$
    where $\boldsymbol{\mu}$ is the centroid and $\Sigma$ is the covariance matrix [S2, unofficial].

2.  **Covariance Factorization:**
    To ensure $\Sigma$ remains positive semi-definite (PSD) during optimization, it is parameterized as:
    $$\Sigma = R S (R S)^T$$
    where $R$ is the rotation matrix derived from the quaternion, and $S$ is a diagonal scaling matrix with entries $s_i = \exp(\text{scale}_i)$ [S2, unofficial].

3.  **Color Computation (Spherical Harmonics):**
    The color $\mathbf{c}$ for a viewing direction $\mathbf{d}$ is computed by evaluating the spherical harmonics basis functions $Y_i(\mathbf{d})$ [S2, unofficial]:
    $$\mathbf{rgb} = \text{clamp}\left(\mathbf{0.5} + \sum_{i=0}^{SH\_COUNT-1} \mathbf{shCoeff}_i \cdot Y_i(\mathbf{d}), 0.0, 1.0\right)$$
    A bias of 0.5 is added to center the output around mid-grey before clamping [S2, unofficial].

4.  **Rendering Equation (Alpha Compositing):**
    For a pixel $p$, let the intersecting Gaussians be sorted by depth $i=1 \dots N$ [S1]. The final intensity $I_f$ is [S1]:
    $$I_f \approx \sum_{i=1}^N c_i \tilde{\sigma}_i \tilde{\mathcal{G}}_i(\mathbf{p}) \prod_{j=1}^{i-1} \left(1 - \tilde{\sigma}_j \tilde{\mathcal{G}}_j(\mathbf{p})\right)$$
    where $c_i$ is the color, $\tilde{\sigma}_i$ is the opacity, and $\tilde{\mathcal{G}}_i(\mathbf{p})$ is the 2D Gaussian value at pixel $p$ [S1]. The product term represents the transmittance from previous Gaussians [S1].

### Steps
1.  **Load Scene:** Parse the `.ply` file into an array of `GaussianSplat` structures [S2, unofficial].
2.  **Transform to Camera Space:** Transform `centroid` from world space to camera space using the View matrix [S2, unofficial]. Transform the covariance matrix $\Sigma$ to camera space: $\Sigma_c = J W \Sigma W^T J^T$, where $W$ is the view matrix and $J$ is the Jacobian of the affine approximation of the perspective projection [G].
3.  **Project to 2D:** Compute the 2D covariance matrix $\Sigma_{2D}$ from $\Sigma_c$ by taking the upper-left $2\times2$ block (affine approximation) [G]. Compute the 2D Gaussian value $\tilde{\mathcal{G}}_i(\mathbf{p})$ at each pixel covered by the splat's bounding box [G].
4.  **Sort:** Sort the Gaussians intersecting each pixel (or tile) by their depth ($z$-coordinate in camera space) [S1].
5.  **Composite:** Iterate through the sorted Gaussians for each pixel and accumulate color using the alpha compositing formula [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_gaussian_splats(splats, camera):
    """
    Renders a scene of Gaussian splats for a given camera.
    splats: list of dicts with keys: centroid, opacity, sh_coeffs, scale, rotation
    camera: dict with keys: view_matrix, projection_matrix, width, height, position
    """
    H, W = camera['height'], camera['width']
    image = [[0.0, 0.0, 0.0] for _ in range(H * W)]
    
    # 1. Transform Gaussians to camera space
    camera_splats = []
    for s in splats:
        # Transform centroid
        centroid_cam = apply_view_matrix(s['centroid'], camera['view_matrix'])
        
        # Transform covariance: Sigma = R * diag(exp(scale)) * diag(exp(scale)) * R^T
        S = diag(s['scale'])
        R = quaternion_to_matrix(s['rotation'])
        Sigma_world = R @ S @ S @ R.T
        
        # Jacobian of perspective projection approximation at centroid
        J = perspective_jacobian(centroid_cam)
        W_mat = camera['view_matrix']
        
        # Sigma in camera space
        Sigma_cam = (J @ W_mat) @ Sigma_world @ (W_mat.T @ J.T)
        
        # Extract 2D covariance (upper left 2x2)
        Sigma_2d = [[Sigma_cam[0][0], Sigma_cam[0][1]], 
                    [Sigma_cam[1][0], Sigma_cam[1][1]]]
        
        camera_splats.append({
            'centroid': centroid_cam[:3],
            'opacity': s['opacity'],
            'sh_coeffs': s['sh_coeffs'],
            'sigma_2d': Sigma_2d,
            'view_dir': normalize(vec3_sub(camera['position'], s['centroid']))
        })

    # 2. Rasterize to pixels
    for y in range(H):
        for x in range(W):
            ray_dir = unproject_pixel(x, y, camera)
            candidates = find_splats_covering_pixel(camera_splats, x, y)
            candidates.sort(key=lambda s: s['centroid'][2])
            
            acc_color = [0.0, 0.0, 0.0]
            acc_alpha = 0.0
            p = [float(x), float(y)]
            
            for s in candidates:
                # Evaluate 2D Gaussian at pixel
                diff = vec2_sub(p, s['centroid'][:2])
                inv_sigma = inverse_2x2(s['sigma_2d'])
                dist_sq = dot(diff, mat_vec_2x2_mul(inv_sigma, diff))
                g_val = exp(-0.5 * dist_sq)
                
                # Evaluate Color using SH
                color = evaluate_sh(s['sh_coeffs'], ray_dir)
                
                # Contribution
                sigma = s['opacity'] * g_val
                weight = sigma * (1.0 - acc_alpha)
                
                acc_color = [acc_color[k] + color[k] * weight for k in range(3)]
                acc_alpha += weight
                
                if acc_alpha > 0.99:
                    break
            
            image[y * W + x] = acc_color
            
    return image

def evaluate_sh(coeffs, direction):
    """Evaluates spherical harmonics for a given direction."""
    rgb = [0.5, 0.5, 0.5]
    for i in range(16):
        basis_val = spherical_harmonic_basis(i, direction)
        rgb[0] += coeffs[i] * basis_val
        rgb[1] += coeffs[16 + i] * basis_val
        rgb[2] += coeffs[32 + i] * basis_val
    return [max(0.0, min(1.0, c)) for c in rgb]

# Helper definitions (standard math/vector operations)
def apply_view_matrix(centroid, view_mat):
    return [sum(centroid[j]*view_mat[j][i] + view_mat[3][i] for j in range(3)) for i in range(3)]

def diag(vals):
    return [[vals[i] if i==j else 0.0 for j in range(3)] for i in range(3)]

def quaternion_to_matrix(q):
    x, y, z, w = q
    return [
        [1-2*(y*y+z*z), 2*(x*y-w*z), 2*(x*z+w*y)],
        [2*(x*y+w*z), 1-2*(x*x+z*z), 2*(y*z-w*x)],
        [2*(x*z-w*y), 2*(y*z+w*x), 1-2*(x*x+y*y)]
    ]

def perspective_jacobian(centroid_cam):
    z = centroid_cam[2]
    return [[1.0/z, 0.0, 0.0], [0.0, 1.0/z, 0.0]]

def unproject_pixel(x, y, camera):
    # Simplified ray direction calculation for orthographic/perspective approximation
    return normalize(vec3(x - camera['width']/2, y - camera['height']/2, -1.0))

def find_splats_covering_pixel(splats, x, y):
    # Returns splats whose 2D bounding box covers pixel (x,y)
    return [s for s in splats if is_inside_bounding_box(s, x, y)]

def is_inside_bounding_box(splat, x, y):
    # Approximate bounding box check using 2D covariance eigenvalues
    # In practice, this checks if the pixel falls within the projected ellipse
    return True  # Placeholder for bounding volume hierarchy or tile-based culling [G]

def vec3_sub(a, b): return [a[i]-b[i] for i in range(3)]
def normalize(v):
    l = (sum(x*x for x in v))**0.5
    return [x/l for x in v]
def dot(a, b): return sum(a[i]*b[i] for i in range(len(a)))
def mat_vec_2x2_mul(M, v): return [sum(M[i][j]*v[j] for j in range(2)) for i in range(2)]
def inverse_2x2(M):
    det = M[0][0]*M[1][1] - M[0][1]*M[1][0]
    return [[M[1][1]/det, -M[0][1]/det], [-M[1][0]/det, M[0][0]/det]]
def spherical_harmonic_basis(i, direction):
    # Standard SH basis evaluation for degree 0-3
    # Returns a float based on direction angles (theta, phi)
    return 1.0  # Simplified for pseudocode completeness [G]
```

### Edge Cases and Pitfalls
*   **Aliasing:** High-frequency content can cause aliasing artifacts. 3DGS uses a dilation-based filter to mitigate this, but it may fail with varying focal lengths or camera distances [S1].
*   **Covariance Validity:** Directly updating the entries of the covariance matrix $\Sigma$ during training does not guarantee it remains positive semi-definite. Using the scale and rotation parameterization ensures $\Sigma$ is always valid [S2, unofficial].
*   **View-Dependent Color:** If spherical harmonics are not used (e.g., in some volume rendering adaptations), the scene appears dimmer and less vibrant because it lacks viewpoint-dependent color shifts [S3, unofficial].

## Subtopics

### [Surface Normal Rendering via External Channel](2099-surface-normal-rendering-via-external-channel.md)
Enables rendering of surface normals from 3D Gaussian splats by treating normals as an auxiliary attribute, bypassing the lack of native geometric surface information in volumetric Gaussian ellipsoids.

## Operating characteristics
*   **Time Complexity:** Rendering scales with the number of pixels and the average number of splats per pixel. Tile-based rasterization reduces the constant factor significantly [S1].
*   **Memory Complexity:** Memory scales linearly with the number of Gaussians, requiring 200,000–500,000 primitives for complex scenes, resulting in significant memory and storage demands compared to NeRFs [S1].
*   **Performance:** Uses a tile-based rasterization strategy for parallelization to accelerate volume rendering [S1].
*   **Strengths:** High visual fidelity, explicit scene representation, differentiable optimization [S1][S2, unofficial].
*   **Limitations:** High memory usage, limited ability to handle highly reflective surfaces (due to emission-absorption model baking lighting into SH), and potential aliasing artifacts [S1][S3, unofficial].

## Background
3D Gaussian Splatting was introduced as a technique to reconstruct 3D scenes from posed images using Gaussian splats instead of triangles [S2, unofficial]. It builds on the principles of volume rendering but adapts them for differentiable optimization of explicit primitives [S1]. The method has since been extended for various applications including surface reconstruction and avatar creation [S1].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] 3D Gaussian Splatting in a Weekend, https://bfeldman.me/3dgs-weekend/, fetched 2026-10-26 (unofficial)
[S3] ShengzhiWu/taichi-volume-renderer, https://github.com/ShengzhiWu/taichi-volume-renderer, fetched 2026-10-26 (unofficial)
