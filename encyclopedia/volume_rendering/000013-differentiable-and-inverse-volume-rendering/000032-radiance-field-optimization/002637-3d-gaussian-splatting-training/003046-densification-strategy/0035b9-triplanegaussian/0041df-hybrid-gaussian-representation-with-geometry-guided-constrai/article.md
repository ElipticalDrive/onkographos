# Hybrid Gaussian Representation with Geometry-Guided Constraints

Also called: Adaptive Hybrid Representation and TensoSDF Constraints, Hybrid Gaussian Splatting with Geometry Guidance

## Overview
This method is a 3D Gaussian Splatting rendering technique that adaptively selects between 2D and 3D Gaussian primitives based on local geometric cues to reduce redundancy in planar structures while maintaining volumetric accuracy [S1]. It integrates TensoSDF for global geometric consistency, enforcing implicit surface constraints during optimization [S1]. Unlike standard 3DGS which uses only 3D ellipsoids, or 2DGS which uses only tangent planes, this method dynamically switches representations to better fit both thin surfaces and volumetric details [S1]. It builds on [TriplaneGaussian](7558-triplanegaussian.md) by introducing explicit geometry-guided constraints via TensoSDF rather than relying solely on tri-plane initialization.

## How it works

### Inputs and Outputs
**Inputs:**
*   Input point cloud (for initialization and surface normals).
*   Camera poses and images for training.
*   Bounding box definition (default: 1m³).

**Outputs:**
*   Optimized hybrid Gaussian field (mix of 2D and 3D Gaussians).
*   Optimized TensoSDF tensor factors.
*   Rendered images with improved geometric consistency and PSNR.

### Data Structures

**1. 3D Gaussian Primitive**
*   Center: $\mu \in \mathbb{R}^3$
*   Covariance: $\Sigma \in \mathbb{R}^{3 \times 3}$ (symmetric positive definite)
*   Opacity: $\alpha \in \mathbb{R}$
*   Color: $c \in \mathbb{R}^3$ (or SH coefficients)
*   Type: `3D`

**2. 2D Gaussian Primitive**
*   Center: $\mu \in \mathbb{R}^3$
*   Tangent Basis: $T = [t_1, t_2] \in \mathbb{R}^{3 \times 2}$ (orthonormal vectors spanning the tangent plane)
*   Surface Normal: $n \in \mathbb{R}^3$ (derived from $T$ or stored explicitly)
*   Scale Matrix: $\Lambda = \text{diag}(\sigma_1^2, \sigma_2^2) \in \mathbb{R}^{2 \times 2}$
*   Opacity: $\alpha \in \mathbb{R}$
*   Color: $c \in \mathbb{R}^3$
*   Type: `2D`

**3. TensoSDF Tensor Factors**
*   $A_x \in \mathbb{R}^{N_x \times r}$
*   $A_y \in \mathbb{R}^{N_y \times r}$
*   $A_z \in \mathbb{R}^{N_z \times r}$
*   Parameters: $N_x=N_y=N_z=128$, $r=32$.

### Parameters
*   $\tau$: Curvature threshold for initial type selection.
*   $\tau_1$: Gradient magnitude threshold for flat geometry detection ($\tau_1 = 0.1$) [S1].
*   $\tau_2$: SDF value threshold for proximity to surface ($\tau_2 = 0.05$ m) [S1].
*   $\eta$: Update step size for Gaussian position fine-tuning.
*   $\lambda$: Weight for gradient consistency in TensoSDF loss ($\lambda = 10$) [S1].
*   AR Threshold: Anisotropy Ratio threshold for conversion ($AR < 0.2$) [S1].
*   $\sigma_n$: Small variance added during 2D-to-3D conversion ($\sigma_n = 10^{-4}$) [S1].
*   $N_{neigh}$: Number of neighbors for 2D-to-3D angle check ($N_{neigh} = 10$) [S1].
*   $\theta_{thresh}$: Mean angle threshold for 2D-to-3D conversion ($30^\circ$) [S1].

### Steps

