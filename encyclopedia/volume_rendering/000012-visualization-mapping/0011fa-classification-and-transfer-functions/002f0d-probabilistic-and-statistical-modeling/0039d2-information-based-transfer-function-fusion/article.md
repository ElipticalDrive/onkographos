# Information-based Transfer Function Fusion

Also called: IBTF, Information-based Data Fusion, Information-based Transfer Function Classification, Information-based Transfer Functions for Multimodal Visualization

## Overview
Information-based Transfer Function Fusion (IBTF) is a method for designing transfer functions in multimodal volume visualization that reduces the complexity of multi-dimensional classification by fusing values from multiple registered modalities into a single scalar value using information theory [S1]. It solves the problem of combining complementary information from different modalities (e.g., CT and MR) into a single coherent visualization where both inputs are represented at each voxel, rather than discarding one [S1, 9731-information-theory-based-multimodal-transfer-function-design.md]. It sets itself apart from manual methods by automating the design and from single-modality optimization methods by defining an information channel between two input data sets and another between viewpoints and bins [S1, 9731-information-theory-based-multimodal-transfer-function-design.md].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two registered volume datasets $X$ and $Y$ with scalar values $f_1$ and $f_2$ at each voxel, and their respective gradient vectors $\nabla f_1$ and $\nabla f_2$.
*   **Outputs:** A fused scalar value $f_{fused}$, a fused gradient vector $\nabla f_{fused}$, and an "opposite information" metric $\delta$. These are used to look up color $c$ and modified opacity $\alpha_{mod}$ from transfer function lookup tables (LUTs).

**Data Structures:**
The method relies on four pre-computed 2D Lookup Tables (LUTs) generated from histograms of the input data:
1.  **$\gamma$ LUT:** Maps input value pairs $(f_1, f_2)$ to the weighting factor $\gamma$.
2.  **$\delta$ LUT:** Maps input value pairs $(f_1, f_2)$ to the opposite information metric $\delta$.
3.  **Transfer Function LUT:** Maps fused coordinates $(f_{fused}, \|\nabla f_{fused}\|)$ to optical properties $(c, \alpha)$.
4.  **Windowing Parameters LUT:** Maps fused coordinates $(f_{fused}, \|\nabla f_{fused}\|)$ to windowing parameters $(\delta_{pos}, \delta_{width})$.

**Steps:**

1.  **Pre-processing (Histogram Generation):**
    *   Generate individual histograms for both modalities to estimate marginal probability distributions $P(f_1)$ and $P(f_2)$.
    *   Generate a 2D "dual" histogram to estimate the joint probability distribution $P(f_1, f_2)$.
    *   Calculate $\gamma(f_1, f_2)$ and $\delta(f_1, f_2)$ for all value pairs in the histogram bins and store them in the $\gamma$ and $\delta$ LUTs respectively. This is done once before rendering [S1].

2.  **Probability Estimation:**
    *   Estimate the marginal probability $P(f)$ for each value $f$ by dividing the count of voxels in each histogram bin by the total number of points $N$ in the volume [S1, 9731-information-theory-based-multimodal-transfer-function-design.md].
    *   Estimate the joint probability $P(f_1, f_2)$ using the dual histogram counts divided by $N$ [S1].

3.  **Information Content Calculation:**
    *   Calculate the information content $I(f)$ for each value based on its frequency. Values occurring rarely have high information [S1].
    *   Formula:
        $$I(f) = -\log_2(P(f))$$
    *   If $P(f) = 0$, $I(f)$ is undefined; in practice, this bin is handled by the joint probability logic below or assigned a maximum information value depending on implementation, but the source defines PMI for $P(f_1, f_2)=0$ as 0 [S1].

4.  **Fusion Weighting ($\gamma$):**
    *   Calculate the weighting factor $\gamma$ which determines how much each modality contributes to the fused value based on its information content.
    *   Formula:
        $$\gamma(f_1, f_2) = \frac{I(f_2)}{I(f_1) + I(f_2)}$$
    *   Properties: $\gamma=0$ if modality 2 has no information ($I(f_2)=0$), $\gamma=1$ if modality 1 has no information ($I(f_1)=0$), and $\gamma=0.5$ if both have equal information [S1].

5.  **Value and Gradient Fusion:**
    *   Fuse the scalar values and gradient vectors using $\gamma$. This ensures the modality with higher information content has a stronger influence on the result [S1].
    *   Fused Value:
        $$f_{fused} = (1 - \gamma) \cdot f_1 + \gamma \cdot f_2$$
    *   Fused Gradient:
        $$\nabla f_{fused} = (1 - \gamma) \cdot \nabla f_1 + \gamma \cdot \nabla f_2$$

