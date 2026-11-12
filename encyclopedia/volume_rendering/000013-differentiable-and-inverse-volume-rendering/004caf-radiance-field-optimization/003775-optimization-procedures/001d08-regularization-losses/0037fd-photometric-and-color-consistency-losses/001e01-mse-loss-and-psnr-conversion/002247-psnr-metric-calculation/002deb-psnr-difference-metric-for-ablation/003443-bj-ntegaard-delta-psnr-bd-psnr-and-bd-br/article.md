# Bjøntegaard Delta PSNR (BD-PSNR) and BD-BR

Also called: BD-BR, BD-PSNR, Bjøntegaard Delta

## Overview
BD-PSNR and BD-BR are metrics used to evaluate lossy compression performance by calculating the average difference in quality (BD-PSNR) or bit-rate (BD-BR) between two rate-distortion (RD) curves [S1]. They provide a single scalar value that facilitates the objective comparison of compression techniques, avoiding the impracticality of displaying full RD curves for multiple experiments [S1].

## How it works
**Inputs:** Two sets of rate-distortion points: a reference set $R_{ref} = \{(R_{ref,i}, D_{ref,i})\}$ and a test set $R_{test} = \{(R_{test,j}, D_{test,j})\}$, where $R$ represents the bit-rate (or model size) and $D$ represents the distortion metric (e.g., PSNR in dB) [S1, S2].

**Outputs:**
- BD-PSNR: The average difference in PSNR (dB) between the two curves [S1].
- BD-BR: The average relative difference in bit-rate between the assessed and reference techniques [S1].

**Data Structures:**
- Two lists of tuples: `reference_points` and `test_points`. Each tuple contains `(rate, distortion)`.

**Steps:**
1. **Interpolation:** Fit a continuous rate-distortion curve for both the reference and test sets using piecewise cubic interpolation [S2]. This step transforms discrete experimental points into continuous functions $D_{ref}(R)$ and $D_{test}(R)$ [S2].
2. **Integration:** Calculate the area between the two interpolated curves over the range of rates where both curves are defined [S1].
   - For BD-PSNR, this area represents the average difference in quality [S1].
   - For BD-BR, the calculation determines the bit saving at the same quality level [S2].
3. **Normalization:** Divide the calculated area by the range of the integration domain to obtain the average difference [S1].

**Formulas:**
The specific mathematical formulation for the integration and normalization is not provided in the sources [S1, S2]. The sources describe the method conceptually as calculating the "average difference" or "bit saving" derived from the area between the curves [S1, S2].

**Parameters:**
- Interpolation method: Piecewise cubic interpolation [S2].

**Edge cases and pitfalls:**
- The method requires overlapping rate ranges for both curves to perform integration [S1].
- Discrete points must be sufficient to capture the shape of the RD curve for accurate interpolation [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_bjontegaard_delta(reference_points, test_points):
    """
    Calculates BD-PSNR and BD-BR using piecewise cubic interpolation.
    Note: The exact integration algorithm is not specified in the sources.
    """
    # 1. Fit continuous curves using piecewise cubic interpolation
    # The sources specify piecewise cubic interpolation [S2]
    ref_curve = fit_piecewise_cubic(reference_points)
    test_curve = fit_piecewise_cubic(test_points)
    
    # 2. Determine the common rate range for integration
    # This is implied by the need to compare curves [S1]
    min_rate = max(p[0] for p in reference_points + test_points) # Placeholder for actual overlap logic
    max_rate = min(p[0] for p in reference_points + test_points) # Placeholder for actual overlap logic
    
    # 3. Integrate the area between the curves
    # The sources describe this as calculating the "average difference" [S1]
    # or "bit saving at the same quality" [S2].
    # The exact formula for the area calculation is not given.
    area = integrate_area_between_curves(ref_curve, test_curve, min_rate, max_rate)
    
    # 4. Normalize to get the average difference
    rate_range = max_rate - min_rate
    if rate_range == 0:
        return 0.0, 0.0
        
    bd_psnr = area / rate_range
    
    # BD-BR calculation involves bit saving at the same quality [S2]
    # The exact transformation from the area to BD-BR is not specified.
    bd_br = calculate_bd_br_from_area(area, ref_curve, test_curve) # Implementation detail not in sources
    
    return bd_psnr, bd_br

def fit_piecewise_cubic(points):
    """
    Fits a piecewise cubic curve to the given rate-distortion points.
    The specific coefficients and knot selection are not detailed in the sources.
    """
    # Sort points by rate
    sorted_points = sorted(points, key=lambda p: p[0])
    rates = [p[0] for p in sorted_points]
    distortions = [p[1] for p in sorted_points]
    
    # Perform piecewise cubic interpolation
    # Standard cubic spline or similar method would be used here
    # The source [S2] explicitly mentions "piecewise cubic interpolation"
    return CubicSpline(rates, distortions)

def integrate_area_between_curves(ref_curve, test_curve, min_rate, max_rate):
    """
    Integrates the area between the reference and test curves.
    The exact numerical integration method is not specified.
    """
    # Numerical integration (e.g., Simpson's rule or trapezoidal rule)
    # would be applied to the difference function |ref_curve(r) - test_curve(r)|
    # over the interval [min_rate, max_rate].
    # The source [S1] refers to this as the "gray area" in Fig. 5.
    pass

def calculate_bd_br_from_area(area, ref_curve, test_curve):
    """
    Calculates BD-BR from the integrated area.
    The exact formula is not provided in the sources.
    """
    pass
```

## Subtopics

### Bjøntegaard Delta-Rate (BD-Rate)
A metric for fair comparison of overall rate-distortion performance that calculates bit saving at the same quality level [S2]. It employs piecewise cubic interpolation to fit the continuous RD curve and calculate the bit saving [S2]. It has been applied to compare SPC-NeRF against VQRF on the Synthetic-NeRF dataset, showing 28% bit saving for the base configuration and 32% for the HR configuration [S2].

## Operating characteristics
**Time and memory complexity:** Not given in the sources.
**Measured performance:** Applied to compare SPC-NeRF against VQRF, showing 28% bit saving for the base configuration and 32% for the HR configuration on the Synthetic-NeRF dataset [S2].
**Strengths and limitations:** Provides a single scalar value for objective comparison, avoiding the impracticality of displaying full RD curves for multiple experiments [S1]. The method relies on accurate interpolation, which may be sensitive to the number of discrete points [S2].
**When to use it:** When comparing lossy compression techniques where a single metric summarizing the rate-distortion performance is needed [S1].

## Background
Originally designed within the Video Coding Experts Group (VCEG) – ITU SG 16 – for comparing video coding techniques [S1]. Implementation was written in Excel VBA [S1].

## Sources
[S1] Wavelet based volumetric medical image compression, https://doi.org/10.1016/j.image.2014.12.007, fetched 2026-10-26
[S2] SPC-NeRF: Spatial Predictive Compression for Voxel-Based Radiance Field, https://doi.org/10.1145/3795528, fetched 2026-10-26