1.  **Initialization**:
    *   Establish a complete 3D Gaussian ellipsoidal field based on the input point cloud following the standard 3DGS workflow [S1].
    *   Initialize TensoSDF tensor factors $A_x, A_y, A_z$ with random values from $\mathcal{N}(0, 0.01)$ [S1].

2.  **Curvature Estimation (Initial Type Selection)**:
    *   For each 3D Gaussian center $\mu$, perform PCA on the local point cluster to extract eigenvalues $\lambda_1 \ge \lambda_2 \ge \lambda_3$ of the cluster covariance matrix [S1].
    *   Calculate local curvature: $\kappa = (\lambda_2 + \lambda_3) / \lambda_1$ [S1].
    *   If $\kappa < \tau$, convert the 3D Gaussian to a 2D Gaussian (see "2D Gaussian Initialization"). Otherwise, keep as 3D [S1].

3.  **2D Gaussian Initialization**:
    *   Set surface normal $n$ as the eigenvector corresponding to the smallest eigenvalue $\lambda_3$ [S1].
    *   Derive tangent plane basis $T = [t_1, t_2]$ via Gram-Schmidt orthogonalization of two random vectors orthogonal to $n$ [S1].
    *   Initialize scale matrix $\Lambda = \text{diag}(\sigma_p^2, \sigma_p^2)$ where $\sigma_p$ is an appropriate initial value [S1].
    *   Keep color and opacity unchanged from the original 3D Gaussian [S1].

4.  **Adaptive Conversion (3D to 2D)**:
    *   For a 3D Gaussian at $\mu$, compute SDF value $d = F(\mu)$ and gradient $g = \nabla F(\mu) / \|\nabla F(\mu)\|$ [S1].
    *   Check proximity: If $\|F(\mu)\| < \tau_2$:
        *   Check flatness: If $\|\nabla F(\mu)\| < \tau_1$:
            *   Calculate Anisotropy Ratio: $AR(\mu) = \min(\sigma_1, \sigma_2, \sigma_3) / \max(\sigma_1, \sigma_2, \sigma_3)$, where $\sigma_i$ are eigenvalues of $\Sigma$ [S1].
            *   If $AR(\mu) < 0.2$, convert to 2D Gaussian using the initialization logic in Step 3 [S1].

5.  **Adaptive Conversion (2D to 3D)**:
    *   For a 2D Gaussian at $\mu$ with normal $n$:
        *   Find $N_{neigh}=10$ neighboring Gaussians via k-nearest neighbors [S1].
        *   Compute mean angle $\bar{\theta}$ between $n$ and neighbors' normals $n_i$:
            $$ \bar{\theta} = \frac{1}{N_{neigh}} \sum_{i=1}^{N_{neigh}} \arccos\left( \frac{n \cdot n_i}{\|n\| \|n_i\|} \right) $$
        *   If $\bar{\theta} > 30^\circ$, convert to 3D Gaussian [S1].
        *   Initialize new covariance: $\Sigma' = R(n) \Lambda R(n)^\top + 2\sigma_n^2 n n^\top$, where $\sigma_n = 10^{-4}$ [S1].

6.  **Gaussian Fine-Tuning (Geometry Guidance)**:
    *   For each Gaussian center $\mu$, compute SDF value $d = F(\mu)$ and normalized gradient $g = \nabla F(\mu) / \|\nabla F(\mu)\|$ [S1].
    *   Update position: $\mu' = \mu - \eta \cdot d \cdot g$ [S1].
    *   Update covariance: $\Sigma' = R(g) \Sigma R(g)^\top$, where $R(g)$ aligns the principal axis with $g$ [S1].
    *   Compute $R(g)$ using Rodrigues' formula:
        $$ R(g) = \cos\theta \cdot I + (1 - \cos\theta) gg^\top + \sin\theta \cdot [g]_\times $$
        where $\theta = \arccos(v_{max}^\top g)$, $v_{max}$ is the eigenvector of $\Sigma$ corresponding to the largest eigenvalue, and $[g]_\times$ is the skew-symmetric matrix of $g$ [S1].

