# CT/PET Modality Fusion via Pseudo-HU Mapping

Also called: CT-PET Signal Integration Macro, Multi-modality VR-prep, PET-CT alignment macro, Pseudo-HU Fusion, intensity override macro

## Overview
This method aligns independently acquired CT and PET image stacks voxel-wise and merges their intensity values into a single dataset using a continuous "pseudo-Hounsfield Unit" (pseudo-HU) scale [S1]. It solves the problem of visualizing joint structural (CT) and functional (PET) information in a single volume for Augmented Reality (AR) display, extending the MIXR application's native CT-only support to PET-CT data [S1]. It sets itself apart from simple overlay techniques by creating a single scalar field where PET signal intensity overrides CT values above a specific threshold, mapping both modalities into a unified 0–999 integer range [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Two aligned TIFF image stacks: one containing CT Hounsfield Units (HU) and one containing PET activity (typically in Bq/mL or standardized uptake values).
*   **Output:** A single TIFF image stack containing pseudo-HU values (integers 0–999) [S1].

**Data Structures:**
*   **CT Stack:** 3D array of floating-point or integer HU values.
*   **PET Stack:** 3D array of PET signal intensity values (e.g., Bq/mL).
*   **Output Stack:** 3D array of 16-bit unsigned integers (0–999) [S1].

**Parameters:**
*   $CT_{min} = -1000$ HU, $CT_{max} = +1000$ HU [S1].
*   $pseudo_{CT\_min} = 0$, $pseudo_{CT\_max} = 499$ [S1].
*   $pseudo_{PET\_min} = 500$, $pseudo_{PET\_max} = 999$ [S1].
*   $PET_{ref} = 100$ Bq/mL (maps to 999 pseudo-HU) [S1].
*   $Threshold$: A predefined PET intensity cutoff value. If PET signal < $Threshold$, CT value is used. If PET signal $\ge$ $Threshold$, PET value overrides CT [S1]. The typical value is not specified in the sources [S1].

**Steps:**

1.  **Preprocessing:** Process CT and PET images independently through the VR-prep pipeline in Fiji to generate two .tiff stacks [S1].
2.  **Alignment:** Align the two .tiff stacks to ensure spatial correspondence (voxel-wise matching) [S1].
3.  **CT Mapping:** For each voxel, map the original CT HU value to the pseudo-HU range [0, 499] using linear scaling with clamping:
    $$
    pseudo\_HU_{CT} = \begin{cases} 
    0 & \text{if } HU_{CT} < -1000 \\
    \text{round}\left( \frac{HU_{CT} - (-1000)}{1000 - (-1000)} \times 499 \right) & \text{if } -1000 \le HU_{CT} \le 1000 \\
    499 & \text{if } HU_{CT} > 1000 
    \end{cases}
    $$
    [S1]
4.  **PET Mapping:** For each voxel, if the PET signal exceeds the $Threshold$, map the PET value to the pseudo-HU range [500, 999] linearly, assuming 0 Bq/mL (or threshold baseline) maps to 500 and 100 Bq/mL maps to 999. The exact lower bound of the PET linear scale (whether it starts at 0 or the threshold value) is described as "scaling linear to the individual set threshold (500 pseudo-HU)" [S1]. This implies the mapping is relative to the threshold or absolute. Given "100 Bq/mL ... defined as 999", and the range starts at 500, we assume a linear interpolation between the threshold value (mapped to 500) and 100 Bq/mL (mapped to 999). Let $PET_{val}$ be the PET intensity.
    $$
    pseudo\_HU_{PET} = \begin{cases} 
    500 & \text{if } PET_{val} \le Threshold \\
    \text{round}\left( 500 + \frac{PET_{val} - Threshold}{100 - Threshold} \times 499 \right) & \text{if } Threshold < PET_{val} < 100 \\
    999 & \text{if } PET_{val} \ge 100 
    \end{cases}
    $$
    *Note: The source states "scaling linear to the individual set threshold (500 pseudo-HU) to a standard-uptake-value of 100 Bq/mL (defined as 999 pseudo-HU)" [S1]. This phrasing suggests the linear segment connects the point $(Threshold, 500)$ to $(100, 999)$. Values below threshold are handled by the override logic, not this mapping.* [S1]
5.  **Fusion:** For each coordinate $(x, y, z)$:
    *   Initialize $final\_pseudo\_HU = pseudo\_HU_{CT}(x, y, z)$.
    *   If $PET(x, y, z) \ge Threshold$:
        *   $final\_pseudo\_HU = pseudo\_HU_{PET}(x, y, z)$.
    *   Else:
        *   $final\_pseudo\_HU = pseudo\_HU_{CT}(x, y, z)$.
    [S1]
