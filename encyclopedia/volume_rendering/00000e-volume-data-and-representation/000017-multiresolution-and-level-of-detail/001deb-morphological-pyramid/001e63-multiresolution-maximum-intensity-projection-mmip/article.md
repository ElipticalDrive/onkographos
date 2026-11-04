# Multiresolution Maximum Intensity Projection (MMIP)

Also called: MIP Volume Rendering with Morphological Pyramids, Multiresolution MIP, Multiresolution MIP based on detail pyramid, Progressive refinement from detail pyramid, level-by-level MMIP

## Overview
Multiresolution Maximum Intensity Projection (MMIP) is a progressive rendering algorithm for 3-D volume data that computes the maximum value along lines through the data set [S1, S2, unofficial]. It solves the problem of interactive rendering latency by using a morphological pyramid to provide coarse approximations during user interaction and progressively refining the image with detail coefficients once interaction ceases [S1, unofficial]. Unlike linear pyramid-based MIP methods which rely on additive reconstruction, MMIP exploits the commutativity of the MIP operator with morphological dilation and upsampling, allowing the projection to be performed on coarse levels and expanded in 2-D, thereby avoiding expensive 3-D reconstruction for certain pyramid types [S1, S2, unofficial].

## How it works
**Inputs:**
1.  A detail pyramid sequence $d_0, d_1, \dots, d_{L-1}$ and the coarsest approximation $f_L$ of the input volume data set $f$.
2.  A viewing orientation $\Theta$, defined by angles $\theta, \phi$ (projection direction perpendicular to the view plane) and $\alpha$ (orientation of the view plane) [S1, S2, unofficial].
3.  A structuring element $A$ for the morphological operations [S1, S2, unofficial].

**Outputs:**
A sequence of progressively refined 2-D approximation images $M_\Theta^{(j)}(f)$ for $j = L, L-1, \dots, 0$ [S1, S2, unofficial].

**Data Structures:**
*   **Pyramid Levels:** The volume is decomposed into levels $j=0 \dots L$. Level $L$ is the coarsest approximation $f_L$. Levels $0 \dots L-1$ contain detail signals $d_j$.
*   **Storage:** For efficient voxel projection, 'interesting' voxels are sorted by grey value using a histogram-based scheme. This involves a value-sorted array of voxel positions and an additional array containing cumulative histogram values [S1, unofficial].
*   **Structuring Element:** The 2-D expand operator uses a structuring element $\tilde{A}$, defined as the MIP of the original 3-D structuring element $A$: $\tilde{A} := M_\Theta(A)$ [S2, unofficial].

**Definitions:**
*   **MIP Operator ($M_\Theta$):** Computes the maximum value along rays defined by $\Theta$.
*   **Upsample ($\text{UPSAMPLE}$):** Increases resolution by a factor of 2 in each spatial dimension, inserting zeros or interpolating (standard definition implied by context of dilation) [S1, S2, unofficial].
*   **Dilation ($\delta_A$):** $\delta_A(f)(x) = \max_{a \in A} (f(x-a))$ [S1, S2, unofficial].
*   **2-D Expand Operator ($\tilde{\psi}_A^\downarrow$):** $\tilde{\psi}_A^\downarrow(f) = \delta_{\tilde{A}}(\text{UPSAMPLE}(f))$ [S2, unofficial].
*   **Generalized Subtraction (Adjunction only):** $t \dot{-} s = \begin{cases} t, & \text{if } t > s \\ 0, & \text{if } t = s \end{cases}$, where 0 is the smallest possible voxel value [S2, unofficial].

**Algorithm Steps:**
1.  **Pyramid Construction:** Recursively apply the REDUCE operator to compute approximations $f_j$ and detail signals $d_j$ until level $L$ is reached. The original volume data can be discarded after construction due to the perfect reconstruction property [S1, unofficial].
    *   For Adjunction Pyramids: $\Psi_A^\downarrow(f) = \text{DOWNSAMPLE}(\epsilon_A(f))$ and $d_j = f_j \dot{-} \Psi_A^\downarrow \Psi_A^\uparrow (f_j)$ [S2, unofficial].
    *   For Sun-Maragos Pyramids: $\Psi_A^\downarrow(f) = \text{DOWNSAMPLE}(\alpha_A(f))$ [S1, unofficial].