7.  **Training**:
    *   Optimize TensoSDF tensor factors and Gaussian field parameters (positions, covariances, opacities, colors) in parallel [S1].
    *   Compute rendering loss (e.g., L1/SSIM) between rendered and input images [S1].
    *   Compute TensoSDF loss $\mathcal{L}_{SDF}$ (see Formulas) [S1].
    *   Backpropagate gradients end-to-end [S1].

### Formulas

**3D Gaussian Density:**
$$ G_{3D}(x) = \exp\left(-\frac{1}{2} (x - \mu)^\top \Sigma^{-1} (x - \mu)\right) $$
where $\mu$ is the center and $\Sigma$ is the covariance matrix [S1].

**2D Gaussian Density:**
$$ G_{2D}(x) = \exp\left(-\frac{1}{2} (T^\top (x - \mu))^\top \Lambda^{-1} (T^\top (x - \mu))\right) $$
where $T = [t_1, t_2]$ is the tangent basis and $\Lambda = \text{diag}(\sigma_1^2, \sigma_2^2)$ is the scale matrix on the tangent plane [S1].

**Local Curvature:**
$$ \kappa = \frac{\lambda_2 + \lambda_3}{\lambda_1} $$
where $\lambda_1 \ge \lambda_2 \ge \lambda_3$ are eigenvalues from PCA on local point clusters [S1].

**Anisotropy Ratio:**
$$ AR(x) = \frac{\min(\sigma_1, \sigma_2, \sigma_3)}{\max(\sigma_1, \sigma_2, \sigma_3)} $$
where $\sigma_1, \sigma_2, \sigma_3$ are eigenvalues of $\Sigma$ sorted descending [S1].

**TensoSDF SDF Function:**
$$ F(x, y, z) \approx \sum_{i=1}^{r} A_x(x, i) \cdot A_y(y, i) \cdot A_z(z, i) $$
where $r$ is the decomposition rank and $A_x, A_y, A_z$ are tensor factors [S1].

**TensoSDF Loss Function:**
$$ \mathcal{L}_{SDF} = \frac{1}{M} \sum_{m=1}^{M} |F(\mu_m)|^2 + \lambda \cdot \frac{1}{M} \sum_{m=1}^{M} \|\nabla F(\mu_m) - n_m\|^2 $$
where $M$ is the number of surface points, $\mu_m$ are Gaussian centers near the surface, $n_m$ are precomputed point cloud normals, and $\lambda = 10$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_tensosdf(Nx=128, Ny=128, Nz=128, r=32):
    """Initialize TensoSDF tensor factors."""
    Ax = np.random.normal(0, 0.01, (Nx, r))
    Ay = np.random.normal(0, 0.01, (Ny, r))
    Az = np.random.normal(0, 0.01, (Nz, r))
    return Ax, Ay, Az

def compute_sdf(x, y, z, Ax, Ay, Az):
    """Compute SDF value using CP decomposition."""
    # x, y, z are scalars or arrays of coordinates
    # Interpolate Ax, Ay, Az at coordinates if necessary (not specified in source, assumed continuous)
    # For implementation, one would typically use trilinear interpolation on the tensor grids.
    # Here we assume a helper function `interpolate(tensor, coord)` exists.
    val = 0.0
    for i in range(Ax.shape[1]):
        val += interpolate(Ax, x)[:, i] * interpolate(Ay, y)[:, i] * interpolate(Az, z)[:, i]
    return val

def compute_sdf_gradient(x, y, z, Ax, Ay, Az):
    """Compute gradient of SDF."""
    # Compute partial derivatives dF/dx, dF/dy, dF/dz
    # Similar to compute_sdf but using gradients of tensor factors
    grad_x = 0.0
    grad_y = 0.0
    grad_z = 0.0
    for i in range(Ax.shape[1]):
        grad_x += d_interpolate(Ax, x, axis=0)[:, i] * interpolate(Ay, y)[:, i] * interpolate(Az, z)[:, i]
        grad_y += interpolate(Ax, x)[:, i] * d_interpolate(Ay, y, axis=0)[:, i] * interpolate(Az, z)[:, i]
        grad_z += interpolate(Ax, x)[:, i] * interpolate(Ay, y)[:, i] * d_interpolate(Az, z, axis=0)[:, i]
    return np.array([grad_x, grad_y, grad_z])

