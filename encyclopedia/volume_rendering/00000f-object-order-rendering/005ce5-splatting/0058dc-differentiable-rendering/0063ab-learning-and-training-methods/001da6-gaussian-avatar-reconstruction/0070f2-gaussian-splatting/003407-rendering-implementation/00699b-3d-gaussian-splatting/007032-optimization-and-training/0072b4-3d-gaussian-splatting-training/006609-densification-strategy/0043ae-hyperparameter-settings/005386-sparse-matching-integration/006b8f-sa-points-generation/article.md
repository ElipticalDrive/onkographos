# SA-Points Generation

Also called: SA-Points, Self-Augmented Points generation, Self-augmented Points

## Overview
SA-Points Generation is a method for creating geometry-consistent 3D point clouds from a single reference view by triangulating dense correspondences between that view and a rendered extrapolated view. It solves the problem of estimating surface-aware scene coverage in sparse, wide-baseline settings where traditional multi-view triangulation is unavailable [S1]. Unlike sibling methods that rely on monocular depth estimates or require multiple input views, SA-Points leverages the generalization capability of foundational 3D models (MASt3R) and the geometric consistency of a current 3D Gaussian Splatting (3DGS) model to produce scale-consistent, surface-aware geometry [S1].

## How it works
**Inputs:**
- Reference image $I_r$ and its camera pose $T_r = [R_r | t_r]$ [S1].
- Perturbation vector $\Delta t$ for the extrapolated pose [S1].
- Pretrained MASt3R model for dense correspondence prediction [S1].
- Camera intrinsics (typically from COLMAP) [S1].
- Current 3DGS model parameters [S1].
- Reprojection error threshold $\tau$ [S1].

**Outputs:**
- A set of 3D points $\{X_i\}$ representing the Self-Augmented Points (SA-Points) [S1].

**Data Structures:**
- 2D correspondences: A set of pairs $\{(p_r^i, p_e^i)\}$, where $p_r^i$ is a 2D point in $I_r$ and $p_e^i$ is its corresponding 2D point in the extrapolated image $I_e$ [S1].
- Projection matrices: $P_r$ and $P_e$ derived from poses $T_r$ and $T_e$ respectively, combined with camera intrinsics [S1].
- 3D Points: $X_i$ in homogeneous coordinates [S1].

**Steps:**
1.  **Render Extrapolated View:** Using the current 3DGS model, render an extrapolated image $I_e$ from a perturbed camera pose $T_e = [R_r | t_r + \Delta t]$. The rotation $R_r$ is kept identical to the reference pose, while the translation is perturbed by $\Delta t$ [S1].
2.  **Predict Dense Correspondences:** Use the pretrained MASt3R model to predict dense 2D correspondences $\{(p_r^i, p_e^i)\}$ between the reference image $I_r$ and the rendered extrapolated image $I_e$ [S1]. MASt3R is chosen for its robustness to moderate viewpoint changes and ability to produce contextually meaningful matches despite minor geometric distortions in the 3DGS rendering [S1].
3.  **Triangulate SA-Points:** For each correspondence pair $(p_r^i, p_e^i)$, triangulate the 3D point $X_i$ using the projection matrices $P_r$ and $P_e$. These matrices are derived from the camera poses $T_r$ and $T_e$ and the camera intrinsics (e.g., from COLMAP) [S1].
4.  **Filter by Reprojection Error:** Calculate the reprojection error $\varepsilon^i$ for each triangulated point $X_i$. Retain points where $\varepsilon^i < \tau$; discard points where $\varepsilon^i \ge \tau$ [S1].

**Formulas:**
The reprojection error $\varepsilon^i$ is defined as [S1]:

$$
\varepsilon^i = \frac{1}{2} \left( \| p_r^i - \pi(P_r X_i) \|_2 + \| p_e^i - \pi(P_e X_i) \|_2 \right)
$$