6.  **Opposite Information Calculation ($\delta$):**
    *   Calculate Pointwise Mutual Information (PMI) to measure the dependency between $f_1$ and $f_2$.
    *   Formula:
        $$PMI(f_1, f_2) = \log_2\left(\frac{P(f_1, f_2)}{P(f_1) \cdot P(f_2)}\right)$$
    *   Edge Case: If $P(f_1, f_2) = 0$, $PMI(f_1, f_2)$ is defined as 0 [S1].
    *   Normalize PMI to the range $[0, 1]$ to create $PMI_{norm}$. The source provides the following normalization formula:
        $$PMI_{norm}(f_1, f_2) = \frac{PMI(f_1, f_2) - \log_2(P(f_1, f_2))}{\log_2(P(f_1)) - \log_2(P(f_1, f_2))}$$
        *(Note: The source text contains a likely typo in the denominator description vs formula, but the formula provided is used as written above [S1]).*
    *   Calculate $\delta$ as the inversion of $PMI_{norm}$. $\delta$ approaches 0 for high information (correlated) pairs and close to 1 for low information (complementary/noise) pairs [S1].
    *   Formula:
        $$\delta(f_1, f_2) = 1 - PMI_{norm}(f_1, f_2)$$

7.  **Rendering Loop (Per Sample Point):**
    *   **Step 1:** Lookup $\gamma$ and $\delta$ from pre-computed LUTs using input values $f_1, f_2$.
    *   **Step 2:** Calculate $f_{fused}$ and $\nabla f_{fused}$ using $\gamma$ and the input gradients.
    *   **Step 3:** Lookup base color $c$ and base opacity $\alpha$ from the Transfer Function LUT using $(f_{fused}, \|\nabla f_{fused}\|)$.
    *   **Step 4:** Lookup windowing parameters $\delta_{pos}$ and $\delta_{width}$ from the Windowing Parameters LUT using $(f_{fused}, \|\nabla f_{fused}\|)$.
    *   **Step 5:** Calculate the windowing weight $\omega(\delta)$ using $\delta$, $\delta_{pos}$, and $\delta_{width}$.
    *   **Step 6:** Calculate modified opacity $\alpha_{mod} = \alpha \cdot \omega(\delta)$.
    *   **Step 7:** Use $c$ and $\alpha_{mod}$ for shading and compositing.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calculate_information(probability):
    """
    Calculates information content I(f) = -log2(P(f)).
    Handles P(f) = 0 by returning a large number or handling via joint prob logic.
    For this implementation, we assume P(f) > 0 for marginal probs in fusion weighting.
    """
    if probability <= 0:
        return float('inf')
    return -math.log2(probability)

def calculate_gamma(I_f1, I_f2):
    """
    Calculates fusion weight gamma.
    """
    if I_f1 == float('inf') and I_f2 == float('inf'):
        return 0.5
    if I_f2 == float('inf'):
        return 1.0
    if I_f1 == float('inf'):
        return 0.0
    return I_f2 / (I_f1 + I_f2)

def calculate_pmi(P_f1, P_f2, P_f1_f2):
    """
    Calculates Pointwise Mutual Information.
    If P(f1, f2) is 0, PMI is 0.
    """
    if P_f1_f2 == 0:
        return 0.0
    return math.log2(P_f1_f2 / (P_f1 * P_f2))

def calculate_pmi_norm(PMI, P_f1, P_f2, P_f1_f2):
    """
    Normalizes PMI to [0, 1].
    Formula from source: (PMI - log2(P(f1,f2))) / (log2(P(f1)) - log2(P(f1,f2)))
    Note: Source text has a typo in denominator description, formula uses P(f1).
    """
    if P_f1_f2 == 0:
        # PMI is 0, log2(0) is -inf. This case is handled by PMI=0 above.
        # If we reach here, P_f1_f2 > 0.
        pass
    
    log_joint = math.log2(P_f1_f2)
    log_marginal_f1 = math.log2(P_f1)
    
    # Avoid division by zero if P(f1) == P(f1, f2)
    denominator = log_marginal_f1 - log_joint
    if denominator == 0:
        return 0.0 # Or handle as edge case
        
    numerator = PMI - log_joint
    return numerator / denominator

