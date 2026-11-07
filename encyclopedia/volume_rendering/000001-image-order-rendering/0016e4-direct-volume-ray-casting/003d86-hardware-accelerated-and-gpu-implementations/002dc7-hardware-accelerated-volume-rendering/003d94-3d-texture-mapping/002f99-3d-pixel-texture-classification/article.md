# 3D Pixel Texture Classification

Also called: VGH Lookup

## Overview
3D Pixel Texture Classification is a hardware-accelerated method for applying multi-dimensional transfer functions during direct volume ray casting by encoding classification parameters into a 3D texture lookup [S1]. It solves the problem of high-dimensional transfer function evaluation by mapping scalar volume properties (Data Value, Gradient Magnitude, and Second Derivative) to color and opacity via a single texture fetch [S1]. It differs from standard 1D or 2D lookup tables by explicitly incorporating boundary information (via the second derivative) into the hardware interpolation pipeline, allowing for precise control over material boundaries without requiring complex shader logic [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume dataset $V$. For each sample point $\mathbf{p}$ along a ray, the method requires three scalar values: the Data Value $D = V(\mathbf{p})$, the Gradient Magnitude $G = \|\nabla V(\mathbf{p})\|$, and the Second Derivative $S = \nabla^2 V(\mathbf{p})$ [S1].
*   **Output:** A color $C$ and opacity $\alpha$ pair for the current sample, retrieved from the 3D pixel texture [S1].

**Data Structures:**
The classification is stored in a 3D texture $T_{VGH}$ with dimensions $N_D \times N_G \times N_S$.
*   **Axis 1 (Data Value):** Resolution $N_D = 256$ (8-bit). Maps directly to the scalar voxel value [S1].
*   **Axis 2 (Gradient Magnitude):** Resolution $N_G = 256$ (8-bit). Maps to the magnitude of the gradient at the sample point [S1].
*   **Axis 3 (Second Derivative):** Resolution $N_S$ is limited (not 256). This axis represents the sign and magnitude of the second derivative [S1].

**Preprocessing and Coordinate Mapping:**
1.  **Data Value ($u$):** The scalar value $D$ is normalized to $[0, 1]$ and used directly as the $u$-coordinate [S1].
2.  **Gradient Magnitude ($v$):** The gradient magnitude $G$ is normalized to $[0, 1]$ and used as the $v$-coordinate [S1].
3.  **Second Derivative ($w$):** The second derivative $S$ is signed. To maintain the sign during hardware interpolation, a scale and bias are applied to map $S$ to the texture coordinate $w \in [0, 1]$ [S1]. The source notes that the sign must be maintained to allow proper interpolation [S1].

**Classification Logic (The "Sheets" Model):**
The third axis ($w$) is not a continuous gradient but is structured as discrete "sheets" representing specific second derivative behaviors [S1].
*   **Control Points:** Three control points are defined: Negative ($S < 0$), Zero ($S = 0$), and Positive ($S > 0$) [S1].
*   **Opacity Control:**
    *   The opacity at the center sheet ($S=0$) is directly controlled by classification widgets [S1].
    *   The opacity at the outer sheets ($S \neq 0$) is scaled from the center sheet opacity based on a "boundary emphasis" slider [S1].
    *   This allows linear control over the opacity of second derivative values [S1].

**Algorithm Steps:**
1.  Compute $D = V(\mathbf{p})$, $G = \|\nabla V(\mathbf{p})\|$, and $S = \nabla^2 V(\mathbf{p})$ for the current sample [S1].
2.  Normalize $D$ and $G$ to $[0, 1]$ [S1].
3.  Apply scale and bias to $S$ to preserve sign and map to $w \in [0, 1]$ [S1].
4.  Perform a trilinear texture lookup $T_{VGH}(u=D, v=G, w=w)$ [S1].
5.  The texture returns the pre-computed color $C$ and opacity $\alpha$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def classify_vgh(data_value, gradient_magnitude, second_derivative, texture_vgh, boundary_emphasis):
    # 1. Normalize Data Value and Gradient Magnitude to [0, 1]
    u = normalize(data_value, min=0, max=255) # Assuming 8-bit data
    v = normalize(gradient_magnitude, min=0, max=1) # Normalized gradient
    
    # 2. Map Second Derivative to w-coordinate
    # Scale and bias to maintain sign and fit in [0, 1]
    # Note: Exact scale/bias factors depend on implementation, 
    # but must preserve sign for interpolation [S1]
    w = map_second_derivative_to_w(second_derivative)
    
    # 3. Texture Lookup
    # The texture contains RGB and Alpha channels
    color, alpha = texture_vgh.sample(u, v, w)
    
    # 4. Apply Boundary Emphasis Scaling
    # The opacity on outer sheets (non-zero 2nd deriv) is scaled 
    # from the center sheet opacity according to the boundary emphasis slider [S1]
    # This scaling is typically baked into the texture or applied post-fetch.
    # If baked: the texture values already reflect the emphasis.
    # If applied:
    alpha = alpha * boundary_emphasis_factor
    
    return color, alpha
```

**Edge Cases and Pitfalls:**
*   **Precision:** Using full 8-bit resolution for all three axes would be prohibitively large ($256^3$ entries) [S1]. Limiting the second derivative axis resolution is necessary [S1].
*   **Sign Preservation:** Failing to maintain the sign of the second derivative via scale and bias will result in incorrect interpolation across the zero-crossing [S1].
*   **Boundary Emphasis:** The "boundary emphasis" slider affects the opacity of the outer sheets relative to the center sheet. If a global boundary emphasis is desired, it must be applied consistently [S1].

## Subtopics

### Adaptive Pre-integrated Voxel Classification
A technique included in a real-time raycaster built by Roettger et al. to exploit the intrinsically parallel nature of GPUs, along with volume clipping [S2]. It addresses the inefficiency of multi-pass rendering pipelines by using a single-pass framework [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample for classification, as it reduces to a single texture fetch [S1].
*   **Memory Complexity:** The 3D pixel texture size is $N_D \times N_G \times N_S \times 4$ bytes (assuming RGBA). With $N_D=256, N_G=256$, and limited $N_S$, memory usage is significantly lower than a full 3D lookup table [S1].
*   **Performance:** Hardware-accelerated via the GPU's texture engine, avoiding manual interpolation in shader code [S1].
*   **Strengths:** Efficient multi-dimensional classification; hardware interpolation handles the "sheets" model effectively [S1].
*   **Limitations:** Limited resolution on the second derivative axis; requires careful tuning of the boundary emphasis slider [S1].

## Background
The method was introduced in the context of interactive volume rendering using multi-dimensional transfer functions [S1]. It addresses the computational cost of evaluating complex transfer functions by leveraging the GPU's texture hardware [S1]. The approach was developed to enable real-time interaction with volumetric data by reducing the classification step to a simple lookup [S1].

## Sources
[S1] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