Where:
- $\| \cdot \|_2$ is the $L_2$ norm in pixel space [S1].
- $\pi(\cdot)$ is the projection function mapping 3D points to 2D image coordinates [S1].
- $P_r, P_e$ are the projection matrices for the reference and extrapolated views [S1].
- $X_i$ is the triangulated 3D point [S1].
- $p_r^i, p_e^i$ are the original 2D correspondences [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_sa_points(reference_image, reference_pose, perturbation_delta_t, 
                       current_3dgs_model, mast3r_model, camera_intrinsics, 
                       reprojection_threshold):
    """
    Generates SA-Points from a single reference view using an extrapolated view.
    
    Args:
        reference_image (Tensor): The input reference image I_r.
        reference_pose (Matrix): Camera pose T_r = [R_r | t_r].
        perturbation_delta_t (Vector): Translation perturbation delta_t.
        current_3dgs_model (Model): The current 3D Gaussian Splatting model.
        mast3r_model (Model): Pretrained MASt3R model for correspondence.
        camera_intrinsics (Matrix): Camera intrinsic matrix K.
        reprojection_threshold (float): Threshold tau for filtering.
        
    Returns:
        List[Vector]: List of valid 3D SA-Points X_i.
    """
    # Step 1: Construct extrapolated pose and render
    # T_e = [R_r | t_r + delta_t]
    R_r = reference_pose[:3, :3]
    t_r = reference_pose[:3, 3]
    t_e = t_r + perturbation_delta_t
    extrapolated_pose = Matrix(4, 4)
    extrapolated_pose[:3, :3] = R_r
    extrapolated_pose[:3, 3] = t_e
    extrapolated_pose[3, 3] = 1
    
    # Render extrapolated image I_e using 3DGS
    extrapolated_image = render_3dgs(current_3dgs_model, extrapolated_pose, camera_intrinsics)
    
    # Step 2: Predict dense correspondences using MASt3R
    # Returns pairs (p_r, p_e)
    correspondences = mast3r_model.predict_correspondences(reference_image, extrapolated_image)
    
    # Construct projection matrices
    # P = K * [R | t]
    P_r = camera_intrinsics @ reference_pose[:3, :]
    P_e = camera_intrinsics @ extrapolated_pose[:3, :]
    
    sa_points = []
    
    for p_r, p_e in correspondences:
        # Step 3: Triangulate 3D point X_i
        # Using linear triangulation from two views
        X_i = triangulate_3d_point(p_r, p_e, P_r, P_e)
        
        # Step 4: Calculate reprojection error
        # Project X_i back to both views
        p_r_reprojected = project_to_2d(X_i, P_r)
        p_e_reprojected = project_to_2d(X_i, P_e)
        
        # Calculate L2 norm in pixel space
        error_r = l2_norm(p_r - p_r_reprojected)
        error_e = l2_norm(p_e - p_e_reprojected)
        
        epsilon_i = 0.5 * (error_r + error_e)
        
        # Filter based on threshold
        if epsilon_i < reprojection_threshold:
            sa_points.append(X_i)
            
    return sa_points

def triangulate_3d_point(p_r, p_e, P_r, P_e):
    """
    Linear triangulation of a 3D point from two 2D correspondences.
    Solves for X in P_r * X = p_r and P_e * X = p_e using least squares.
    """
    # Convert 2D points to homogeneous coordinates
    p_r_h = np.array([p_r[0], p_r[1], 1])
    p_e_h = np.array([p_e[0], p_e[1], 1])
    
    # Build linear system Ax = 0
    # For each view, we have cross product equation: p x P*X = 0
    # This yields 2 independent equations per view
    A = np.zeros((4, 4))
    
    # Equations from first view
    A[0] = p_r_h[0] * P_r[2, :] - P_r[0, :]
    A[1] = p_r_h[1] * P_r[2, :] - P_r[1, :]
    
    # Equations from second view
    A[2] = p_e_h[0] * P_e[2, :] - P_e[0, :]
    A[3] = p_e_h[1] * P_e[2, :] - P_e[1, :]
    
    # Solve for X using SVD
    U, S, Vt = np.linalg.svd(A)
    X_h = Vt[-1, :]
    
    # Convert to Euclidean coordinates
    X = X_h[:3] / X_h[3]
    return X

def project_to_2d(X, P):
    """
    Projects 3D point X to 2D using projection matrix P.
    """
    X_h = np.append(X, 1)
    p_h = P @ X_h
    p = p_h[:2] / p_h[2]
    return p

def l2_norm(a, b):
    return np.linalg.norm(a - b)
```

**Parameters:**
- $\Delta t$: Translation perturbation magnitude. Not given in the sources.
- $\tau$: Reprojection error threshold. Not given in the sources.
- MASt3R: Pretrained model for dense correspondence. Default in [S1].

**Edge Cases and Pitfalls:**
- **Rendering Noise:** Since triangulation occurs during training, the 3DGS model may be fitting to sparse/incomplete geometry, causing rendering noise in $I_e$. The reprojection error filter is critical to discard geometrically inconsistent points arising from this noise [S1].
- **Scale Consistency:** The method produces scale-consistent geometry from a single image, avoiding the scale ambiguity often associated with monocular depth estimates [S1].

## Operating characteristics
- **Time Complexity:** Dominated by rendering the extrapolated view and running the MASt3R correspondence model. Not given in the sources.
- **Memory Complexity:** Requires storing the 3DGS model, the rendered extrapolated image, and the resulting point cloud. Not given in the sources.
- **Strengths:** Produces scale-consistent, surface-aware geometry from a single image without requiring multi-view input or monocular depth estimates [S1]. Robust to moderate viewpoint changes and minor geometric distortions [S1].
- **Limitations:** Relies on the quality of the current 3DGS model for rendering $I_e$ and the accuracy of MASt3R for correspondences. Performance may degrade if the perturbation $\Delta t$ is too large or too small. Not given in the sources.
- **When to use:** In sparse, wide-baseline settings for active scene reconstruction where surface-aware coverage estimation is needed [S1].
- **When not to use:** When high-quality multi-view input is available, as other methods may be more direct. Not given in the sources.

## Background
SA-Points Generation was introduced in the SA-ResGS (Self-augmented Residual 3D Gaussian Splatting) framework [S1]. It is designed to stabilize uncertainty quantification and enhance uncertainty-aware supervision for Next-Best-View selection in active scene reconstruction [S1]. The method addresses the lack of learning signals in underrepresented regions within sparse, wide-baseline settings [S1]. It builds on the concept of physically grounded view selection using surface coverage [S1].

## Sources
[S1] SA-ResGS: Self-augmented Residual 3D Gaussian Splatting for Next Best View Selection, https://doi.org/10.1007/978-3-032-37013-6_17, fetched 2026-10-29
