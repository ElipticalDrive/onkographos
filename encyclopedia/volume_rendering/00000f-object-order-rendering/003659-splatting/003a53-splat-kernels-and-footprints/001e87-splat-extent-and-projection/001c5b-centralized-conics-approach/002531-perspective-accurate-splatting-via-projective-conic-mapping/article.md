# Perspective Accurate Splatting via Projective Conic Mapping

Also called: Affine Approximation of Projective Conics, Homogeneous Coordinate Splatting, Perspective Accurate Splatting, Perspective Correct Splatting, Projective Conic Splatting, SPH Perspective Rendering Fix

## Overview
This method computes perspective-correct splat shapes for point-based rendering by approximating the projective mapping of Gaussian reconstruction kernels from their local tangent planes to image space using affine mappings derived from conic isocontours [S2, unofficial]. Unlike previous methods that used affine approximations exact only at the kernel center, this approach ensures accuracy at the kernel cutoff boundary, preventing artifacts such as holes and incorrect elongation under extreme perspective projections [S2, unofficial]. It solves the problem where previous projections integrated kernels along the z-axis rather than the oblique ray from the camera, causing particles to be too faint by $\cos(\theta)$ and insufficiently elongated radially [S1].

## How it works
**Inputs:**
- A set of point samples $p_k$ with associated color $f_k$ and local tangent frames defined by tangent vectors $t_u, t_v$ [S2, unofficial].
- A Gaussian reconstruction kernel $g$ with a user-specified cutoff value $F_g$ (typically $1 < F_g < 2$) [S2, unofficial].
- Camera projection parameters.

**Outputs:**
- Screen-space rasterization of splats with correct perspective shape and intensity [S2, unofficial].

**Data Structures:**
- **Tangent Plane Matrix ($M_k$):** A $3 \times 3$ matrix mapping local tangent coordinates $(u, v)$ to 3D world space.
  $$ M_k = \begin{bmatrix} t_u \\ t_v \\ p_k \end{bmatrix} $$
  Points $p$ on the plane are expressed as $p = (u, v, 1) M_k$ [S2, unofficial].
- **Conic Matrix ($Q_h$):** A $3 \times 3$ symmetric matrix representing the implicit form of a general conic in homogeneous coordinates:
  $$ Q_h = \begin{bmatrix} A & B & D \\ B & C & E \\ D & E & -F \end{bmatrix} $$
  The conic equation is $[x \ y \ 1] Q_h [x \ y \ 1]^T = 0$ [S2, unofficial].
