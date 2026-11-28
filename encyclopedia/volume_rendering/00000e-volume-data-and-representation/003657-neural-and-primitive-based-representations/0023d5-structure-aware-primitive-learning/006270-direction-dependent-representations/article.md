# Direction-Dependent Representations

Also called: Direction-dependent coverage, Direction-dependent error

## Overview
Direction-Dependent Representations extend scalar coverage and error metrics for Gaussian primitives by encoding how these values vary with the viewing direction [S1, unofficial]. This method solves the limitation of direction-independent aggregations, which treat all viewing angles equally, by weighting contributions based on the angular proximity to training views [S1, unofficial]. It sets itself apart from its siblings, such as Volume Exploration-Optimized INR and Mixture-of-Experts Neural Fields, by explicitly modeling the anisotropic reliability of primitive contributions using spherical distributions rather than learned latent codes or mixture weights [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A set of $K$ Gaussian primitives. For each primitive $k$, the method requires its opacity $\alpha_k(\bar{x})$, transmittance $T_k(\bar{x})$, and the per-pixel reconstruction error $e_{\bar{x}}$ (typically $\ell_1$) for all pixels $\bar{x}$ in the training set $\Omega$. It also requires the viewing direction $\vec{\nu}_v$ for each training view $v \in \Omega$.
*   **Output:** Two direction-dependent feature maps per primitive: the direction-dependent coverage $V^*(k, \vec{d})$ and the direction-dependent error $E^*(k, \vec{d})$, evaluated at a target direction $\vec{d}$. These are encoded as spherical harmonics coefficients for efficient rendering.

**Data Structures**
*   **Spherical Harmonics (SH):** Coefficients for degree 4 (16 coefficients per channel) are used to encode the directional variation, consistent with standard Gaussian Splatting color encoding [S1, unofficial].
*   **Aggregation:** The aggregation function $\text{agg}$ is selected from $\{\text{max}, \text{sum}, \text{mean}\}$ [S1, unofficial].

**Formulas**
The core weighting mechanism uses the von Mises-Fisher distribution [S1, unofficial]:
$$ f(\vec{x}; \vec{\nu}, \kappa) = \frac{\kappa e^{\kappa \vec{\nu}^\top \vec{x}}}{2\pi \sinh(\kappa)} $$
where $\vec{\nu}$ is the mean direction and $\kappa$ is the concentration parameter. The density is rescaled so its maximum equals one [S1, unofficial].

The direction-dependent coverage representation is:
$$ V^*(k, \vec{d}) = \max_{v \in \Omega} \text{agg}_{\bar{x} \in v^{(k)}} \left( \alpha_k(\bar{x}) T_k(\bar{x}) e^{\kappa \vec{\nu}_v^\top \vec{d} - \kappa} \right) $$

The direction-dependent error representation is:
$$ E^*(k, \vec{d}) = \text{mean}_{v \in \Omega} \text{agg}_{\bar{x} \in v^{(k)}} \left( e_{\bar{x}} \alpha_k(\bar{x}) T_k(\bar{x}) e^{\kappa \vec{\nu}_v^\top \vec{d} - \kappa} \right) $$

Here, $v^{(k)}$ denotes the pixels in view $v$ where the $k$-th Gaussian is visible (typically $\alpha_k(\bar{x}) \ge 1/255$ and $T_k(\bar{x}) \ge 10^{-3}$) [S1, unofficial]. $\vec{\nu}_v$ is the direction of training view $v$, and $\vec{d}$ is the evaluation direction [S1, unofficial].

**Parameters**
*   **Concentration $\kappa$:** Set to 8 in experiments [S1, unofficial].
*   **SH Degree:** 4 [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def von_mises_fisher_weight(nu, d, kappa):
    """
    Computes the rescaled von Mises-Fisher weight.
    nu: mean direction (3,)
    d: evaluation direction (3,)
    kappa: concentration parameter
    Returns: weight value
    """
    # Rescaled so max equals 1: exp(kappa * nu^T d - kappa)
    # Note: The source formula includes normalization constants, 
    # but states "rescaled so its maximum equals one". 
    # The exponential term exp(kappa * nu^T d - kappa) has max 1 when nu=d.
    return np.exp(kappa * np.dot(nu, d) - kappa)

def compute_direction_dependent_representatives(primitives, training_views, kappa=8, agg_func='max'):
    """
    primitives: list of Gaussian primitives. Each has opacity, transmittance, and visibility mask per view.
    training_views: list of view objects. Each has direction nu_v and pixel errors e_x.
    kappa: concentration parameter.
    agg_func: aggregation function ('max', 'sum', 'mean').
    """
    # Initialize storage for SH coefficients for V* and E*
    # Degree 4 SH has 16 coefficients. We store these per primitive.
    # For implementation, we sample directions or fit SH to the computed values.
    
    v_star_values = []
    e_star_values = []
    
    # Directions to evaluate (e.g., uniform sphere samples or specific view directions)
    # In practice, these are fitted to SH coefficients.
    eval_directions = get_uniform_sphere_samples(100) 
    
    for k, prim in enumerate(primitives):
        v_vals = []
        e_vals = []
        
        for d in eval_directions:
            v_sum = []
            e_sum = []
            
            for v in training_views:
                # Identify visible pixels for this primitive in this view
                visible_pixels = get_visible_pixels(prim, v, alpha_thresh=1/255, t_thresh=1e-3)
                
                if not visible_pixels:
                    continue
                
                # Aggregate contributions for this view
                view_contributions = []
                view_errors = []
                
                for x in visible_pixels:
                    # Weight by directional proximity
                    weight = von_mises_fisher_weight(v.nu, d, kappa)
                    
                    # Coverage term
                    coverage_term = prim.opacity[x] * prim.transmittance[x] * weight
                    view_contributions.append(coverage_term)
                    
                    # Error term
                    error_term = v.pixel_error[x] * prim.opacity[x] * prim.transmittance[x] * weight
                    view_errors.append(error_term)
                
                if not view_contributions:
                    continue
                    
                # Apply aggregation function across pixels for this view
                if agg_func == 'max':
                    agg_v = max(view_contributions)
                    agg_e = max(view_errors) # Note: Source uses mean over views, agg inside
                    # Correction: Source Eq 7 uses agg inside, then max over views.
                    # Source Eq 8 uses agg inside, then mean over views.
                    
                    # Re-reading Eq 7: max_v ( agg_x ( ... ) )
                    # Re-reading Eq 8: mean_v ( agg_x ( ... ) )
                    
                    # Let's compute agg_x first
                    agg_cov = max(view_contributions) if agg_func == 'max' else \
                              sum(view_contributions) if agg_func == 'sum' else \
                              np.mean(view_contributions)
                    
                    agg_err = max(view_errors) if agg_func == 'max' else \
                              sum(view_errors) if agg_func == 'sum' else \
                              np.mean(view_errors)
                    
                    v_sum.append(agg_cov)
                    e_sum.append(agg_err)
            
            # Aggregate over views
            if v_sum:
                if agg_func == 'max': # For V*, it's always max over views
                    v_val = max(v_sum)
                else:
                    v_val = max(v_sum) # Eq 7 specifies max over views regardless of inner agg? 
                    # Source Eq 7: max_v agg_x ...
                    # Source Eq 8: mean_v agg_x ...
                    # So V* is always max over views. E* is always mean over views.
                    v_val = max(v_sum) 
                
                if agg_func == 'mean': # For E*, it's always mean over views
                    e_val = np.mean(e_sum)
                else:
                    e_val = np.mean(e_sum)
                    
                v_vals.append(v_val)
                e_vals.append(e_val)
            else:
                v_vals.append(0.0)
                e_vals.append(0.0)
                
        # Fit Spherical Harmonics to v_vals and e_vals
        # This step is implied by "encoded using spherical harmonics"
        v_sh_coeffs = fit_sh(v_vals, eval_directions, degree=4)
        e_sh_coeffs = fit_sh(e_vals, eval_directions, degree=4)
        
        primitives[k].v_sh = v_sh_coeffs
        primitives[k].e_sh = e_sh_coeffs
```

**Edge Cases and Pitfalls**
*   **Visibility Thresholds:** Primitives are considered visible only when $\alpha_k(\bar{x}) \ge 1/255$ and $T_k(\bar{x}) \ge 10^{-3}$ [S1, unofficial]. Failing to apply these thresholds may include negligible contributions.
*   **Opacity Term:** The opacity term $\alpha_k(\bar{x})$ can be omitted from the aggregation to highlight low-opacity Gaussians that still influence reconstruction quality [S1, unofficial].
*   **Regression Model Capacity:** Linear regression models produce less nuanced uncertainty maps than gradient boosting models due to limited capacity [S1, unofficial].

## Subtopics

### Direction-Dependent PRIMU
A variant of PRIMU that uses direction-dependent coverage and error maps to improve uncertainty estimation accuracy [S1, unofficial]. It augments direction-dependent uncertainty feature maps with a Field of View (FoV) counter to compute predicted uncertainty [S1, unofficial]. Using direction-dependent uncertainty feature maps makes the predicted uncertainty maps more visually similar to the actual error maps compared to direction-independent maps [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The computation involves iterating over all primitives, training views, and visible pixels, then fitting spherical harmonics. The fitting step adds overhead but is done offline or during training. Rendering requires evaluating SH coefficients, which is standard.
*   **Memory Complexity:** Requires storing SH coefficients (degree 4, 16 floats) for coverage and error per primitive, in addition to standard Gaussian parameters.
*   **Performance:** In uncertainty estimation tasks, direction-dependent representations outperform direction-independent ones, particularly in scenes with complex geometry or varying lighting where viewing angle significantly impacts reconstruction quality [S1, unofficial].
*   **Strengths:** Captures anisotropic reliability; improves visual similarity of predicted error maps to ground truth [S1, unofficial].
*   **Limitations:** Requires training views with known directions; performance may drop in scene separation studies if depth scales are inconsistent across datasets (e.g., LF vs TUM) [S1, unofficial].

## Background
Direction-Dependent Representations were introduced in the context of Gaussian Splatting uncertainty estimation [S1, unofficial]. The method builds on the observation that scalar coverage and error metrics do not account for the directional nature of Gaussian splats. It utilizes the von Mises-Fisher distribution, a spherical analogue of the normal distribution, to model directional variation [S1, unofficial]. The approach was proposed in PRIMU: Uncertainty Estimation for Novel Views in Gaussian Splatting from Primitive-Based Representations of Error and Coverage [S1, unofficial].

## Sources
[S1] PRIMU: Uncertainty Estimation for Novel Views in Gaussian Splatting from Primitive-Based Representations of Error and Coverage, https://arxiv.org/abs/2508.02443v2, fetched 2026-10-26 (unofficial)
