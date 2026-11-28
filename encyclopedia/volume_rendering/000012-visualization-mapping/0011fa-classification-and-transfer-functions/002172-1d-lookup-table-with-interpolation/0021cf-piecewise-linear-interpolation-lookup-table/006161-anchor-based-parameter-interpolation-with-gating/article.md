# Anchor-based Parameter Interpolation with Gating

Also called: Interval Interpolation and Gate

## Overview
This method interpolates 3D Gaussian parameters between two sparse anchors to support continuous budget decoding in 3D Gaussian Splatting [S1, unofficial]. It solves the problem of smoothly transitioning from a lower-budget state to a higher-budget state by linearly interpolating shared members and gating the opacity of newly entering members based on normalized progress [S1, unofficial]. It differs from standard [Piecewise Linear Interpolation](16011-piecewise-linear-interpolation.md) by using power-law exponents for both parameter mixing and opacity gating, allowing non-linear temporal control over the budget expansion [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: Lower anchor state $\theta(L)$, upper anchor state $\theta(U)$, requested member count $K$, lower anchor count $L$, upper anchor count $U$ (where $L < K < U$), normalized progress $t \in (0, 1)$, exponents $p > 0$ and $q > 0$, and opacity bound $\epsilon > 0$ [S1, unofficial].
- **Outputs**: Interpolated parameters $\theta_i(K)$ for shared members and gated opacity $a_i(K)$ for entering members [S1, unofficial].

**Data Structures**
- Members are ordered by a fixed nested sequence [S1, unofficial].
- Members $i \le L$ are "shared" (present in both anchors).
- Members $L < i \le K$ are "entering" (present only in the upper anchor, admitted into the current budget) [S1, unofficial].

**Steps**
1. Calculate normalized progress $t$ based on the requested count $K$ relative to the anchor interval:
   $$ t = \frac{K - L}{U - L} $$
2. For each shared member $i \le L$, interpolate parameters $\theta_i$ using power-law weighting:
   $$ \theta_i(K) = (1 - t^p)\theta_i(L) + t^p \theta_i(U) $$
   Here, $p$ controls the speed of convergence to the upper state [S1, unofficial].
3. For each entering member $i$ where $L < i \le K$:
   - Retrieve geometry and appearance from the upper anchor $\theta_i(U)$ [S1, unofficial].
   - Calculate the base opacity logit $o_i(U)$ and convert to probability $a_i(U) = \sigma(o_i(U))$ [S1, unofficial].
   - Gate the opacity by scaling with $t^q$ and clipping to $[\epsilon, 1-\epsilon]$:
     $$ a_i(K) = \text{clip}(t^q \cdot a_i(U), \epsilon, 1 - \epsilon) $$
   - Convert the clipped opacity back to a finite logit for storage/usage [S1, unofficial].

**Parameters**
- $p$: Exponent for parameter interpolation. $p=1$ is linear. $p<1$ approaches the upper state earlier; $p>1$ approaches later [S1, unofficial].
- $q$: Exponent for opacity gating. Controls how quickly entering members reach full opacity [S1, unofficial].
- $\epsilon$: Small value bounding opacity to avoid extreme values during sigmoid conversion [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid(x):
    return 1.0 / (1.0 + math.exp(-x))

def anchor_interpolation(theta_L, theta_U, L, K, U, t, p, q, epsilon):
    """
    Interpolate parameters and gate opacity for a requested budget K
    between anchors L and U.
    
    Args:
        theta_L: List of parameter tuples for shared members (indices 0 to L-1).
        theta_U: List of parameter tuples for all upper anchor members (indices 0 to U-1).
        L: Number of shared members.
        K: Requested number of members.
        U: Total number of members in upper anchor.
        t: Normalized progress (0 to 1).
        p: Interpolation exponent.
        q: Opacity gating exponent.
        epsilon: Opacity bound.
        
    Returns:
        theta_K: List of interpolated parameter tuples for K members.
    """
    theta_K = []
    
    # Precompute powers
    t_p = t ** p
    t_q = t ** q
    
    # 1. Interpolate shared members (i <= L)
    # Note: Indices in lists are 0-based. 
    # Shared members are indices 0 to L-1 in both theta_L and theta_U.
    for i in range(L):
        # Interpolate each parameter component
        theta_i_L = theta_L[i]
        theta_i_U = theta_U[i]
        
        # Linear interpolation in parameter space with power-law weighting
        theta_i_K = tuple(
            (1 - t_p) * val_L + t_p * val_U 
            for val_L, val_U in zip(theta_i_L, theta_i_U)
        )
        theta_K.append(theta_i_K)
        
    # 2. Handle entering members (L < i <= K)
    # These members exist in theta_U but not in theta_L.
    # We take their geometry/appearance from theta_U and gate their opacity.
    for i in range(L, K):
        theta_i_U = theta_U[i]
        
        # Extract opacity logit from upper anchor state
        # Assuming the last element of the parameter tuple is the opacity logit
        # [G] Specific field index depends on the Gaussian parameter layout.
        # Standard 3DGS: [mean, scale, rotation, opacity_logit, sh_coeffs...]
        # Here we assume opacity_logit is accessible.
        opacity_logit_U = theta_i_U[-1] 
        
        # Convert to probability
        opacity_prob_U = sigmoid(opacity_logit_U)
        
        # Gate opacity
        gated_opacity = t_q * opacity_prob_U
        
        # Clip to [epsilon, 1 - epsilon]
        clipped_opacity = max(epsilon, min(1 - epsilon, gated_opacity))
        
        # Convert back to logit
        # [G] Inverse sigmoid: log(clipped / (1 - clipped))
        if clipped_opacity <= 0 or clipped_opacity >= 1:
            # Fallback if clipping results in exact 0 or 1 (should be prevented by epsilon)
            clipped_logit = 0.0
        else:
            clipped_logit = math.log(clipped_opacity / (1 - clipped_opacity))
            
        # Construct new parameter tuple with gated opacity
        # Geometry and appearance (all fields except opacity logit) are taken from upper anchor
        new_params = list(theta_i_U)
        new_params[-1] = clipped_logit
        theta_K.append(tuple(new_params))
        
    return theta_K
```

**Edge Cases and Pitfalls**
- Opacity must be bounded to $[\epsilon, 1-\epsilon]$ before sigmoid conversion to avoid extreme logit values [S1, unofficial].
- The nested order of members must be fixed and consistent between anchors [S1, unofficial].
- $t$ is calculated as $(K-L)/(U-L)$, so $t=0$ corresponds to $K=L$ and $t=1$ corresponds to $K=U$ [S1, unofficial].

## Subtopics

### [Layered Residual Atlas](18368-layered-residual-atlas.md)
An encoding scheme for storing anchor changes relative to a full reference model, organized by member layer and parameter field. It solves the problem of efficiently storing and decoding specific anchor endpoints by decomposing the parameter space into layers of newly introduced members and applying orthogonal transforms to share residual bases across anchors. Unlike standard anchor storage which might duplicate full parameter sets, this method leverages the fact that most parameters are shared with the full model, storing only the residuals for members present at specific anchors.

## Operating characteristics
- **Time Complexity**: Linear in the number of members $K$ for interpolation and gating [S1, unofficial].
- **Memory Complexity**: Only two scalars ($p, q$) are stored per interval, plus the compressed anchor endpoints [S1, unofficial].
- **Performance**: Calibration of $p$ and $q$ is done per interval on training views while fixing anchors and member order [S1, unofficial].
- **Strengths**: Allows continuous budget decoding with smooth transitions. Efficient storage of only exponents and compressed endpoints [S1, unofficial].
- **Limitations**: Requires fixed nested order of members. Calibration adds preprocessing overhead [S1, unofficial].
- **When to use**: When continuous budget control is needed in 3D Gaussian Splatting with sparse anchors [S1, unofficial].
- **When not to use**: When member order is dynamic or not fixed [S1, unofficial].

## Background
This method is part of the ParetoGS framework, which uses sparse adapted anchors for high-quality continuous-budget 3D Gaussian Splatting [S1, unofficial]. It was introduced in 2026 [S1, unofficial].

## Sources
[S1] ParetoGS: High-Quality Continuous-Budget 3D Gaussian Splatting with Sparse Adapted Anchors, https://arxiv.org/abs/2610.34235, fetched 2026-10-29 (unofficial)
