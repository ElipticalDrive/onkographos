# Perspective Accurate Splatting via Projective Conic Mapping

Also called: Affine Approximation of Projective Conics, Homogeneous Coordinate Splatting, Perspective Accurate Splatting, Perspective Correct Splatting, Projective Conic Splatting, SPH Perspective Rendering Fix

## Overview
This method computes perspective-correct splat shapes for point-based rendering by approximating the projective mapping of Gaussian reconstruction kernels from their local tangent planes to image space using affine mappings derived from conic isocontours [S2, unofficial]. Unlike previous methods that used affine approximations exact only at the kernel center, this approach ensures accuracy at the kernel cutoff boundary, preventing artifacts such as holes and incorrect elongation under extreme perspective projections [S2, unofficial]. It solves the problem where previous projections integrated kernels along the z-axis rather than the oblique ray from the camera, causing particles to be too faint by $\cos(\theta)$ and insufficiently elongated radially [S1].

## How it works
**Inputs:**
- A set of point samples $\mathbf{p}_k$ with associated color $f_k$ and local tangent frames defined by tangent vectors $\mathbf{t}_u, \mathbf{t}_v$ [S2, unofficial].
- A Gaussian reconstruction kernel with a user-specified cutoff value $F_g$ (typically $1 < F_g < 2$) [S2, unofficial].
- Camera projection parameters.

**Outputs:**
- Screen-space rasterization of splats with correct perspective shape and intensity [S2, unofficial].