2.  **Coarsest Approximation:** Compute the MIP of the coarsest volume level $f_L$, then expand it to the full image resolution using the 2-D expand operator applied $L$ times:
    $$M_\Theta^{(L)}(f) = \tilde{\psi}_A^{\downarrow L}(M_\Theta(f_L))$$
    where $\tilde{\psi}_A^{\downarrow L}$ denotes $L$-fold composition of $\tilde{\psi}_A^\downarrow$ [S1, S2, unofficial].
3.  **Progressive Refinement:** Iterate from $j = L$ down to $1$ to refine the image:
    *   **For Adjunction Pyramids (2D-only refinement):**
        Compute the next approximation by taking the maximum of the expanded MIP of the detail signal and the previous approximation:
        $$M_\Theta^{(j-1)}(f) = \tilde{\psi}_A^{\downarrow (j-1)}(M_\Theta(d_{j-1})) \vee M_\Theta^{(j)}(f)$$
        Since detail signals $d_j$ are non-negative, the projections increase pointwise: $M_\Theta^{(j-1)}(f) \ge M_\Theta^{(j)}(f)$ [S2, unofficial].
    *   **For Sun-Maragos Pyramids (3D reconstruction required):**
        Reconstruct the volume at level $j-1$ from level $j$ and the detail $d_{j-1}$, then project and expand:
        $$f_{j-1} = \psi_A^\uparrow(f_j) \dot{+} d_{j-1}$$
        $$M_\Theta^{(j-1)}(f) = \tilde{\psi}_A^{\downarrow (j-1)}(M_\Theta(f_{j-1}))$$
        This requires a 3-D dilation for reconstruction, which is less efficient for value-sorted arrays [S1, unofficial].

**Pseudocode (Adjunction Pyramid Variant):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mmip_adjunction(f_L, details, L, theta, structuring_element_A):
    """
    Perform Multiresolution MIP using an adjunction pyramid.
    
    Args:
        f_L: Coarsest approximation volume (level L).
        details: List of detail volumes [d_0, d_1, ..., d_{L-1}].
        L: Decomposition depth.
        theta: Viewing orientation parameters.
        structuring_element_A: 3D structuring element for morphological ops.
        
    Returns:
        List of 2D images [M_L, M_{L-1}, ..., M_0] representing progressive refinement.
    """
    # Precompute 2D structuring element for expansion
    # M_theta(A) computes the MIP of the structuring element A along theta
    structuring_element_A_2d = mip_2d(structuring_element_A, theta)
    
    # Step 1: Coarsest approximation
    # Project coarse volume
    mip_coarse = mip_3d(f_L, theta)
    # Expand L times to full resolution
    current_image = mip_coarse
    for _ in range(L):
        current_image = expand_2d(current_image, structuring_element_A_2d)
        
    results = [current_image]
    
    # Step 2: Progressive refinement from L down to 1
    # details[j-1] corresponds to d_{j-1}
    for j in range(L, 0, -1):
        detail_volume = details[j-1]
        
        # Project detail volume
        mip_detail = mip_3d(detail_volume, theta)
        
        # Expand (j-1) times
        expanded_detail = mip_detail
        for _ in range(j-1):
            expanded_detail = expand_2d(expanded_detail, structuring_element_A_2d)
            
        # Take maximum with previous approximation
        # Note: In adjunction pyramids, details are non-negative, 
        # so this effectively adds detail without subtracting.
        current_image = max(current_image, expanded_detail)
        
        results.append(current_image)
        
    return results

def expand_2d(image, structuring_element_2d):
    """
    Perform 2D expansion: Upsample by 2 in each dimension, then dilate.
    """
    upscaled = upsample_2d(image, factor=2)
    dilated = dilate_2d(upscaled, structuring_element_2d)
    return dilated