def pca_local_cluster(points):
    """Perform PCA on local point cluster."""
    cov = np.cov(points.T)
    eigenvalues, eigenvectors = np.linalg.eigh(cov)
    # Sort descending
    idx = np.argsort(eigenvalues)[::-1]
    eigenvalues = eigenvalues[idx]
    eigenvectors = eigenvectors[:, idx]
    return eigenvalues, eigenvectors

def gram_schmidt(v1, v2):
    """Orthogonalize v2 against v1."""
    u = v2 - np.dot(v2, v1) * v1
    return u / np.linalg.norm(u)

def init_2d_gaussian(mu, eigenvalues, eigenvectors):
    """Initialize 2D Gaussian from PCA results."""
    # n is eigenvector for smallest eigenvalue (lambda3)
    n = eigenvectors[:, 2]
    
    # Generate two random vectors orthogonal to n
    # Simple method: cross product with arbitrary vector not parallel to n
    random_vec = np.random.rand(3)
    if np.abs(np.dot(random_vec, n)) > 0.9:
        random_vec = np.array([1, 0, 0]) # Fallback
    t1 = np.cross(n, random_vec)
    t1 = t1 / np.linalg.norm(t1)
    t2 = np.cross(n, t1)
    
    T = np.column_stack((t1, t2))
    
    # Initialize scale matrix
    sigma_p = 0.1 # Example value, source says "appropriate initial value"
    Lambda = np.diag([sigma_p**2, sigma_p**2])
    
    return T, Lambda, n

def rodrigues_rotation(g, v_max):
    """Compute rotation matrix R(g) aligning v_max to g."""
    # g is normalized gradient
    # v_max is principal axis (eigenvector of Sigma for largest eigenvalue)
    
    cos_theta = np.dot(v_max, g)
    # Clamp for numerical stability
    cos_theta = np.clip(cos_theta, -1.0, 1.0)
    sin_theta = np.sqrt(1 - cos_theta**2)
    
    if sin_theta < 1e-6:
        return np.eye(3)
        
    g = g / np.linalg.norm(g)
    
    # Skew-symmetric matrix of g
    g_x = np.array([
        [0, -g[2], g[1]],
        [g[2], 0, -g[0]],
        [-g[1], g[0], 0]
    ])
    
    R = np.cos(np.arccos(cos_theta)) * np.eye(3) + \
        (1 - np.cos(np.arccos(cos_theta))) * np.outer(g, g) + \
        np.sin(np.arccos(cos_theta)) * g_x
        
    return R

def convert_3d_to_2d(gaussian, point_cloud_cluster, tau):
    """Convert 3D Gaussian to 2D if curvature is low."""
    eigenvalues, eigenvectors = pca_local_cluster(point_cloud_cluster)
    lambda1, lambda2, lambda3 = eigenvalues[0], eigenvalues[1], eigenvalues[2]
    
    kappa = (lambda2 + lambda3) / lambda1
    
    if kappa < tau:
        T, Lambda, n = init_2d_gaussian(gaussian.mu, eigenvalues, eigenvectors)
        gaussian.type = '2D'
        gaussian.T = T
        gaussian.Lambda = Lambda
        gaussian.n = n
        # Color and opacity unchanged
        return True
    return False