**Data Structures:**
- **Tangent Plane Matrix ($M_k$):** A $3 \times 3$ matrix mapping local tangent coordinates $(u, v)$ to 3D world space, formed by rows $\mathbf{t}_u$, $\mathbf{t}_v$, and $\mathbf{p}_k$ [S2, unofficial]. Points $\mathbf{p}$ on the plane are expressed as $\mathbf{p} = (u, v, 1) M_k$ [S2, unofficial].
- **Conic Matrix ($Q_h$):** A $3 \times 3$ symmetric matrix representing the implicit form of a general conic in homogeneous coordinates: $Q_h = \begin{bmatrix} A & B & D \\ B & C & E \\ D & E & -F \end{bmatrix}$ [S2, unofficial]. The conic equation is $[x \ y \ 1] Q_h [x \ y \ 1]^T = 0$ [S2, unofficial].
- **Variance Matrix ($R'_k$):** The covariance matrix of the reconstruction kernel in the tangent plane [S2, unofficial].

**Steps:**

1.  **Define Tangent Plane and Isocontour:**
    For each point $\mathbf{p}_k$, construct $M_k$ from $\mathbf{t}_u$, $\mathbf{t}_v$, and $\mathbf{p}_k$ [S2, unofficial]. The Gaussian kernel is truncated at an isocontour defined by:
    $$ \begin{bmatrix} u & v & 1 \end{bmatrix} \begin{bmatrix} R'_k & 0 \\ 0 & -F_g^2 \end{bmatrix} \begin{bmatrix} u \\ v \\ 1 \end{bmatrix} = 0 $$
    This defines a central conic in the tangent plane [S2, unofficial].

2.  **Compute Projective Mapping:**
    The mapping from the tangent plane to the image plane is a projective transformation [S2, unofficial]. The method approximates this projective mapping with an affine mapping that is exact at the conic isocontour rather than the kernel center [S2, unofficial].

3.  **Calculate Mapped Variance Matrix ($Q'''$):**
    The variance matrix of the reconstruction kernel in image space, denoted $Q'''$, is derived from the projective mapping of the tangent-space conic [S2, unofficial]. It requires inverting $M_k$ and transforming the conic to screen space [S2, unofficial].

4.  **Check Numerical Stability:**
    Invert $M_k$ to perform the mapping. Check the condition number of $M_k$. If it exceeds a threshold, discard the splat to handle numerical ill-conditioning when the splat degenerates into a line segment [S2, unofficial].

5.  **Validate Conic Type:**
    Determine if the mapped conic defined by $Q'''$ is an ellipse [S2, unofficial]. Calculate the discriminant $\Delta = AC - B^2$ from the upper-left $2 \times 2$ submatrix of $Q'''$ [S2, unofficial].
    - If $\Delta > 0$, it is an ellipse.
    - If $\Delta = 0$, it is a parabola.
    - If $\Delta < 0$, it is a hyperbola.
    Discard the splat if the conic is not an ellipse (i.e., if $\Delta \le 0$) [S2, unofficial].

6.  **Compute Bounding Box and Point Size:**
    Calculate the bounding box of the elliptical splat in screen space to determine the OpenGL point size [S2, unofficial].

7.  **Rasterization and Evaluation:**
    Render the splat as an OpenGL point. In the fragment program, for each pixel $\mathbf{x}$ covered by the point, evaluate the ellipse equation:
    $$ r^2 = \mathbf{x} (Q''' + H)^{-1} \mathbf{x}^T $$
    where $H$ is the resampling filter variance matrix [S2, unofficial]. If the pixel is inside the ellipse, accumulate the color weighted by the Gaussian value [S2, unofficial].

8.  **SPH-Specific Adjustments:**
    - Evaluate the kernel at the impact parameter (perpendicular to the camera ray) rather than the offset in the particle plane, and widen the pixel range accordingly [S1].
    - For slice-based rendering, place pixels at the depth of the slicing plane $z=z_0$ as seen from the camera, not the particle depth; discard slices if the plane is behind the camera [S1].
    - Compute perspective geometry in double precision to improve accuracy [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def perspective_accurate_splatting(points, camera_proj, image_dims, Fg=1.5):
    """
    Renders points using perspective accurate splatting via projective conic mapping.
    
    Args:
        points: List of dicts with 'p' (3D pos), 'color', 'tu' (tangent u), 'tv' (tangent v), 'Rk' (2x2 variance).
        camera_proj: 4x4 projection matrix.
        image_dims: Tuple (height, width).
        Fg: Cutoff value for Gaussian truncation.
        
    Returns:
        Accumulated color buffer and weight buffer.
    """
    height, width = image_dims
    color_buffer = np.zeros((height, width, 3))
    weight_buffer = np.zeros((height, width))
    
    for pt in points:
        pk = np.array(pt['p'])
        tu = np.array(pt['tu'])
        tv = np.array(pt['tv'])
        Rk = np.array(pt['Rk']) # 2x2 matrix
        
        # 1. Construct Tangent Plane Matrix Mk
        Mk = np.vstack([tu, tv, pk]) # 3x3 matrix
        
        # 2. Check Condition Number of Mk
        cond_Mk = np.linalg.cond(Mk)
        if cond_Mk > 1e6: # Threshold [G]
            continue # Discard splat if ill-conditioned
            
        # 3. Invert Mk
        Mk_inv = np.linalg.inv(Mk)
        
        # 4. Compute Mapped Variance Matrix Q'''
        # Construct 3x3 conic matrix for isocontour in tangent space
        Q_tangent = np.zeros((3, 3))
        Q_tangent[:2, :2] = Rk
        Q_tangent[2, 2] = -Fg**2
        
        # Compute projective mapping matrix from tangent plane to screen space
        # Approximate using Jacobian of perspective projection at pk [G]
        pk_hom = np.append(pk, 1.0)
        pk_proj = camera_proj @ pk_hom
        if pk_proj[2] <= 0:
            continue # Behind camera
            
        # Transform tangent vectors as directions (ignore translation)
        tu_proj = camera_proj[:3, :3] @ tu
        tv_proj = camera_proj[:3, :3] @ tv
        
        # Perspective divide to get screen-space tangent vectors
        scale_x = width / 2.0
        scale_y = height / 2.0
        J = np.array([
            (tu_proj[:2] / tu_proj[2] - pk_proj[:2] / pk_proj[2]) * scale_x,
            (tv_proj[:2] / tv_proj[2] - pk_proj[:2] / pk_proj[2]) * scale_y
        ])
        
        # Transform covariance to screen space
        Q_screen_2x2 = J @ Rk @ J.T
        Q_prime_prime_prime = np.zeros((3, 3))
        Q_prime_prime_prime[:2, :2] = Q_screen_2x2
        Q_prime_prime_prime[2, 2] = -Fg**2
        
        # 5. Check if Conic is an Ellipse
        A = Q_prime_prime_prime[0, 0]
        B = Q_prime_prime_prime[0, 1]
        C = Q_prime_prime_prime[1, 1]
        discriminant = A * C - B**2
        if discriminant <= 0:
            continue # Discard if not an ellipse
            
        # 6. Compute Bounding Box
        eigenvalues = np.linalg.eigvalsh(Q_screen_2x2)
        major_axis = Fg * np.sqrt(1.0 / eigenvalues.min())
        bbox_size = int(np.ceil(major_axis))
        center_x, center_y = pk_proj[:2] * scale_x, pk_proj[:2] * scale_y
        bbox = (
            max(0, int(center_x) - bbox_size),
            min(width, int(center_x) + bbox_size),
            max(0, int(center_y) - bbox_size),
            min(height, int(center_y) + bbox_size)
        )
        
        # 7. Rasterize
        H = np.eye(3) * 0.25 # Resampling filter variance [G]
        Total_Var = Q_prime_prime_prime + H
        try:
            Total_Var_inv = np.linalg.inv(Total_Var)
        except np.linalg.LinAlgError:
            continue
            
        for y in range(bbox[2], bbox[3]):
            for x in range(bbox[0], bbox[1]):
                x_vec = np.array([x - center_x, y - center_y, 1.0])
                val = x_vec @ Total_Var_inv @ x_vec.T
                if val < 1.0:
                    weight = np.exp(-0.5 * val)
                    color_buffer[y, x] += pt['color'] * weight
                    weight_buffer[y, x] += weight

    # Normalize
    for y in range(height):
        for x in range(width):
            w = weight_buffer[y, x]
            if w > 0:
                color_buffer[y, x] /= w
                
    return color_buffer
```

## Operating characteristics
- **Time Complexity:** Dominated by the per-pixel evaluation in the fragment shader for pixels covered by the splat's bounding box. The vertex shader performs matrix inversions and condition number checks, which are constant time per splat [S2, unofficial].
- **Memory Complexity:** Low, as it operates on a per-splat basis in the vertex shader and per-pixel in the fragment shader. No large auxiliary structures are required beyond the standard framebuffers [S2, unofficial].
- **Performance:** Implemented in three passes: depth image, splat accumulation, and normalization [S2, unofficial]. Uses modern GPU vertex and fragment programs [S2, unofficial].
- **Strengths:** Eliminates holes and artifacts caused by affine approximations in extreme perspective projections [S2, unofficial]. Provides high image quality with anisotropic texture filtering (EWA resampling) [S2, unofficial].
- **Limitations:** Computationally more expensive than simple affine splatting due to the conic mapping and validation steps [S2, unofficial]. Numerical instability can occur if the splat degenerates, requiring discarding [S2, unofficial].
- **When to use:** When perspective accuracy is critical, especially for extreme viewing angles or close-up views where affine approximations fail [S2, unofficial].
- **When not to use:** For orthographic projections where affine approximations are sufficient, or in performance-critical applications where the overhead of conic mapping is unacceptable [G].

## Background
The method was introduced by Jussi Rääsen, Mario Botsch, Carsten Dachsbacher, and Mark Pauly [S2, unofficial]. It addresses limitations in previous point rendering techniques by Heckbert and Zwicker, which used affine approximations of the perspective projection that were exact only at the kernel center [S2, unofficial]. The source is an unofficial record hosted at EPFL, though the authors are affiliated with MIT, Helsinki University, TU Aachen, and FAU Erlangen-Nürnberg [S2, unofficial]. The SPH perspective fix described in [S1] is a related implementation detail in the pynbody library, correcting integration along the oblique ray [S1].

## Sources
[S1] Opt-in exact AMR rendering, new pynbody.imaging package, and SPH perspective fixes, https://github.com/pynbody/pynbody/pull/1048, fetched 2026-10-26
[S2] Perspective Accurate Splatting, http://infoscience.epfl.ch/record/149345, fetched 2026-10-26 (unofficial)