def upsample_2d(image, factor):
    """
    Upsample image by inserting zeros/interpolating.
    Standard definition: new_image[x, y] = image[x/factor, y/factor] if divisible, else 0.
    """
    new_h, new_w = image.height * factor, image.width * factor
    new_image = ZeroImage(new_h, new_w)
    for y in range(image.height):
        for x in range(image.width):
            new_image[y * factor, x * factor] = image[y, x]
    return new_image

def dilate_2d(image, structuring_element):
    """
    2D Morphological Dilation.
    """
    result = ZeroImage(image.height, image.width)
    for y in range(image.height):
        for x in range(image.width):
            max_val = 0 # Assuming 0 is min value
            for dy, dx in structuring_element:
                ny, nx = y + dy, x + dx
                if 0 <= ny < image.height and 0 <= nx < image.width:
                    max_val = max(max_val, image[ny, nx])
            result[y, x] = max_val
    return result
```

## Subtopics

### [Multiresolution MIP based on adjunction pyramid](1898-multiresolution-mip-based-on-adjunction-pyramid.md)
This method performs progressive Maximum Intensity Projection (MIP) volume rendering by refining the image directly in the 2D plane using an adjunction pyramid, avoiding 3D volume reconstruction. It leverages the commutativity of MIP and dilation to update the projection by taking the maximum of the expanded detail projection and the previous approximation. Unlike general detail pyramids that require 3D reconstruction, this variant exploits the specific properties of adjunction pyramids where detail signals are non-negative, allowing for simpler, purely 2D progressive refinement.

## Operating characteristics
*   **Time Complexity:** The complexity depends on the pyramid type. For adjunction pyramids, the refinement is purely 2D, making it significantly faster as it avoids 3D reconstruction. For Sun-Maragos pyramids, each refinement step requires a 3D dilation for reconstruction, which is computationally expensive, especially with value-sorted arrays [S1, unofficial].
*   **Memory Complexity:** Requires storage for the pyramid levels ($f_L$ and $d_0 \dots d_{L-1}$). The original volume can be discarded after pyramid construction [S1, unofficial].
*   **Performance:** Interactive rendering is fast because only coarse levels are used during interaction. Progressive refinement adds detail incrementally. The adjunction variant is more efficient for streaming or interactive use due to the 2D-only update [S2, unofficial].
*   **Strengths:** Perfect reconstruction capability; compatibility with MIP due to commutativity of dilation/upsampling with maximum; reduced data size for interaction; integer computations only (for flat pyramids) [S1, unofficial].
*   **Limitations:** Image quality is lower in preview mode due to loss of detail; Sun-Maragos variant is inefficient for certain data structures due to 3D reconstruction requirements [S1, unofficial].
*   **When to use:** Interactive visualization of large volumetric datasets (e.g., MRA, ultrasound) where user interaction speed is critical and progressive detail addition is acceptable [S1, unofficial].
*   **When not to use:** When high-fidelity static rendering is required without progressive refinement, or when the overhead of 3D reconstruction for Sun-Maragos pyramids is prohibitive [S1, unofficial].

## Background
MMIP was introduced in the context of multiresolution algorithms for Maximum Intensity Projection volume rendering, widely used in magnetic resonance angiography (MRA) and ultrasound data display [S1, unofficial]. The method builds on the properties of morphological pyramids, specifically the commutativity of morphological operators with the MIP operator [S1, unofficial]. The adjunction pyramid variant was detailed in works by Roe et al. [S2, unofficial], while the Sun-Maragos pyramid variant was discussed in comparison studies [S1, unofficial]. The approach addresses the major problem of data size in interactive rendering by allowing coarse previews and progressive refinement [S1, unofficial].

## Sources
[S1] Comparison of Morphological Pyramids for Multiresolution MIP Volume Rendering, https://doi.org/10.2312/vissym/vissym02/061-070, fetched 2026-10-29 (unofficial)
[S2] Multiresolution MIP Rendering of Large Volumetric Data Accelerated on Graphics Hardware, https://doi.org/10.2312/vissym/eurovis07/243-250, fetched 2026-10-26 (unofficial)
