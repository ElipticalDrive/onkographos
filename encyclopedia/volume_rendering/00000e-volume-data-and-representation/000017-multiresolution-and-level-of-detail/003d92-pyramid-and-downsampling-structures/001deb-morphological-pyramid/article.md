# Morphological Pyramid

Also called: Adjunction Pyramid, Erosion/Dilation Pyramid, Morphological Adjunction Pyramid, Morphological Pyramid Construction, Pyramid Synthesis for MIP

## Overview
The Morphological Pyramid is a multiresolution representation scheme for volume data that uses morphological erosion and dilation combined with dyadic downsampling and upsampling to create a hierarchy of approximations and detail signals [S1, S2]. It enables perfect reconstruction of the original data and improves approximation accuracy when rendering from higher pyramid levels [S1, S2]. Unlike linear pyramid-based methods that rely on additive reconstruction, the Morphological Pyramid exploits the commutativity of the maximum intensity projection (MIP) operator with morphological dilation and upsampling, allowing projections to be performed on coarse levels and expanded in 2-D [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset $f_0$ defined on a discrete grid domain $F \subseteq \mathbb{Z}^d$ (where $d=3$ for volumes) taking values in a finite set of nonnegative integers [S1, S2]. A structuring element $A \subseteq \mathbb{Z}^d$ [S2].
*   **Output:** An approximation pyramid $\{f_0, f_1, \dots, f_L\}$ and a detail pyramid $\{d_0, d_1, \dots, d_{L-1}, f_L\}$, where $L$ is the decomposition depth [S1].

**Data Structures:**
*   **Approximation Pyramid:** A set of signals $\{f_j\}_{j=0}^L$ where each $f_j$ has increasingly reduced size [S1]. Each level $j$ reduces the spatial resolution by a factor of $2^j$ in each dimension compared to $f_0$ [S2].
*   **Detail Pyramid:** A set of signals $\{d_j\}_{j=0}^{L-1}$ and the coarsest approximation $f_L$ [S1]. Each detail signal $d_j$ has the same spatial resolution as its corresponding approximation $f_j$ [S2].

**Morphological Operators:**
Let $f$ be a signal with domain $F$ and $A$ be the structuring element [S1, S2].
*   **Dilation:** $\delta_A(f)(x) = \max_{y \in A, x-y \in F} f(x-y)$ [S1, S2].
*   **Erosion:** $\epsilon_A(f)(x) = \min_{y \in A, x+y \in F} f(x+y)$ [S1, S2].
*   **Opening:** $\alpha_A(f)(x) = \delta_A(\epsilon_A(f))(x)$ [S1, S2].

**Pyramid Operators:**
*   **REDUCE (Analysis):** $\psi_A^\uparrow(f) = \text{DOWNSAMPLE}(\epsilon_A(f))$ for Adjunction Pyramids [S2]. For Sun-Maragos Pyramids, $\psi_A^\uparrow(f) = \text{DOWNSAMPLE}(\alpha_A(f))$ [S1].
*   **EXPAND (Synthesis):** $\psi_A^\downarrow(f) = \delta_A(\text{UPSAMPLE}(f))$ [S2].
    *   In MIP rendering contexts, a 2-D variant $\psi_{\tilde{A}}^\downarrow$ is used, where $\tilde{A}$ is the MIP of $A$ (i.e., $\tilde{A} := M_\Theta(A)$) [S2].

**Generalized Subtraction:**
To ensure non-negative detail signals, the generalized subtraction operator $\dot{-}$ is defined as [S1, S2]:
$$ t \dot{-} s = \begin{cases} t, & \text{if } t > s \\ 0, & \text{if } t = s \end{cases} $$
where $0$ represents the smallest possible image or voxel value [S1, S2].

**Parameters:**
*   **Structuring Element $A$:** A subset of $\mathbb{Z}^d$ defining the local neighborhood for erosion and dilation [S1, S2]. It must satisfy the **pyramid condition**: there exists an $a \in A$ such that the translates of $a$ over an even number of grid steps are never contained in $A$ [S1, S2].
*   **Decomposition Depth $L$:** The number of levels in the pyramid [S1].

**Algorithm Steps:**
1.  **Initialization:** Set $f_0$ as the input volume [S2].
2.  **Analysis (Decomposition):** For $j = 0$ to $L-1$:
    a. Compute the approximation at level $j+1$: $f_{j+1} = \text{REDUCE}(f_j)$ [S2].
    b. Compute the expanded version of the coarser approximation: $\hat{f}_{j+1} = \text{EXPAND}(f_{j+1})$ [S2].
    c. Compute the detail signal: $d_j = f_j \dot{-} \hat{f}_{j+1}$ [S2].
3.  **Reconstruction (Synthesis):** To reconstruct $f_0$ from $\{d_k\}$ and $f_L$:
    $$ f = \psi_A^{\downarrow L}(f_L) \vee \bigvee_{k=0}^{L-1} \psi_A^{\downarrow k}(d_k) $$
    where $\vee$ denotes the maximum operation [S1]. This relies on the generalized addition being defined as $t \dot{+} s = t \vee s = \max(t, s)$ [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def morphological_pyramid_decomposition(volume, structuring_element_A, depth_L):
    """
    Constructs a morphological pyramid from a volume.
    Args:
        volume: 3D array f_0.
        structuring_element_A: Set of coordinate offsets defining A.
        depth_L: Number of pyramid levels L.
    Returns:
        approximations: List [f_0, f_1, ..., f_L]
        details: List [d_0, d_1, ..., d_{L-1}]
    """
    approximations = [volume]
    details = []
    current_f = volume
    
    for j in range(depth_L):
        # 1. Erosion: min over neighborhood A
        eroded = morphological_erosion(current_f, structuring_element_A)
        
        # 2. Downsampling: keep every other voxel in each dimension
        f_next = downsample(eroded)
        approximations.append(f_next)
        
        # 3. Upsampling: insert zeros at intermediate grid points
        upscaled = upsample(f_next)
        
        # 4. Dilation: max over neighborhood A
        expanded = morphological_dilation(upscaled, structuring_element_A)
        
        # 5. Generalized Subtraction: d_j = f_j dot-minus expanded
        d_j = generalized_subtraction(current_f, expanded)
        details.append(d_j)
        
        current_f = f_next

    return approximations, details

def morphological_pyramid_reconstruction(approximations, details, structuring_element_A):
    """
    Reconstructs the volume from the pyramid using the maximum-based formula.
    Args:
        approximations: List [f_0, ..., f_L]
        details: List [d_0, ..., d_{L-1}]
        structuring_element_A: Structuring element for dilation.
    Returns:
        reconstructed_volume: 3D array f_0
    """
    L = len(approximations) - 1
    f_L = approximations[L]
    
    # Start with the coarsest approximation expanded back to full resolution
    reconstructed = expand_n_times(f_L, L, structuring_element_A)
    
    # Combine with expanded detail signals using maximum
    for k in range(L):
        d_k_expanded = expand_n_times(details[k], k, structuring_element_A)
        reconstructed = max(reconstructed, d_k_expanded)
        
    return reconstructed

def morphological_erosion(volume, A):
    """Computes erosion: min_{y in A, x+y in F} f(x+y)."""
    shape = volume.shape
    result = volume.copy()
    for x in range(shape[0]):
        for y in range(shape[1]):
            for z in range(shape[2]):
                min_val = float('inf')
                for dy, dz, dw in A:
                    nx, ny, nz = x + dy, y + dz, z + dw
                    if 0 <= nx < shape[0] and 0 <= ny < shape[1] and 0 <= nz < shape[2]:
                        min_val = min(min_val, volume[nx, ny, nz])
                result[x, y, z] = min_val if min_val != float('inf') else volume[x, y, z]
    return result

def morphological_dilation(volume, A):
    """Computes dilation: max_{y in A, x-y in F} f(x-y)."""
    shape = volume.shape
    result = volume.copy()
    for x in range(shape[0]):
        for y in range(shape[1]):
            for z in range(shape[2]):
                max_val = float('-inf')
                for dy, dz, dw in A:
                    nx, ny, nz = x - dy, y - dz, z - dw
                    if 0 <= nx < shape[0] and 0 <= ny < shape[1] and 0 <= nz < shape[2]:
                        max_val = max(max_val, volume[nx, ny, nz])
                result[x, y, z] = max_val if max_val != float('-inf') else volume[x, y, z]
    return result

def downsample(volume):
    """Dyadic downsampling by factor of 2 in each dimension."""
    return volume[::2, ::2, ::2]

def upsample(volume):
    """Dyadic upsampling by factor of 2 in each dimension, padding with zeros."""
    new_shape = (volume.shape[0] * 2, volume.shape[1] * 2, volume.shape[2] * 2)
    result = volume.zeros(new_shape)
    result[::2, ::2, ::2] = volume
    return result

def generalized_subtraction(t, s):
    """Computes t dot-minus s: t if t > s else 0."""
    result = t.copy()
    result[t <= s] = 0
    return result

def expand_n_times(f, n, A):
    """Applies EXPAND operator n times. EXPAND(f) = dilation(upsample(f))."""
    current = f
    for _ in range(n):
        upscaled = upsample(current)
        current = morphological_dilation(upscaled, A)
    return current
```

**Edge Cases and Pitfalls:**
*   **Pyramid Condition:** If the structuring element $A$ does not satisfy the pyramid condition (existence of $a \in A$ such that translates over even grid steps are never contained in $A$), perfect reconstruction may fail [S1, S2].
*   **Detail Signal Size:** In Adjunction Pyramids, too few small features are retained in higher levels, meaning the detail signals are 'too large' [S1].
*   **Precision:** The generalized subtraction operator defines the result as $0$ when $t = s$, where $0$ is the smallest image or voxel value possible [S1, S2].

## Subtopics

### [Multiresolution Maximum Intensity Projection (MMIP)](1745-multiresolution-maximum-intensity-projection-mmip.md)
Multiresolution Maximum Intensity Projection (MMIP) is a progressive rendering algorithm for 3-D volume data that computes the maximum value along lines through the data set. It uses a morphological pyramid to provide coarse approximations and progressively refines the image with detail coefficients. Unlike linear pyramid-based MIP methods which rely on additive reconstruction, MMIP exploits the commutativity of the MIP operator with morphological dilation and upsampling, allowing the projection to be performed on coarse levels and expanded in 2-D, thereby avoiding expensive 3-D reconstruction for certain pyramid types.

### Sun-Maragos Pyramid
The Sun-Maragos Pyramid is an alternative morphological pyramid scheme for MIP volume rendering where a morphological opening is used for pyramidal analysis instead of erosion, while the synthesis operator remains a dilation [S1]. The opening is defined as $\alpha_A(f)(x) = \delta_A(\epsilon_A(f))(x)$ [S1, S2]. This variant improves approximation accuracy when rendering from higher levels of the pyramid compared to the standard Adjunction Pyramid [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** The method scales with resolution due to the multiresolution nature, allowing coarse levels to be processed first [S1, S2].
*   **Strengths:** Perfect reconstruction property; commutativity with MIP allows efficient 2-D projection; progressive refinement [S1, S2].
*   **Limitations:** Detail signals can be large in Adjunction Pyramids; requires careful selection of structuring element to satisfy pyramid condition [S1, S2].
*   **When to use:** Interactive MIP volume rendering where progressive refinement is beneficial [S1, S2].
*   **When not to use:** When small features are critical and Adjunction Pyramids are used without the Sun-Maragos variant; when memory is extremely constrained [S1].

## Background
The Morphological Pyramid was introduced in the context of multiresolution algorithms for Maximum Intensity Projection (MIP) volume rendering [S1, S2]. It is widely used in the display of magnetic resonance angiography (MRA) and ultrasound data due to its computational simplicity [S1]. The concept of adjunction pyramids was introduced by Roefer [S2]. The Sun-Maragos variant was studied by Sun and Maragos and referenced in subsequent morphological image processing literature [S1]. The method is categorized under Interaction techniques (I.3.6) and Image Representation, Hierarchical, Morphological (I.4.10) [S1].

## Sources
[S1] Comparison of Morphological Pyramids for Multiresolution MIP Volume Rendering, https://doi.org/10.2312/vissym/vissym02/061-070, fetched 2026-10-29
[S2] Multiresolution MIP Rendering of Large Volumetric Data Accelerated on Graphics Hardware, https://doi.org/10.2312/vissym/eurovis07/243-250, fetched 2026-10-26