def convert_3d_to_2d_adaptive(gaussian, F, tau1, tau2):
    """Adaptive conversion based on TensoSDF."""
    mu = gaussian.mu
    d = F(mu)
    
    if np.abs(d) < tau2:
        grad = compute_sdf_gradient(mu[0], mu[1], mu[2], ...) # Requires full F impl
        grad_norm = np.linalg.norm(grad)
        if grad_norm < tau1:
            # Calculate Anisotropy Ratio
            eigenvalues = np.linalg.eigvalsh(gaussian.Sigma)
            eigenvalues = np.sort(eigenvalues)[::-1]
            sigma1, sigma2, sigma3 = eigenvalues
            AR = np.min([sigma1, sigma2, sigma3]) / np.max([sigma1, sigma2, sigma3])
            
            if AR < 0.2:
                # Perform conversion (similar to init_2d_gaussian but using local geometry)
                # Note: Source implies using local point cluster PCA for normal, 
                # but here we might use SDF gradient or local cluster. 
                # Source says "reapply PCA on local point clusters" for initial, 
                # and for adaptive it checks SDF cues. 
                # We assume we still need a normal for 2D init. 
                # Let's use SDF gradient as normal for consistency with fine-tuning?
                # Source says: "For a 3D Gaussian... check if close to surface... 
                # if flat... calculate AR... convert". 
                # It doesn't explicitly state how to get T for adaptive conversion, 
                # but implies using the same 2D parameterization. 
                # We will use the SDF gradient as the normal for the tangent plane.
                n = grad / grad_norm
                # Generate T from n
                random_vec = np.random.rand(3)
                if np.abs(np.dot(random_vec, n)) > 0.9:
                    random_vec = np.array([1, 0, 0])
                t1 = np.cross(n, random_vec)
                t1 = t1 / np.linalg.norm(t1)
                t2 = np.cross(n, t1)
                T = np.column_stack((t1, t2))
                
                sigma_p = 0.1
                Lambda = np.diag([sigma_p**2, sigma_p**2])
                
                gaussian.type = '2D'
                gaussian.T = T
                gaussian.Lambda = Lambda
                gaussian.n = n
                return True
    return False

def convert_2d_to_3d(gaussian, neighbors):
    """Convert 2D Gaussian to 3D if normals are inconsistent."""
    n = gaussian.n
    angles = []
    for neighbor in neighbors:
        n_i = neighbor.n
        cos_theta = np.dot(n, n_i) / (np.linalg.norm(n) * np.linalg.norm(n_i))
        cos_theta = np.clip(cos_theta, -1.0, 1.0)
        angles.append(np.arccos(cos_theta))
        
    mean_angle = np.mean(angles)
    
    if mean_angle > np.deg2rad(30):
        # Initialize Sigma
        # Sigma = R(n) Lambda R(n)^T + 2 sigma_n^2 n n^T
        # R(n) aligns principal axis with n. 
        # Since 2D Gaussian has no principal axis in 3D (degenerate), 
        # we treat n as the principal axis for the rotation?
        # Source: "Sigma = R(n) Lambda R(n)^T + 2 sigma_n^2 n n^T"
        # R(n) is a rotation matrix. If we want the principal axis to be n,
        # we can construct R(n) such that R(n) * [1,0,0]^T = n (assuming Lambda is aligned with x).
        # However, Lambda is 2x2. We need to embed it.
        # Let's assume Lambda corresponds to the tangent plane axes.
        # We need a 3x3 rotation matrix that maps the tangent basis to T and normal to n.
        # Actually, the formula suggests R(n) rotates the *original* 2D basis.
        # Let's construct R such that it maps the standard basis e1, e2 to t1, t2 and e3 to n.
        # But we only have n. We need T.
        # We can recompute T from n.
        random_vec = np.random.rand(3)
        if np.abs(np.dot(random_vec, n)) > 0.9:
            random_vec = np.array([1, 0, 0])
        t1 = np.cross(n, random_vec)
        t1 = t1 / np.linalg.norm(t1)
        t2 = np.cross(n, t1)
        
        # Construct rotation matrix R where columns are t1, t2, n
        R = np.column_stack((t1, t2, n))
        
        # Embed Lambda into 3x3
        sigma_p = gaussian.Lambda[0,0]**0.5 # Assuming isotropic for simplicity or use diag
        # Source: Lambda = diag(sigma_p^2, sigma_p^2)
        Lambda_3d = np.zeros((3,3))
        Lambda_3d[:2, :2] = gaussian.Lambda
        
        sigma_n = 1e-4
        Sigma_new = R @ Lambda_3d @ R.T + 2 * (sigma_n**2) * np.outer(n, n)
        
        gaussian.type = '3D'
        gaussian.Sigma = Sigma_new
        # Remove 2D specific fields
        del gaussian.T
        del gaussian.Lambda
        del gaussian.n
        return True
    return False