6.  **Output:** Save the resulting 3D array as a single TIFF file [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fuse_ct_pet(ct_stack, pet_stack, threshold):
    """
    Fuses CT and PET stacks into a single pseudo-HU stack.
    
    Args:
        ct_stack: 3D numpy array of CT HU values.
        pet_stack: 3D numpy array of PET intensity values (Bq/mL).
        threshold: Float, the PET intensity cutoff.
        
    Returns:
        fused_stack: 3D numpy array of integers (0-999).
    """
    # Dimensions
    D, H, W = ct_stack.shape
    
    # Initialize output
    fused_stack = np.zeros((D, H, W), dtype=np.uint16)
    
    # Constants
    CT_MIN_HU = -1000.0
    CT_MAX_HU = 1000.0
    PSEUDO_CT_MIN = 0
    PSEUDO_CT_MAX = 499
    
    PSEUDO_PET_MIN = 500
    PSEUDO_PET_MAX = 999
    PET_REF_UPTAKE = 100.0  # Bq/mL
    
    for d in range(D):
        for h in range(H):
            for w in range(W):
                # 1. Map CT value
                hu_val = ct_stack[d, h, w]
                
                if hu_val < CT_MIN_HU:
                    pseudo_ct = PSEUDO_CT_MIN
                elif hu_val > CT_MAX_HU:
                    pseudo_ct = PSEUDO_CT_MAX
                else:
                    # Linear mapping: -1000 -> 0, 1000 -> 499
                    # Range is 2000 HU. Scale factor: 499 / 2000
                    pseudo_ct = int(round((hu_val - CT_MIN_HU) * (PSEUDO_CT_MAX - PSEUDO_CT_MIN) / (CT_MAX_HU - CT_MIN_HU)))
                
                # 2. Check PET value
                pet_val = pet_stack[d, h, w]
                
                if pet_val >= threshold:
                    # 3. Map PET value
                    # Linear mapping from (threshold, 500) to (100, 999)
                    # If pet_val > 100, cap at 999
                    if pet_val > PET_REF_UPTAKE:
                        pseudo_pet = PSEUDO_PET_MAX
                    else:
                        # Linear interpolation
                        # Slope = (999 - 500) / (100 - threshold)
                        # pseudo_pet = 500 + slope * (pet_val - threshold)
                        if threshold == PET_REF_UPTAKE:
                            # Avoid division by zero if threshold equals ref uptake
                            pseudo_pet = PSEUDO_PET_MIN
                        else:
                            pseudo_pet = int(round(PSEUDO_PET_MIN + (pet_val - threshold) * (PSEUDO_PET_MAX - PSEUDO_PET_MIN) / (PET_REF_UPTAKE - threshold)))
                    
                    # Override CT with PET
                    fused_stack[d, h, w] = pseudo_pet
                else:
                    # Retain CT
                    fused_stack[d, h, w] = pseudo_ct
                    
    return fused_stack
```

**Edge Cases and Pitfalls:**
*   **Threshold Selection:** The threshold value is critical but not specified numerically in the sources [S1]. An inappropriate threshold may lead to excessive PET override (noise) or insufficient override (missed lesions) [S1].
*   **PET Signal Below Threshold:** Voxels with PET signal below the threshold retain their CT-derived pseudo-HU value, which is clamped to [0, 499] [S1].
*   **PET Signal Above Reference:** PET values exceeding 100 Bq/mL are capped at 999 pseudo-HU [S1].
*   **CT Out of Range:** CT values outside [-1000, 1000] are clamped to 0 or 499 respectively [S1].
*   **Alignment Errors:** Misalignment between CT and PET stacks will result in incorrect fusion, as the method assumes voxel-wise correspondence [S1].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the total number of voxels in the volume, as each voxel is processed once [S1].
*   **Memory Complexity:** $O(N)$ to store the input CT stack, input PET stack, and output fused stack [S1].
*   **Performance:** The method is part of the VR-prep pipeline, which is designed to reduce file size and processing time for AR visualization on smartphones (e.g., iPad Pro) [S1]. Specific timing metrics for the fusion step alone are not provided, but the overall VR-prep pipeline was quantified for upload and download times to the MIXR app [S1].
*   **Strengths:** Combines structural and functional data into a single, lightweight 16-bit integer stack suitable for AR [S1]. Simple linear mappings are computationally inexpensive [S1].
*   **Limitations:** The method is rigid; it does not allow for dynamic adjustment of the fusion ratio during rendering [S1]. The threshold is static [S1].
*   **When to use:** When preparing PET-CT data for AR visualization in the MIXR application where a single scalar field is required [S1].
*   **When not to use:** When quantitative analysis of separate modalities is required simultaneously, or when the PET signal distribution is highly non-linear and requires more complex blending [S1].

## Background
This method was developed as part of the VR-prep workflow to enable the visualization of PET-CT data in Augmented Reality on smartphones, specifically for the 'Medical Imaging XR' (MIXR) application [S1]. It was introduced in the paper "VR-prep improves visualization of medical data in augmented reality on smartphones" [S1]. The method builds on the initial VR-prep macro that integrates intensity values into a single .tiff file [S1]. It was tested on patients with metastasized prostate cancer and kidney scans [S1].

## Sources
[S1] VR-prep improves visualization of medical data in augmented reality on smartphones., https://doi.org/10.1038/s41598-025-26856-7, fetched 2026-10-26