- **Variance Matrix ($R'_k$):** The covariance matrix of the reconstruction kernel in the tangent plane [S2, unofficial].

**Steps:**

1.  **Define the Tangent Plane and Isocontour:**
    For each point $p_k$, define the tangent plane using $M_k$. The Gaussian kernel is truncated at an isocontour defined by:
    $$ \begin{bmatrix} u & v & 1 \end{bmatrix} \begin{bmatrix} R'_k & 0 \\ 0 & -F_g^2 \end{bmatrix} \begin{bmatrix} u \\ v \\ 1 \end{bmatrix} = 0 $$
    This defines a central conic in the tangent plane [S2, unofficial].

2.  **Compute the Projective Mapping:**
    The mapping from the tangent plane to the image plane is a projective transformation. The method approximates this projective mapping with an affine mapping that is exact at the conic isocontour rather than the center [S2, unofficial].

3.  **Calculate the Mapped Variance Matrix ($Q'''$):**
    The variance matrix of the reconstruction kernel in image space, denoted $Q'''$, is derived from the projective mapping. This involves inverting the matrix $M_k$ [S2, unofficial].
    *Note: The exact formula for $Q'''$ in terms of $M_k$ and $R'_k$ is referenced as Equation 11 in the source, but the explicit algebraic expansion is not provided in the text. The implementation computes $Q'''$ to represent the variance matrix of the reconstruction kernel $R'_k$ after projection [S2, unofficial].*

4.  **Check for Numerical Stability:**
    Invert $M_k$ to perform the mapping. Check the condition number of $M_k$. If the condition number exceeds a threshold, discard the splat. This handles cases where the splat degenerates into a line segment [S2, unofficial].

5.  **Validate Conic Type:**
    Determine if the mapped conic defined by $Q'''$ is an ellipse. Calculate the discriminant $\Delta = AC - B^2$ from the upper-left $2 \times 2$ submatrix of $Q'''$.
    - If $\Delta > 0$, it is an ellipse.
    - If $\Delta = 0$, it is a parabola.
    - If $\Delta < 0$, it is a hyperbola.
    Discard the splat if the conic is not an ellipse (i.e., if $\Delta \le 0$) [S2, unofficial].

6.  **Compute Bounding Box and Point Size:**
    Calculate the bounding box of the elliptical splat in screen space using the method described in the source's Section 4.2 (not detailed in the provided text). Use this to determine the OpenGL point size [S2, unofficial].

7.  **Rasterization and Evaluation:**
    Render the splat as an OpenGL point. In the fragment program, for each pixel $x$ covered by the point, evaluate the ellipse equation:
    $$ r^2 = x (Q''' + H)^{-1} x^T $$
    where $H$ is the resampling filter variance matrix [S2, unofficial]. If the pixel is inside the ellipse, accumulate the color weighted by the Gaussian value.

8.  **SPH Specific Adjustments (if applicable):**
    - Evaluate the kernel at the impact parameter (perpendicular to the ray from the camera) rather than the offset in the particle plane [S1].
    - Widen the pixel range accordingly [S1].
    - For slice-based rendering, place pixels at the depth of the slicing plane $z=z_0$ as seen from the camera, not the particle depth. Empty slices if the plane is behind the camera [S1].
    - Compute perspective geometry in double precision to improve accuracy [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def perspective_accurate_splatting(points, camera_matrix, image_plane_matrix, Fg=1.5):
    """
    Renders points using perspective accurate splatting via projective conic mapping.
    
    Args:
        points: List of dicts with 'p' (3D pos), 'color', 'tu' (tangent u), 'tv' (tangent v), 'Rk' (2x2 variance).
        camera_matrix: 4x4 projection matrix.
        image_plane_matrix: 3x3 matrix defining the image plane basis.
        Fg: Cutoff value for Gaussian truncation.
        
    Returns:
        Accumulated color buffer and weight buffer.
    """
    # Initialize buffers
    color_buffer = np.zeros((height, width, 3))
    weight_buffer = np.zeros((height, width))
    
    for pt in points:
        pk = np.array(pt['p'])
        tu = np.array(pt['tu'])
        tv = np.array(pt['tv'])
        Rk = np.array(pt['Rk']) # 2x2 matrix
        
        # 1. Construct Tangent Plane Matrix Mk
        # Mk maps (u, v, 1) from tangent space to 3D world space
        Mk = np.vstack([tu, tv, pk]) # 3x3 matrix
        
        # 2. Check Condition Number of Mk
        cond_Mk = np.linalg.cond(Mk)
        if cond_Mk > 1e6: # Threshold [G]
            continue # Discard splat if ill-conditioned
            
        # 3. Invert Mk
        Mk_inv = np.linalg.inv(Mk)
        
        # 4. Compute Mapped Variance Matrix Q'''
        # The source states Q''' is derived from the projective mapping.
        # The exact formula for Q''' from Mk and Rk is not explicitly expanded in the text.
        # We assume a function compute_mapped_variance exists based on the projective transform.
        # Based on Eq 11 reference, Q''' relates to the variance in image space.
        # Note: The source mentions using Q''' for the variance matrix of the reconstruction kernel R'k.
        # We approximate the transformation of the conic isocontour.
        
        # Construct the 3x3 conic matrix for the isocontour in tangent space
        # Isocontour: [u v 1] [Rk 0; 0 -Fg^2] [u v 1]^T = 0
        Q_tangent = np.zeros((3, 3))
        Q_tangent[:2, :2] = Rk
        Q_tangent[2, 2] = -Fg**2
        
        # Projective mapping from tangent plane to image plane coordinates
        # The source implies an affine approximation of the projective mapping.
        # The mapping x = u * Mk transforms tangent coords to 3D.
        # Then camera projects 3D to 2D.
        # The source uses homogeneous coordinates for the conic.
        
        # Let's assume the projective transform from tangent (u,v) to image (x,y) is approximated.
        # The source says: "we use it as our affine approximation for the projective mapping x = uMk"
        # This part is ambiguous in the text. However, the key is computing Q''' which represents
        # the variance matrix in image space.
        
        # Standard EWA splatting computes the 2x2 covariance matrix in screen space.
        # Here, we use the conic approach.
        # The mapped conic matrix Q''' is computed.
        # We will use a placeholder for the specific algebraic derivation of Q''' from Mk and Q_tangent
        # as the exact formula is not in the provided text snippets.
        # However, the source says: "In the vertex program we compute the variance matrix R'k + H...
        # Here, we use our novel approximation Q''' ... for the variance matrix of the reconstruction kernel R'k."
        
        # Let's assume Q''' is the 2x2 upper-left block of the projected conic, scaled appropriately.
        # Or more likely, Q''' is the 3x3 matrix representing the conic in image space.
        
        # To implement from text alone, we must rely on the structure:
        # 1. Map the conic from tangent space to image space.
        # 2. Extract the ellipse parameters.
        
        # Since the exact formula for Q''' is missing, we note this gap.
        # However, the source mentions: "Q''' represents a general conic... determine if the conic is an ellipse".
        
        # Let's assume the standard projective transformation of a conic:
        # If C is the conic in tangent space, and T is the transformation matrix to image space,
        # then C' = T^{-T} C T^{-1}.
        # The transformation from tangent (u,v) to image (x,y) is not linear (projective).
        # The method approximates this with an affine mapping.
        
        # The source states: "we choose the affine mapping such that it matches the projective mapping of a conic isocontour".
        # This implies we solve for an affine matrix A such that the ellipse defined by Rk and Fg maps correctly.
        
        # Given the lack of explicit formula for Q''' in the text, we cannot write the exact matrix math.
        # We will proceed with the validation steps which are explicit.
        
        # Placeholder for Q''' computation [G]
        # In practice, this involves projecting the tangent plane to the image plane and computing the 
        # Jacobian of the projection at the isocontour.
        Q_prime_prime_prime = compute_Q_ppp(Mk, Rk, Fg, camera_matrix) 
        
        # 5. Check if Conic is an Ellipse
        # Q''' is a 3x3 matrix [A B D; B C E; D E -F]
        A = Q_prime_prime_prime[0, 0]
        B = Q_prime_prime_prime[0, 1]
        C = Q_prime_prime_prime[1, 1]
        
        discriminant = A * C - B**2
        
        if discriminant <= 0:
            continue # Discard if not an ellipse
            
        # 6. Compute Bounding Box
        # Use method from Section 4.2 [G]
        bbox = compute_bounding_box(Q_prime_prime_prime)
        
        # 7. Rasterize
        # Render as OpenGL point with size determined by bbox
        # Fragment shader evaluates: r^2 = x (Q''' + H)^-1 x^T
        # Accumulate color and weight
        
        # For each pixel in bbox:
        for y in range(bbox.min_y, bbox.max_y):
            for x in range(bbox.min_x, bbox.max_x):
                # Pixel coordinate relative to center
                x_vec = np.array([x - bbox.center_x, y - bbox.center_y, 1])
                
                # Compute resampling filter variance H [G]
                H = compute_filter_variance(x_vec)
                
                # Total variance matrix
                Total_Var = Q_prime_prime_prime + H
                
                # Invert Total_Var
                try:
                    Total_Var_inv = np.linalg.inv(Total_Var)
                except np.linalg.LinAlgError:
                    continue
                    
                # Evaluate ellipse equation
                # Note: The source says x (Q''' + H)^-1 x^T. 
                # This yields a scalar value related to the squared distance in the metric defined by the inverse variance.
                val = x_vec @ Total_Var_inv @ x_vec.T
                
                # If val < 1 (or some threshold), the pixel is inside the splat
                # The actual Gaussian value needs to be computed.
                # The source says: "evaluating the ellipse equation... If the pixel is inside..."
                # The weight is typically exp(-val/2) or similar, depending on normalization.
                # The source mentions "accumulated filter weights" and "normalization".
                
                # Compute Gaussian weight
                # The source uses g_{Q'''^m-1}. The exact normalization factor is (1/|Q'''|^(1/2)).
                # We approximate the weight accumulation.
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

def compute_Q_ppp(Mk, Rk, Fg, camera_matrix):
    """
    Placeholder for the computation of Q''' as described in Equation 11.
    The exact algebraic derivation is not provided in the source text.
    This function would implement the affine approximation of the projective mapping
    that is exact at the conic isocontour.
    """
    # Implementation detail missing from sources [G]
    # Returns a 3x3 matrix Q'''
    return np.eye(3)

def compute_filter_variance(x_vec):
    """
    Placeholder for computing the resampling filter variance H.
    """
    return np.zeros((3, 3))

def compute_bounding_box(Q):
    """
    Placeholder for computing the bounding box from the conic matrix.
    """
    return type('Box', (), {'min_x': 0, 'max_x': 1, 'min_y': 0, 'max_y': 1, 'center_x': 0, 'center_y': 0})()
```

## Operating characteristics
- **Time Complexity:** Dominated by the per-pixel evaluation in the fragment shader for pixels covered by the splat's bounding box. The vertex shader performs matrix inversions and condition number checks, which are constant time per splat.
- **Memory Complexity:** Low, as it operates on a per-splat basis in the vertex shader and per-pixel in the fragment shader. No large auxiliary structures are required beyond the standard framebuffers.
- **Performance:** Implemented in three passes: depth image, splat accumulation, and normalization [S2, unofficial]. Uses modern GPU vertex and fragment programs [S2, unofficial].
- **Strengths:** Eliminates holes and artifacts caused by affine approximations in extreme perspective projections [S2, unofficial]. Provides high image quality with anisotropic texture filtering (EWA resampling) [S2, unofficial].
- **Limitations:** Computationally more expensive than simple affine splatting due to the conic mapping and validation steps. Numerical instability can occur if the splat degenerates, requiring discarding [S2, unofficial].
- **When to use:** When perspective accuracy is critical, especially for extreme viewing angles or close-up views where affine approximations fail [S2, unofficial].
- **When not to use:** For orthographic projections where affine approximations are sufficient, or in performance-critical applications where the overhead of conic mapping is unacceptable.

## Background
The method was introduced by Jussi Rääsen, Mario Botsch, Carsten Dachsbacher, and Mark Pauly [S2, unofficial]. It addresses limitations in previous point rendering techniques by Heckbert and Zwicker, which used affine approximations of the perspective projection that were exact only at the kernel center [S2, unofficial]. The source is an unofficial record from EPFL [S2, unofficial]. The SPH perspective fix described in [S1] is a related implementation detail in the pynbody library, correcting integration along the oblique ray.

## Sources
[S1] Opt-in exact AMR rendering, new pynbody.imaging package, and SPH perspective fixes, https://github.com/pynbody/pynbody/pull/1048, fetched 2026-10-26
[S2] Perspective Accurate Splatting, http://infoscience.epfl.ch/record/149345, fetched 2026-10-26 (unofficial)
