# Scalar to Color Mapping Scale

Also called: Linear scale, Log Scale Mapping, Log10 scale, Logarithmic scale

## Overview
Scalar to Color Mapping Scale determines the type of scale used to map scalar values to the transfer function range, supporting linear or logarithmic (base 10) modes [S1]. It differs from sibling methods like [Colormap Conversion](7117-colormap-conversion.md) by defining the input domain transformation rather than the output color space interpolation [G].

## How it works
**Inputs:** Scalar value $v$, data range $[v_{\min}, v_{\max}]$, scale type $S \in \{\text{linear}, \text{log10}\}$ [S1].
**Outputs:** Normalized value $n \in [0, 1]$ [G].

**Data Structures:**
The scale type is stored as an integer flag within the transfer function object [G].

**Steps:**
1. Determine the effective scale type $S_{\text{eff}}$ [S1]. If $S = \text{log10}$ and the data range $[v_{\min}, v_{\max}]$ contains zero, set $S_{\text{eff}} = \text{linear}$ to avoid undefined behavior [S1]. Otherwise, $S_{\text{eff}} = S$.
2. If $S_{\text{eff}} = \text{linear}$, compute the normalized value:
   $$ n = \frac{v - v_{\min}}{v_{\max} - v_{\min}} $$
3. If $S_{\text{eff}} = \text{log10}$, compute the normalized value:
   $$ n = \frac{\log_{10}(v) - \log_{10}(v_{\min})}{\log_{10}(v_{\max}) - \log_{10}(v_{\min})} $$
   Log scale mapping assumes data is in the non-zero, positive range [S2].
4. Clamp $n$ to $[0, 1]$.

**Parameters:**
- Default scale type: Linear [S1].

**Edge Cases and Pitfalls:**
- **Zero in Range:** If logarithmic scale is requested but the data range contains zero, the color mapping defaults to linear [S1].
- **Non-Positive Data:** Log scale mapping assumes data is in the non-zero, positive range [S2]. ParaView reports errors and tries to automatically fix the range if it is invalid for log mapping [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def map_scalar_to_color_scale(v, v_min, v_max, scale_type):
    """
    Maps a scalar value to [0, 1] using the specified scale.
    
    Args:
        v: Scalar value to map.
        v_min: Minimum value of the data range.
        v_max: Maximum value of the data range.
        scale_type: 'linear' or 'log10'.
        
    Returns:
        Normalized value in [0, 1].
    """
    effective_scale = scale_type
    
    # Pitfall: Log scale fails if range contains zero
    if effective_scale == 'log10' and (v_min <= 0 or v_max <= 0):
        effective_scale = 'linear'
        
    if effective_scale == 'linear':
        if v_max == v_min:
            return 0.0
        n = (v - v_min) / (v_max - v_min)
    elif effective_scale == 'log10':
        log_min = math.log10(v_min)
        log_max = math.log10(v_max)
        if log_max == log_min:
            return 0.0
        log_v = math.log10(v)
        n = (log_v - log_min) / (log_max - log_min)
    else:
        if v_max == v_min:
            return 0.0
        n = (v - v_min) / (v_max - v_min)
        
    return max(0.0, min(1.0, n))
```

## Operating characteristics
Not given in the sources.

## Background
The implementation originates from the VTK library (`vtkColorTransferFunction`), which provides `SetScaleToLinear()` and `SetScaleToLog10()` methods to control the normalization function [S1]. Visualization tools like ParaView adopt this behavior, offering a "Use log scale when mapping data to colors" option that enforces the non-zero, positive data assumption and includes automatic range correction mechanisms [S2].

## Sources
[S1] vtkColorTransferFunctions, http://www.vtk.org/doc/nightly/html/classvtkColorTransferFunction.html, fetched 2026-10-29
[S2] 2. Color maps and transfer functions — ParaView ..., https://cep.tacc.utexas.edu/user-guide/ReferenceManual/colorMapping.html, fetched 2026-10-29
