# PRIMU

Also called: Primitive-based Representations of Uncertainty

## Overview
PRIMU is a post-hoc uncertainty estimation (UE) framework for 3D Gaussian Splatting (GS) that constructs primitive-level representations of error and visibility/coverage from training views to estimate uncertainties for novel views [S1, unofficial]. Unlike methods that estimate per-primitive uncertainty during training or rely on parameter variances, PRIMU operates entirely as post-processing without requiring GS retraining [S1, unofficial]. It extracts 13 optionally direction-dependent features from training-view coverage and reconstruction error, rendering them as uncertainty feature maps which are aggregated via pixel-wise regression on holdout data [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A trained 3D Gaussian Splatting model (parameters: positions, covariances, opacities, spherical harmonic coefficients); a set of training views with known ground truth; a set of holdout views with known ground truth (for regression training); a set of novel views (for inference).
*   **Outputs:** Per-pixel uncertainty maps (color and depth) for novel views; optionally, active view selection recommendations.

**Data Structures**
The method operates on the standard 3D Gaussian primitives. Each primitive $k$ is defined by:
*   Mean position $\mu_k \in \mathbb{R}^3$.
*   Covariance matrix $\Sigma_k \in \mathbb{R}^{3 \times 3}$ (or rotation/scale representation).
*   Opacity scalar $g_k$.
*   Color coefficients for spherical harmonics.

The method computes a feature vector $f_k \in \mathbb{R}^{13}$ for each primitive $k$. These features are derived from training view statistics.

**Numbered Steps**

1.  **Project Training Errors and Coverage to Primitives:**
    For each training view $v \in \Omega$ (where $\Omega$ is the set of training views):
    a. Render the scene using the trained GS model to obtain the rendered color $C_v(\bar{x})$ and depth $D_v(\bar{x})$ for each pixel $\bar{x}$.
    b. Compute the per-pixel reconstruction error $e_{\bar{x}}$ (e.g., $\ell_1$ error between rendered and ground truth).
    c. For each Gaussian primitive $k$, identify the set of pixels $v^{(k)}$ in view $v$ where the primitive's opacity $\alpha_k(\bar{x})$ exceeds a small threshold.
    d. Compute the contribution factor for primitive $k$ at pixel $\bar{x}$: $c_{k, \bar{x}} = \alpha_k(\bar{x}) T_k(\bar{x})$, where $T_k(\bar{x}) = \prod_{j=1}^{k-1} (1 - \alpha_j(\bar{x}))$ is the accumulated transparency.
    e. Aggregate these contributions to form primitive-level representations. The paper defines three specific representations, which can be combined or selected:
        i.  **Field-of-View (FoV) Count:** $F(k) = |\Omega_k|$, where $\Omega_k$ is the set of training views in which primitive $k$ lies within the viewing cone [S1, unofficial].
        ii. **Training-View Coverage:** $V(k) = \max_{v \in \Omega} \left( \text{agg}_{\bar{x} \in v^{(k)}} (\alpha_k(\bar{x}) T_k(\bar{x})) \right)$, where $\text{agg} \in \{\text{max, sum, mean}\}$ [S1, unofficial].
        iii. **Rendering Error:** $E(k) = \text{mean}_{v \in \Omega} \left( \text{agg}_{\bar{x} \in v^{(k)}} (e_{\bar{x}} \cdot \alpha_k(\bar{x}) T_k(\bar{x})) \right)$ [S1, unofficial].
    f. Construct the 13-dimensional feature vector $f_k$ for each primitive. The exact composition of the 13 features is not fully detailed in the provided text, but it includes the above representations and is optionally direction-dependent [S1, unofficial].

2.  **Render Uncertainty Feature Maps:**
    For a novel view $v_{novel}$:
    a. Project all Gaussian primitives onto the image plane of $v_{novel}$ to obtain 2D Gaussians with mean $\bar{\mu}_k$ and covariance $\bar{\Sigma}_k$.
    b. The opacity of primitive $k$ at pixel $\bar{x}$ is $\alpha_k(\bar{x}) = g_k \Phi(\bar{x}; \bar{\mu}_k, \bar{\Sigma}_k)$, where $\Phi$ is the 2D Gaussian density [S1, unofficial].
    c. Render the scene using the standard alpha-blending equation to get the final color $C(\bar{x})$ and depth $D(\bar{x})$:
       $$C(\bar{x}) = \sum_{k=1}^{K} \alpha_k(\bar{x}) T_k(\bar{x}) c_k(\vec{d})$$
       where $c_k(\vec{d})$ is the color of the $k$-th Gaussian dependent on view direction $\vec{d}$, and $T_k(\bar{x}) = \prod_{j=1}^{k-1} (1 - \alpha_j(\bar{x}))$ [S1, unofficial].
    d. Simultaneously, render the primitive-level feature maps. For each feature $f_k$, compute its contribution to each pixel $\bar{x}$ using the same opacity and transparency weights:
       $$U_f(\bar{x}) = \sum_{k=1}^{K} \alpha_k(\bar{x}) T_k(\bar{x}) f_k$$
       This produces an uncertainty feature map $U_f$ for each of the 13 features.

3.  **Aggregate via Pixel-wise Regression:**
    a. Collect the rendered uncertainty feature maps $U_{f_1}, \dots, U_{f_{13}}$ for the novel view.
    b. Train a gradient boosting regression model on holdout data. The input to the model is the vector of feature map values at each pixel $[U_{f_1}(\bar{x}), \dots, U_{f_{13}}(\bar{x})]$. The target is the true error (e.g., $\ell_1$ error between rendered and ground truth) for those pixels in the holdout views.
    c. Apply the trained regression model to the feature maps of the novel view to predict the final uncertainty estimate $U(\bar{x})$ for each pixel [S1, unofficial].

4.  **Active View Selection (Optional):**
    a. Use the uncertainty feature maps directly to guide active view selection (AVS).
    b. Prioritize viewpoints with high uncertainty indicators (e.g., high mean or max values in the feature maps) to select new training views, without requiring the regression model [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from sklearn.ensemble import GradientBoostingRegressor

def compute_primitive_features(primitives, training_views, ground_truths):
    """
    Computes 13-dimensional feature vectors for each Gaussian primitive
    based on training view coverage and error.
    
    primitives: List of Gaussian objects with attributes:
        - mean: (3,)
        - cov: (3,3) or (scale, rot)
        - opacity: scalar
        - sh_coeffs: for color
    training_views: List of view parameters (cameras)
    ground_truths: Dict mapping view_id to (color_img, depth_img)
    
    Returns:
        features: (N_primitives, 13) array
    """
    N = len(primitives)
    features = np.zeros((N, 13))
    
    # Placeholder for the exact 13 features. 
    # Based on the text, these include FoV count, coverage, error, 
    # and potentially direction-dependent variants.
    
    for k, prim in enumerate(primitives):
        # 1. Field-of-View Count
        fov_count = 0
        for v_id, cam in enumerate(training_views):
            # Check if primitive is within viewing cone of cam
            if is_in_fov(prim, cam):
                fov_count += 1
        features[k, 0] = fov_count
        
        # 2. Training-View Coverage (V)
        # Aggregate contribution factors alpha * T over visible pixels
        max_agg_coverage = -np.inf
        for v_id, cam in enumerate(training_views):
            # Render primitive contributions for this view
            # Get pixels where alpha > threshold
            visible_pixels = get_visible_pixels(prim, cam, threshold=1e-4)
            if len(visible_pixels) == 0:
                continue
            
            # Compute contribution factors for these pixels
            # alpha_k(x) = g * Phi(x; mu, Sigma)
            # T_k(x) = product(1 - alpha_j) for j < k
            # Note: In practice, T_k is computed during standard rendering
            contributions = []
            for px in visible_pixels:
                alpha = compute_alpha(prim, px, cam)
                # T_k requires sorting and previous alphas. 
                # Assuming standard GS rendering provides this or we approximate
                # by using the primitive's own alpha as a proxy if T is not stored per-pixel per-primitive
                # The paper uses alpha_k * T_k. 
                # For simplicity in this pseudocode, we assume we can access 
                # the contribution factor c_k = alpha_k * T_k from the rendering pass.
                c_k = get_contribution_factor(prim, px, cam) 
                contributions.append(c_k)
            
            if contributions:
                agg_val = np.max(contributions) # agg in {max, sum, mean}
                if agg_val > max_agg_coverage:
                    max_agg_coverage = agg_val
        
        features[k, 1] = max_agg_coverage if max_agg_coverage > -np.inf else 0
        
        # 3. Rendering Error (E)
        mean_error = 0.0
        error_sum = 0.0
        count = 0
        for v_id, cam in enumerate(training_views):
            visible_pixels = get_visible_pixels(prim, cam, threshold=1e-4)
            if len(visible_pixels) == 0:
                continue
            
            gt_color, gt_depth = ground_truths[v_id]
            rendered_color, rendered_depth = render_view(cam, primitives) # Full scene render
            
            for px in visible_pixels:
                c_k = get_contribution_factor(prim, px, cam)
                # Per-pixel L1 error
                err = np.mean(np.abs(rendered_color[px] - gt_color[px]))
                error_sum += err * c_k
                count += c_k
        
        if count > 0:
            mean_error = error_sum / count
        features[k, 2] = mean_error
        
        # 4-13. Other features (direction-dependent, etc.)
        # The paper mentions 13 features, optionally direction-dependent.
        # Specific definitions for features 3-12 are not provided in the excerpt.
        # [G] Assuming these might include directional coverage, error variance, 
        # or SH coefficient magnitudes. We leave them as zeros for now.
        
    return features

def render_uncertainty_maps(primitives, novel_view):
    """
    Renders the uncertainty feature maps for a novel view.
    
    primitives: List of Gaussian objects
    novel_view: Camera parameters
    
    Returns:
        feature_maps: (13, H, W) array
        color_map: (3, H, W)
        depth_map: (1, H, W)
    """
    H, W = novel_view.resolution
    feature_maps = np.zeros((13, H, W))
    
    # Standard GS rendering for color and depth
    color_map, depth_map = render_gs(primitives, novel_view)
    
    # Get primitive features (precomputed)
    # In a real implementation, features are computed once and stored
    # Here we assume we have access to them
    # features = compute_primitive_features(...) 
    
    # Project primitives to 2D
    projected_prims = project_to_2d(primitives, novel_view)
    
    # Sort front-to-back
    projected_prims.sort(key=lambda p: p.depth)
    
    # Render feature maps using alpha blending
    accumulated_transparency = np.ones((H, W))
    
    for prim in projected_prims:
        k = prim.index
        alpha = prim.alpha # Computed from 2D Gaussian density
        
        # Create alpha mask for this primitive
        alpha_mask = np.zeros((H, W))
        # Fill alpha_mask with alpha values at relevant pixels
        # [G] Efficient rasterization of 2D Gaussian is required here.
        # For pseudocode, we assume a function that returns the alpha field.
        alpha_field = rasterize_2d_gaussian(prim)
        
        # Contribution weight
        contribution = alpha_field * accumulated_transparency
        
        # Add to feature maps
        # features[k] is the 13-dim vector for this primitive
        for f_idx in range(13):
            feature_maps[f_idx] += contribution * prim.features[f_idx]
        
        # Update transparency
        accumulated_transparency *= (1 - alpha_field)
        
    return feature_maps, color_map, depth_map

def train_and_predict_uncertainty(training_feature_maps, training_errors, novel_feature_maps):
    """
    Trains a gradient boosting regressor and predicts uncertainty for a novel view.
    
    training_feature_maps: List of (13, H, W) arrays for holdout views
    training_errors: List of (H, W) arrays of true errors for holdout views
    novel_feature_maps: (13, H, W) array for the novel view
    
    Returns:
        uncertainty_map: (H, W) array
    """
    # Flatten feature maps and errors
    n_views = len(training_feature_maps)
    H, W = training_feature_maps[0].shape
    
    X_train = []
    y_train = []
    
    for i in range(n_views):
        # Flatten (13, H, W) to (H*W, 13)
        f_maps = training_feature_maps[i].transpose(1, 2, 0) # (H, W, 13)
        errors = training_errors[i] # (H, W)
        
        # Reshape to (N_pixels, 13)
        X_train.append(f_maps.reshape(-1, 13))
        y_train.append(errors.flatten())
        
    X_train = np.vstack(X_train)
    y_train = np.concatenate(y_train)
    
    # Train Gradient Boosting Regressor
    model = GradientBoostingRegressor(n_estimators=100, max_depth=5)
    model.fit(X_train, y_train)
    
    # Predict for novel view
    f_maps_novel = novel_feature_maps.transpose(1, 2, 0) # (H, W, 13)
    X_novel = f_maps_novel.reshape(-1, 13)
    
    y_pred = model.predict(X_novel)
    uncertainty_map = y_pred.reshape(H, W)
    
    return uncertainty_map
```

**Parameters**
*   **Number of features:** 13 [S1, unofficial].
*   **Regression model:** Gradient boosting regression [S1, unofficial].
*   **Aggregation functions:** max, sum, or mean for contribution factors [S1, unofficial].
*   **Opacity threshold:** A small threshold to determine visible pixels (value not specified) [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Geometry Misalignment:** Gaussian splats may misalign with scene geometry even when rendered appearance is correct, leading to depth error between true geometry and splat distribution [S1, unofficial]. PRIMU aims to capture this via error-based features.
*   **Undefined Depth Error:** Undefined depth error is represented as light gray in visualizations, implying handling of missing or invalid depth values is necessary [S1, unofficial].
*   **Generalization:** The regression model shows generalization capabilities to unseen scenes, but performance may vary [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method adds a post-processing step. Computing primitive features requires iterating over training views and primitives, which is $O(N_{views} \cdot N_{primitives} \cdot N_{pixels})$ in a naive implementation, but optimized via rasterization. Rendering feature maps is $O(N_{primitives} \cdot N_{pixels})$. Regression training is $O(N_{holdout\_pixels} \cdot N_{trees} \cdot \text{depth})$. Inference is $O(N_{novel\_pixels})$.
*   **Memory Complexity:** Requires storing the 13-dimensional feature vector per primitive ($O(N_{primitives})$) and the rendered feature maps for holdout views during training ($O(N_{holdout\_views} \cdot H \cdot W \cdot 13)$).
*   **Strengths:** Post-hoc (no retraining); interpretable features; strong correlation with true errors, especially for depth and foreground objects; generalizes to unseen scenes; enables active view selection [S1, unofficial].
*   **Limitations:** Requires holdout data for regression training (unless generalizing to unseen scenes); 13 features may be complex to interpret fully; performance depends on the quality of the initial GS model [S1, unofficial].
*   **When to use:** Safety-critical domains like robotics and medical imaging where reliable uncertainty estimates are essential [S1, unofficial].
*   **When not to use:** When computational resources for post-processing are extremely limited and approximate variance-based methods are sufficient; when no holdout data is available and generalization to unseen scenes is not acceptable.

## Background
PRIMU was introduced by Stehr et al. from the University of Wuppertal, Queensland University of Technology, and University of Osnabrück [S1, unofficial]. It addresses the challenge of uncertainty estimation in 3D Gaussian Splatting, which is critical for deploying GS in safety-critical domains. Existing methods often estimate per-primitive uncertainty during training or rely on parameter variances, whereas PRIMU operates as a post-processing framework [S1, unofficial].

## Sources
[S1] PRIMU: Uncertainty Estimation for Novel Views in Gaussian Splatting from Primitive-Based Representations of Error and Coverage, https://arxiv.org/abs/2508.02443v2, fetched 2026-10-26 (unofficial)
