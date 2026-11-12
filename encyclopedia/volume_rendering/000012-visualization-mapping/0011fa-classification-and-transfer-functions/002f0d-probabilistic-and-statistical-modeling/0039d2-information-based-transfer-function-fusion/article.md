# Information-based Transfer Function Fusion

Also called: IBTF, Information-based Data Fusion, Information-based Transfer Function Classification, Information-based Transfer Functions for Multimodal Visualization

## Overview
Information-based Transfer Function Fusion (IBTF) is a method for designing transfer functions in multimodal volume visualization that reduces the complexity of multi-dimensional classification by fusing values from multiple registered modalities into a single scalar value using information theory [S1]. It solves the problem of combining complementary information from different modalities (e.g., CT and MR) into a single coherent visualization where both inputs are represented at each voxel, rather than discarding one [S1]. It sets itself apart from manual methods by automating the design and from single-modality optimization methods by defining an information channel between two input data sets [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two registered volume datasets $X$ and $Y$ with scalar values $f_1$ and $f_2$ at each voxel, and their respective gradient vectors $\nabla f_1$ and $\nabla f_2$ [S1].
*   **Outputs:** A fused scalar value $f_{fused}$, a fused gradient vector $\nabla f_{fused}$, and an "opposite information" metric $\delta$ [S1]. These are used to look up color $c$ and modified opacity $\alpha_{mod}$ from transfer function lookup tables (LUTs) [S1].

**Data Structures:**
The method relies on four pre-computed 2D Lookup Tables (LUTs) generated from histograms of the input data [S1]:
1.  **$\gamma$ LUT:** Maps input value pairs $(f_1, f_2)$ to the weighting factor $\gamma$ [S1].
2.  **$\delta$ LUT:** Maps input value pairs $(f_1, f_2)$ to the opposite information metric $\delta$ [S1].
3.  **Transfer Function LUT:** Maps fused coordinates $(f_{fused}, \|\nabla f_{fused}\|)$ to optical properties $(c, \alpha)$ [S1].
4.  **Windowing Parameters LUT:** Maps fused coordinates $(f_{fused}, \|\nabla f_{fused}\|)$ to windowing parameters $(\delta_{pos}, \delta_{width})$ [S1].

**Steps:**
1.  **Pre-processing (Histogram Generation):** Generate individual histograms for both modalities to estimate marginal probability distributions $P(f_1)$ and $P(f_2)$, and generate a 2D "dual" histogram to estimate the joint probability distribution $P(f_1, f_2)$ [S1]. Calculate $\gamma(f_1, f_2)$ and $\delta(f_1, f_2)$ for all value pairs in the histogram bins and store them in the $\gamma$ and $\delta$ LUTs respectively [S1]. This is done once before rendering as histograms are static [S1].
2.  **Probability Estimation:** Estimate the marginal probability $P(f)$ for each value $f$ by dividing the count of voxels in each histogram bin by the total number of points $N$ in the volume [S1]. Estimate the joint probability $P(f_1, f_2)$ using the dual histogram counts divided by $N$ [S1].
3.  **Information Content Calculation:** Calculate the information content $I(f)$ for each value based on its frequency, where values occurring rarely have high information [S1]. Formula: $I(f) = -\log_2(P(f))$ [S1].
4.  **Fusion Weighting ($\gamma$):** Calculate the weighting factor $\gamma$ which determines how much each modality contributes to the fused value based on its information content [S1]. Formula: $\gamma(f_1, f_2) = \frac{I(f_2)}{I(f_1) + I(f_2)}$ [S1]. Properties: $\gamma=0$ if modality 2 has no information, $\gamma=1$ if modality 1 has no information, and $\gamma=0.5$ if both have equal information [S1].
5.  **Value and Gradient Fusion:** Fuse the scalar values and gradient vectors using $\gamma$ to ensure the modality with higher information content has a stronger influence on the result [S1]. Fused Value: $f_{fused} = (1 - \gamma) \cdot f_1 + \gamma \cdot f_2$ [S1]. Fused Gradient: $\nabla f_{fused} = (1 - \gamma) \cdot \nabla f_1 + \gamma \cdot \nabla f_2$ [S1].
6.  **Opposite Information Calculation ($\delta$):** Calculate Pointwise Mutual Information (PMI) to measure the dependency between $f_1$ and $f_2$ [S1]. Formula: $PMI(f_1, f_2) = \log_2\left(\frac{P(f_1, f_2)}{P(f_1) \cdot P(f_2)}\right)$ [S1]. Normalize PMI to the range $[0, 1]$ to create $PMI_{norm}$ using the source formula: $PMI_{norm}(f_1, f_2) = \frac{PMI(f_1, f_2) - \log_2(P(f_1, f_2))}{\log_2(P(f_1)) - \log_2(P(f_1, f_2))}$ [S1]. Calculate $\delta$ as the inversion of $PMI_{norm}$, where $\delta$ approaches 0 for high information (correlated) pairs and close to 1 for low information (complementary/noise) pairs [S1]. Formula: $\delta(f_1, f_2) = 1 - PMI_{norm}(f_1, f_2)$ [S1].
7.  **Rendering Loop (Per Sample Point):** Lookup $\gamma$ and $\delta$ from pre-computed LUTs using input values $f_1, f_2$ [S1]. Calculate $f_{fused}$ and $\nabla f_{fused}$ using $\gamma$ and the input gradients [S1]. Lookup base color $c$ and base opacity $\alpha$ from the Transfer Function LUT using $(f_{fused}, \|\nabla f_{fused}\|)$ [S1]. Lookup windowing parameters $\delta_{pos}$ and $\delta_{width}$ from the Windowing Parameters LUT using $(f_{fused}, \|\nabla f_{fused}\|)$ [S1]. Calculate the windowing weight $\omega(\delta)$ using $\delta$, $\delta_{pos}$, and $\delta_{width}$ [S1]. Calculate modified opacity $\alpha_{mod} = \alpha \cdot \omega(\delta)$ [S1]. Use $c$ and $\alpha_{mod}$ for shading and compositing [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calculate_information(probability):
    """I(f) = -log2(P(f))"""
    if probability <= 0:
        return float('inf')
    return -math.log2(probability)

def calculate_gamma(I_f1, I_f2):
    """γ = I(f2) / (I(f1) + I(f2))"""
    if I_f1 == float('inf') and I_f2 == float('inf'):
        return 0.5
    if I_f2 == float('inf'):
        return 1.0
    if I_f1 == float('inf'):
        return 0.0
    return I_f2 / (I_f1 + I_f2)

def calculate_pmi(P_f1, P_f2, P_f1_f2):
    """PMI = log2(P(f1,f2) / (P(f1)*P(f2))). PMI=0 if P(f1,f2)=0."""
    if P_f1_f2 == 0:
        return 0.0
    return math.log2(P_f1_f2 / (P_f1 * P_f2))

def calculate_pmi_norm(PMI, P_f1, P_f1_f2):
    """PMI_norm = (PMI - log2(P(f1,f2))) / (log2(P(f1)) - log2(P(f1,f2)))"""
    log_joint = math.log2(P_f1_f2)
    log_marginal_f1 = math.log2(P_f1)
    denominator = log_marginal_f1 - log_joint
    if denominator == 0:
        return 0.0
    numerator = PMI - log_joint
    return numerator / denominator

def calculate_delta(PMI_norm):
    """δ = 1 - PMI_norm"""
    return 1.0 - PMI_norm

def windowing_function(delta, delta_pos, delta_width):
    """ω(δ) = max(|(δ - δ_pos) / (0.5 * δ_width)|, 0)"""
    if delta_width == 0:
        return 0.0
    val = abs((delta - delta_pos) / (0.5 * delta_width))
    return max(1.0 - val, 0.0)

def get_bin_index(value, num_bins, min_val, max_val):
    """Standard linear binning helper [G]. Maps continuous value to discrete bin index."""
    if max_val == min_val:
        return 0
    normalized = (value - min_val) / (max_val - min_val)
    return max(0, min(num_bins - 1, int(normalized * num_bins)))

def render_sample_point(f1, f2, grad1, grad2, 
                        hist_marginal_f1, hist_marginal_f2, hist_joint, 
                        N, num_bins, min_val, max_val,
                        tf_lut, windowing_lut):
    """Renders a single sample point using IBTF."""
    # 1. Map values to histogram bins
    bin_f1 = get_bin_index(f1, num_bins, min_val, max_val)
    bin_f2 = get_bin_index(f2, num_bins, min_val, max_val)
    
    # 2. Estimate Probabilities
    P_f1 = hist_marginal_f1[bin_f1] / N
    P_f2 = hist_marginal_f2[bin_f2] / N
    P_f1_f2 = hist_joint[(bin_f1, bin_f2)] / N
    
    # 3. Calculate Information Content
    I_f1 = calculate_information(P_f1)
    I_f2 = calculate_information(P_f2)
    
    # 4. Calculate Gamma
    gamma = calculate_gamma(I_f1, I_f2)
    
    # 5. Fuse Values and Gradients
    f_fused = (1 - gamma) * f1 + gamma * f2
    grad_fused_x = (1 - gamma) * grad1.x + gamma * grad2.x
    grad_fused_y = (1 - gamma) * grad1.y + gamma * grad2.y
    grad_fused_z = (1 - gamma) * grad1.z + gamma * grad2.z
    norm_grad_fused = math.sqrt(grad_fused_x**2 + grad_fused_y**2 + grad_fused_z**2)
    
    # 6. Calculate Delta (Opposite Information)
    PMI = calculate_pmi(P_f1, P_f2, P_f1_f2)
    PMI_norm = calculate_pmi_norm(PMI, P_f1, P_f1_f2)
    delta = calculate_delta(PMI_norm)
    
    # 7. Lookup Optical Properties
    color_c, alpha_base = tf_lut.lookup(f_fused, norm_grad_fused)
    delta_pos, delta_width = windowing_lut.lookup(f_fused, norm_grad_fused)
    
    # 8. Modify Opacity
    omega = windowing_function(delta, delta_pos, delta_width)
    alpha_mod = alpha_base * omega
    
    return color_c, alpha_mod
```

**Parameters:**
*   $\delta_{pos}$: The center position parameter for the $\delta$ windowing function. Defines the target $\delta$ value range for tissue selection [S1].
*   $\delta_{width}$: The width parameter for the $\delta$ windowing function. Defines the spread of the $\delta$ value range accepted for a tissue [S1].

**Edge Cases and Pitfalls:**
*   **Zero Joint Probability:** If $P(f_1, f_2) = 0$, PMI is defined as 0 by definition [S1]. The normalization formula handles this via the bounds without requiring explicit zero-probability checks before calculation [S1].
*   **Noise:** Noise flattens the probability distribution function but does not change the relation between probabilities, so the $\delta$ value is not affected by noise [S1].
*   **Gradient Fusion:** Gradients must be fused using the same $\gamma$ as the values to ensure shading and gradient-based classification use consistent information weighting [S1].

## Subtopics

### [Information Theory-Based Multimodal Transfer Function Design](9731-information-theory-based-multimodal-transfer-function-design.md)
An automated method for designing multimodal transfer functions for two registered volume datasets $X$ and $Y$, using mutual information decomposition to guide color fusion and opacity optimization. It solves the problem of combining complementary information from different modalities (e.g., CT and MR) into a single coherent visualization where both inputs are represented at each voxel, rather than discarding one. It sets itself apart from manual methods by automating the design and from single-modality optimization methods by defining an information channel between two input data sets.

## Operating characteristics
*   **Time Complexity:** Pre-processing complexity is dominated by histogram generation, which is $O(N)$ for $N$ points [S1]. Rendering complexity per sample point is $O(1)$ due to LUT lookups and simple arithmetic [S1].
*   **Memory Complexity:** Requires storage for four 2D LUTs, where the size depends on the number of bins in the histograms [S1].
*   **Performance:** Enables real-time frame rates on graphics hardware because all processing steps can be executed efficiently and the rendering loop involves only fast lookups and arithmetic [S1].
*   **Strengths:** Reduces the complexity of multi-dimensional classification to a standard 2D transfer function space [S1]. Automates transfer function design [S1]. Provides better separation of different tissues by using the $\delta$ metric for complementary information [S1]. Robust to noise [S1].
*   **Limitations:** Requires registered volume datasets [S1]. The quality of the fusion depends on the accuracy of the histogram estimation [S1]. The normalization formula for PMI contains a potential ambiguity/typo in the source literature [S1].
*   **When to use:** Multimodal visualization where complementary information from different modalities needs to be combined automatically without manual tuning [S1].
*   **When not to use:** Single-modality visualization (simpler methods suffice) or when manual control over the fusion weighting is preferred over information-theoretic automation [S1].

## Background
IBTF was introduced by Haidacher et al. in 2008 [S1]. It is based on methods from information theory, specifically using Shannon's information content and Pointwise Mutual Information (PMI) to guide the fusion of multimodal data [S1]. The method addresses the challenge of designing transfer functions for high-dimensional parameter spaces by fusing them into a simpler 2D space augmented with an opposite information metric [S1].

## Sources
[S1] Information-based Transfer Functions for Multimodal Visualization, https://doi.org/10.2312/vcbm/vcbm08/101-108, fetched 2026-10-26
[S2] Information Theory-Based Automatic Multimodal Transfer Function Design, https://doi.org/10.1109/jbhi.2013.2263227, fetched 2026-10-27
