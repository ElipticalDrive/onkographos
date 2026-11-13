# Range of Emphasis Opacity
Also called: Opacity Scaling with Range of Emphasis

## Overview
Range of Emphasis Opacity is a transfer function technique that emphasizes specific density ranges by scaling opacity differently inside and outside a user-defined interval $[\rho_{min}, \rho_{max}]$ [S1, unofficial]. It sets itself apart from standard linear mapping by providing distinct control over the visibility of features within the emphasis range versus those outside it, allowing users to isolate structures of interest in volumetric data [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A scalar density value $\rho$ for a sample along the ray, typically normalized to $[0, 1]$.
*   **Parameters:**
    *   $\rho_{min}$: Lower bound of the emphasis range.
    *   $\rho_{max}$: Upper bound of the emphasis range.
    *   $\alpha_1$: Opacity multiplier for density values inside the range of emphasis.
    *   $\alpha_2$: Out-of-range opacity multiplier.
*   **Output:** The computed opacity $\alpha$ for the sample.

**Data Structures**
No specific complex data structures are required beyond the scalar parameters and the density value. The method is evaluated per sample during ray casting.

**Steps**
1.  Retrieve the normalized density value $\rho$ for the current sample.
2.  Calculate the opacity $\alpha$ using the piecewise function defined below.
3.  Use the calculated $\alpha$ in front-to-back alpha blending along the ray.
4.  Terminate the ray if the accumulated alpha is $\ge 0.96$ or the ray exits the volume [S1, unofficial].

**Formula**
The opacity $\alpha$ is calculated as:

$$
\alpha = \begin{cases} 
\rho \alpha_1 & \rho_{min} < \rho < \rho_{max} \\
\rho \left( \left(1 - \frac{\rho}{\rho_{min}}\right) \alpha_2 + \frac{\rho}{\rho_{min}} \alpha_1 \right) & \rho \le \rho_{min} \\
\rho \left( \left(1 - \frac{\rho - \rho_{max}}{1 - \rho_{max}}\right) \alpha_1 + \frac{\rho - \rho_{max}}{1 - \rho_{max}} \alpha_2 \right) & \rho \ge \rho_{max}
\end{cases}
$$

Where:
*   $\rho$ is the density value.
*   $\alpha_1$ is the opacity multiplier for the range of emphasis.
*   $\alpha_2$ is the out-of-range opacity multiplier.
*   $\rho_{min}$ is the lower bound of the density range of emphasis.
*   $\rho_{max}$ is the upper bound of the density range of emphasis.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_opacity(density, rho_min, rho_max, alpha1, alpha2):
    """
    Calculate opacity using Range of Emphasis Opacity.
    
    Args:
        density (float): Normalized density value rho.
        rho_min (float): Lower bound of emphasis range.
        rho_max (float): Upper bound of emphasis range.
        alpha1 (float): Opacity multiplier inside range.
        alpha2 (float): Opacity multiplier outside range.
        
    Returns:
        float: Calculated opacity alpha.
    """
    if rho_min < density < rho_max:
        # Inside the range of emphasis
        return density * alpha1
    elif density <= rho_min:
        # Below the range of emphasis
        # Linear interpolation between alpha2 (at rho=0) and alpha1 (at rho=rho_min)
        # scaled by density
        term1 = (1 - (density / rho_min)) * alpha2
        term2 = (density / rho_min) * alpha1
        return density * (term1 + term2)
    else:
        # Above the range of emphasis
        # Linear interpolation between alpha1 (at rho=rho_max) and alpha2 (at rho=1)
        # scaled by density
        # Note: The formula uses (rho - rho_max) / (1 - rho_max) as the weight for alpha2
        # and (1 - that weight) as the weight for alpha1
        weight_alpha2 = (density - rho_max) / (1 - rho_max)
        weight_alpha1 = 1 - weight_alpha2
        term1 = weight_alpha1 * alpha1
        term2 = weight_alpha2 * alpha2
        return density * (term1 + term2)
```

**Parameters and Typical Values**
*   $\alpha_1$: Typical values are 0.16 (Karman Vortex Street), 0.1 (Jet Stream), 0.1 (Dam Break) [S1, unofficial].
*   $\alpha_2$: Typical values are 0.01 (Karman Vortex Street, Jet Stream), 0.03 (Dam Break) [S1, unofficial].
*   $\rho_{min}$: Typical values are 0.31 (Karman Vortex Street), 0.55 (Jet Stream), 0.24 (Dam Break) [S1, unofficial].
*   $\rho_{max}$: Typical values are 0.37 (Karman Vortex Street), 0.59 (Jet Stream), 0.27 (Dam Break) [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Division by Zero:** If $\rho_{min} = 0$, the term $\rho / \rho_{min}$ in the $\rho \le \rho_{min}$ case causes division by zero. The sources do not specify a guard for this, but an implementation should handle $\rho_{min} = 0$ explicitly (e.g., treating the lower region as having opacity 0 or using a small epsilon) [G].
*   **Division by Zero:** If $\rho_{max} = 1$, the term $(1 - \rho_{max})$ in the denominator of the $\rho \ge \rho_{max}$ case causes division by zero. An implementation should handle $\rho_{max} = 1$ explicitly [G].
*   **Range Validity:** The method assumes $\rho_{min} < \rho_{max}$. If $\rho_{min} \ge \rho_{max}$, the "inside" range is empty or invalid, and the behavior is undefined in the sources [G].

## Operating characteristics
**Time and Memory Complexity**
The opacity calculation is $O(1)$ per sample, involving basic arithmetic operations. It adds negligible overhead compared to standard linear opacity mapping. Memory complexity is $O(1)$ as it requires only storing the four scalar parameters.

**Measured Performance**
Specific performance metrics (FPS, memory usage) are not provided in the sources. The method is described as suitable for real-time rendering [S1, unofficial].

**Strengths and Limitations**
*   **Strengths:** Allows interactive adjustment of emphasis ranges to highlight specific features (e.g., backflow regions in vortex streets) [S1, unofficial].
*   **Limitations:** Requires careful tuning of four parameters ($\rho_{min}, \rho_{max}, \alpha_1, \alpha_2$) for each dataset. The linear interpolation outside the range may not provide smooth transitions if parameters are poorly chosen.

**When to Use**
Use when the goal is to isolate and emphasize a specific density range in volumetric data, particularly in particle systems or density fields where background noise or high-density regions obscure features of interest [S1, unofficial].

## Background
**Origin**
The method was introduced in the paper "Illustrative Rendering of Particle Systems" [S1, unofficial].

**History and Development**
The technique was developed to enhance the visualization of particle systems, specifically for datasets like the von Karman vortex street, jet stream, and dam break simulations. It allows users to interactively change the range of emphasis to view different regions [S1, unofficial].

**Where it is Used**
It is used in the illustrative rendering of particle systems, often in conjunction with silhouette edges and shading models (Phong or Gooch) to enhance feature visibility [S1, unofficial].

## Sources
[S1] Illustrative Rendering of Particle Systems, https://doi.org/10.2312/pe.vmv.vmv13.177-185, fetched 2026-10-26 (unofficial)