def calculate_delta(PMI_norm):
    """
    Calculates opposite information delta.
    """
    return 1.0 - PMI_norm

def windowing_function(delta, delta_pos, delta_width):
    """
    Calculates the windowing weight omega(delta).
    """
    if delta_width == 0:
        return 0.0
    val = abs(delta - delta_pos) / (0.5 * delta_width)
    return max(1.0 - val, 0.0)

def render_sample_point(f1, f2, grad1, grad2, 
                        hist_marginal_f1, hist_marginal_f2, hist_joint, 
                        N, 
                        tf_lut, windowing_lut):
    """
    Renders a single sample point using IBTF.
    hist_marginal_f1: dict mapping bin_index -> count
    hist_marginal_f2: dict mapping bin_index -> count
    hist_joint: dict mapping (bin_idx_f1, bin_idx_f2) -> count
    N: total number of voxels
    """
    
    # 1. Get bin indices for f1 and f2 (implementation specific binning)
    bin_f1 = get_bin_index(f1)
    bin_f2 = get_bin_index(f2)
    
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
    PMI_norm = calculate_pmi_norm(PMI, P_f1, P_f2, P_f1_f2)
    delta = calculate_delta(PMI_norm)
    
    # 7. Lookup Optical Properties
    # Note: In practice, f_fused and norm_grad_fused are quantized to LUT indices
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
*   **Zero Joint Probability:** If $P(f_1, f_2) = 0$, PMI is defined as 0 by definition [S1]. The normalization formula must handle this to avoid division by zero or log(0) errors, typically by checking for zero probability before calculation [S1].
*   **Noise:** Noise flattens the probability distribution function but does not change the relation between probabilities, so the $\delta$ value is not affected by noise [S1].
*   **Gradient Fusion:** Gradients must be fused using the same $\gamma$ as the values to ensure shading and gradient-based classification use consistent information weighting [S1].

## Subtopics

### [Information Theory-Based Multimodal Transfer Function Design](9731-information-theory-based-multimodal-transfer-function-design.md)
An automated method for designing multimodal transfer functions for two registered volume datasets $X$ and $Y$, using mutual information decomposition to guide color fusion and opacity optimization. It solves the problem of combining complementary information from different modalities (e.g., CT and MR) into a single coherent visualization where both inputs are represented at each voxel, rather than discarding one. It sets itself apart from manual methods by automating the design and from single-modality optimization methods by defining an information channel between two input data sets and another between viewpoints and bins.

## Operating characteristics
*   **Time Complexity:** Pre-processing complexity is dominated by histogram generation, which is $O(N)$ for $N$ points. Rendering complexity per sample point is $O(1)$ due to LUT lookups and simple arithmetic [S1].
*   **Memory Complexity:** Requires storage for four 2D LUTs. The size of the LUTs depends on the number of bins in the histograms.
*   **Performance:** Enables real-time frame rates on graphics hardware because all processing steps (histogram generation, LUT creation) can be executed efficiently, and the rendering loop involves only fast lookups and arithmetic [S1].
*   **Strengths:** Reduces the complexity of multi-dimensional classification to a standard 2D transfer function space. Automates transfer function design. Provides better separation of different tissues by using the $\delta$ metric for complementary information. Robust to noise.
*   **Limitations:** Requires registered volume datasets. The quality of the fusion depends on the accuracy of the histogram estimation. The normalization formula for PMI contains a potential ambiguity/typo in the source literature [S1].
*   **When to use:** Multimodal visualization where complementary information from different modalities needs to be combined automatically without manual tuning.
*   **When not to use:** Single-modality visualization (simpler methods suffice) or when manual control over the fusion weighting is preferred over information-theoretic automation.

## Background
IBTF was introduced by Haidacher et al. in 2008 [S1]. It is based on methods from information theory, specifically using Shannon's information content and Pointwise Mutual Information (PMI) to guide the fusion of multimodal data. The method addresses the challenge of designing transfer functions for high-dimensional parameter spaces (e.g., value, gradient, and multiple modalities) by fusing them into a simpler 2D space (fused value and fused gradient magnitude) augmented with an opposite information metric [S1].

## Sources
[S1] Information-based Transfer Functions for Multimodal Visualization, https://doi.org/10.2312/vcbm/vcbm08/101-108, fetched 2026-10-26
[S2] Information Theory-Based Automatic Multimodal Transfer Function Design, https://doi.org/10.1109/jbhi.2013.2263227, fetched 2026-10-27
