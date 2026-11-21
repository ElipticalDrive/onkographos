# Regression-based FisherRF

Also called: FisherRF with meta-regression, Modified FisherRF

## Overview
Regression-based FisherRF is a variant of the FisherRF uncertainty estimation method that replaces direct variance-to-pixel mapping with a meta-regression model [S1, unofficial]. It solves the problem of estimating pixel-wise rendering uncertainty by using grouped Gaussian parameter variances as input feature maps to predict uncertainty, rather than summing color parameter variances directly [S1, unofficial]. It differs from the original FisherRF by aligning its pipeline with PRIMU's feature map regression approach, allowing for fair comparison of uncertainty estimation quality [S1, unofficial].

## How it works
**Inputs:**
*   A set of 3D Gaussian primitives, each defined by mean position, scale, rotation, opacity, and spherical harmonics (SH) coefficients [S1, unofficial].
*   Camera parameters for the target view [S1, unofficial].

**Outputs:**
*   A set of six uncertainty feature maps, one for each parameter group [S1, unofficial].
*   A pixel-wise uncertainty map predicted by the regression model [S1, unofficial].

**Data Structures:**
The method relies on the standard 3D Gaussian Splatting primitive structure [14, cited in S1, unofficial]. The parameters are grouped into six categories based on the original GS implementation [S1, unofficial]:
1.  Mean (position)
2.  Scale
3.  Rotation
4.  Opacity factor
5.  Spherical harmonics color parameters of degree zero ($c_0$)
6.  Higher degree spherical harmonics color parameters ($c_{k>0}$)

Note: The scale and rotation groups correspond to the primitive's covariance matrix [S1, unofficial].

**Steps:**
1.  **Compute Parameter Variances:** For each Gaussian primitive, compute the variance of its parameters. In the context of FisherRF, these variances are derived from the diagonal entries of the Fisher Information Matrix of the Gaussian primitive with respect to the gradient of the rendering function [S1, unofficial].
2.  **Group Variances:** Aggregate the computed variances per primitive into the six predefined categories [S1, unofficial].
3.  **Form Feature Maps:** Aggregate the grouped variances across all primitives to form six distinct uncertainty feature maps, one for each category [S1, unofficial].
4.  **Regression:** Feed the six feature maps into a meta-regression model [S1, unofficial].
5.  **Predict Uncertainty:** The regression model outputs a pixel-wise uncertainty map [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def regression_based_fisherrf(gaussians, camera):
    """
    Compute uncertainty feature maps and predict pixel-wise uncertainty.
    
    Args:
        gaussians: List of Gaussian primitives. Each primitive has:
            - mean: (3,)
            - scale: (3,)
            - rotation: (4,) (quaternion)
            - opacity: (1,)
            - sh_coeffs: (K, 3) where K is number of SH bands
        camera: Camera parameters for projection.
        
    Returns:
        uncertainty_map: (H, W) predicted uncertainty.
        feature_maps: List of 6 tensors, each (H, W).
    """
    # Step 1: Compute variances for each primitive
    # In FisherRF, variance is derived from the Fisher Information Matrix diagonal.
    # This requires computing the gradient of the rendering function w.r.t. parameters.
    # For implementation, we assume a function `compute_fisher_variances` exists
    # that returns the variance for each parameter group per primitive.
    variances = compute_fisher_variances(gaussians, camera)
    
    # Step 2 & 3: Group variances and form feature maps
    # Group 1: Mean
    mean_variance = variances['mean']
    feature_map_mean = aggregate_to_image(mean_variance, gaussians, camera)
    
    # Group 2: Scale
    scale_variance = variances['scale']
    feature_map_scale = aggregate_to_image(scale_variance, gaussians, camera)
    
    # Group 3: Rotation
    rotation_variance = variances['rotation']
    feature_map_rotation = aggregate_to_image(rotation_variance, gaussians, camera)
    
    # Group 4: Opacity
    opacity_variance = variances['opacity']
    feature_map_opacity = aggregate_to_image(opacity_variance, gaussians, camera)
    
    # Group 5: SH Degree 0
    sh0_variance = variances['sh_degree_0']
    feature_map_sh0 = aggregate_to_image(sh0_variance, gaussians, camera)
    
    # Group 6: SH Degree > 0
    sh_high_variance = variances['sh_higher']
    feature_map_sh_high = aggregate_to_image(sh_high_variance, gaussians, camera)
    
    feature_maps = [
        feature_map_mean,
        feature_map_scale,
        feature_map_rotation,
        feature_map_opacity,
        feature_map_sh0,
        feature_map_sh_high
    ]
    
    # Step 4 & 5: Regression
    # The regression model is not specified in the source.
    # It is assumed to be a neural network that takes the 6 feature maps as input.
    # [G] Standard practice would be a CNN or MLP.
    uncertainty_map = regression_model(feature_maps)
    
    return uncertainty_map, feature_maps

def compute_fisher_variances(gaussians, camera):
    """
    Compute variances based on Fisher Information Matrix.
    [G] This is a placeholder for the specific FisherRF variance computation.
    The source states that diagonal entries of the Fisher Information Matrix
    correspond to variances of the Gaussian primitive parameters.
    """
    # Implementation details for Fisher Information Matrix computation are not provided
    # in the source text. This function would compute the Hessian or gradient
    # statistics to estimate parameter variances.
    raise NotImplementedError("Fisher Information Matrix computation details not provided in source.")

def aggregate_to_image(variances, gaussians, camera):
    """
    Aggregate per-primitive variances into a 2D image feature map.
    [G] Standard alpha-blending or accumulation based on rasterization order.
    """
    # Implementation details for aggregation are not provided in the source text.
    # This function would project primitives to 2D and accumulate variances
    # at each pixel, likely using the same rasterization pipeline as rendering.
    raise NotImplementedError("Aggregation details not provided in source.")
```

**Parameters:**
*   **Parameter Grouping:** The six groups are fixed as defined in the original GS implementation [14, cited in S1, unofficial].
*   **Regression Model:** The specific architecture of the meta-regression model is not given in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Fisher Information Matrix Computation:** The source does not detail how the Fisher Information Matrix is computed or approximated. This is a critical step for obtaining the variances [S1, unofficial].
*   **Aggregation:** The method of aggregating per-primitive variances into feature maps is not specified [S1, unofficial].

## Subtopics

### FisherRF
The original method that uses the sum of color parameter variances for direction-dependent uncertainty. Regression-based FisherRF is a modification of this [S1, unofficial].

### PRIMU
The primary method that uses a meta-regression model for uncertainty estimation. Regression-based FisherRF is designed to be comparable to PRIMU [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Used as a baseline for uncertainty estimation (UE) scene separation studies and out-of-scene generalization experiments [S1, unofficial].
*   **Strengths:** Aligns with PRIMU's pipeline for fair comparison [S1, unofficial].
*   **Limitations:** Relies on the Fisher Information Matrix approximation, which may be computationally expensive or inaccurate [S1, unofficial].

## Background
Regression-based FisherRF was introduced in the study of uncertainty estimation for novel views in Gaussian Splatting [S1, unofficial]. It is based on the original FisherRF method, which uses the Fisher Information Matrix to estimate uncertainty [S1, unofficial]. The modification was made to make FisherRF more similar to PRIMU's method, allowing for a fair comparison of uncertainty estimation quality [S1, unofficial].

## Sources
[S1] PRIMU: Uncertainty Estimation for Novel Views in Gaussian Splatting from Primitive-Based Representations of Error and Coverage, https://arxiv.org/abs/2508.02443v2, fetched 2026-10-26 (unofficial)