def fine_tune_gaussian(gaussian, F, eta):
    """Fine-tune Gaussian position and covariance using TensoSDF."""
    mu = gaussian.mu
    d = F(mu)
    grad = compute_sdf_gradient(mu[0], mu[1], mu[2], ...)
    grad_norm = np.linalg.norm(grad)
    if grad_norm < 1e-8:
        return
    
    g = grad / grad_norm
    
    # Update position
    mu_new = mu - eta * d * g
    gaussian.mu = mu_new
    
    # Update covariance
    # Find principal axis v_max
    eigenvalues, eigenvectors = np.linalg.eigh(gaussian.Sigma)
    idx = np.argmax(eigenvalues)
    v_max = eigenvectors[:, idx]
    
    R = rodrigues_rotation(g, v_max)
    Sigma_new = R @ gaussian.Sigma @ R.T
    gaussian.Sigma = Sigma_new
```

## Operating characteristics

**Time and Memory Complexity:**
*   The method introduces additional storage overhead for TensoSDF tensor factors ($A_x, A_y, A_z$) and potentially more Gaussian primitives due to adaptive conversion [S1].
*   On the Mip-NeRF dataset, average memory consumption increased from 734MB (3DGS) to 845MB (proposed method), an increase of ~15.1% [S1].
*   Increases of 16.1% and 18.2% were observed on DeepBlending and Tanks&Temples datasets respectively [S1].
*   Training requires parallel optimization of Gaussian parameters and TensoSDF factors, adding computational cost per iteration [S1].

**Measured Performance:**
*   On NeRF-synthetic dataset, the method attained an average PSNR improvement of 6.1% over 3DGS [S1].
*   Outperforms 3DGS in global geometric consistency and reconstruction performance for highly reflective materials [S1].

**Strengths and Limitations:**
*   **Strengths:** Reduces redundancy in planar/thin structures; improves geometric consistency via TensoSDF; handles specular reflections better (Specular Module ablation shows significant enhancement) [S1].
*   **Limitations:** Suboptimal performance in highly transparent environments [S1]. Increased memory usage compared to standard 3DGS [S1].

**When to Use:**
*   Scenes with significant planar or thin structures where 3DGS redundancy is high.
*   Applications requiring high geometric accuracy and consistency.
*   Scenes with reflective materials.

**When Not to Use:**
*   Highly transparent environments.
*   Memory-constrained applications where the 15-18% overhead is unacceptable.

## Background

**Origin:**
*   Proposed by Yanshan Zhang, Luyao Wang, and Yuanzhang Fan in 2026 [S1].
*   Published in the 2026 2nd International Conference on Artificial Intelligence and Computational Intelligence (AICI 2026) [S1].

**History and Development:**
*   Builds upon 3D Gaussian Splatting (3DGS) and 2D Gaussian Splatting (2DGS) [S1].
*   Integrates TensoSDF, a tensor-based Signed Distance Function representation, to provide geometric constraints [S1].
*   Addresses limitations of 3DGS in balancing planar and volumetric structures and achieving global geometric consistency [S1].

**Usage:**
*   High-precision 3D Gaussian Splatting rendering.
*   Real-time 3D reconstruction and rendering.
*   Virtual reality, augmented reality, digital twins, and film special effects [S1].

## Sources

[S1] Hybrid Gaussian Representation with Geometry-Guided Constraints for High-Precision 3D Gaussian Splatting Rendering--Hybrid Gaussian Splatting with Geometry Guidance--Adaptive Hybrid Representation and TensoSDF Constraints for High-Accuracy Real-Time 3D Reconstruction and Rendering, https://doi.org/10.1145/3810280.3810302, fetched 2026-10-29
