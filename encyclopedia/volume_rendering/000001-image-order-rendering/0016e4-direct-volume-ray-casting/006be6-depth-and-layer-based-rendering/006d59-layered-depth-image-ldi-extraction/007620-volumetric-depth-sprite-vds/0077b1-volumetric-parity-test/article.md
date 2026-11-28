# Volumetric Parity Test

Also called: Ray-marching shader implementation

## Overview
The Volumetric Parity Test (VPT) is a real-time algorithm that classifies a 3D point as inside or outside a solid volume represented by a Volumetric Depth Sprite (VDS) [S1]. It solves the problem of pixel-precise clipping against multiple arbitrary-shaped meshes in a single rendering pass, avoiding the expensive multi-pass image-based Constructive Solid Geometry (CSG) approaches [S1]. Unlike its siblings, VPT performs this classification via a parity-swap ray march through the VDS's layered depth layers, enabling efficient implementation in programmable pipeline stages [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D point $P$ (in world or clip space), the VDS parameters (position $P_S$, model matrix $M_S$, depth complexity $d_S$, and the 3D texture $LDI_S$), and the current orientation matrix $O$ [S1].
*   **Output:** A boolean parity $p_T \in \{0, 1\}$, where 1 indicates the point is inside the volume and 0 indicates it is outside [S1].

**Data Structures:**
*   **VDS:** Defined by $(P_S, M_S, LDI_S, d_S)$ [S1].
    *   $P_S \in \mathbb{R}^3$: World position of the sprite [S1].
    *   $M_S$: Linear transformation matrix (rotation, scaling) of the VDS [S1].
    *   $d_S \in \mathbb{N} \setminus \{0, 1\}$: Depth complexity, representing the number of unique depth layers [S1].
    *   $LDI_S$: A 3D texture storing depth values. The texture is sampled in LDI texture space. For a 2D point $(s,t)$, the depth value $d_i$ for the $i$-th layer ($0 \le i < d_S$) is obtained by sampling at $(s, t, \frac{1}{d_S} \cdot i)$ [S1].

**Transformation to LDI Texture Space:**
Before testing, the point $P$ must be transformed into the specific LDI texture space of the VDS [S1]. The transformation $T = (T_s, T_t, T_r)$ is calculated as:
$$ T = M_S \cdot O^{-1} \cdot (P - P_S) $$
Where:
*   $P$ is the input point [S1].
*   $O$ is the current orientation matrix [S1].
*   $P - P_S$ translates the point to the LDI space relative to the sprite's position [S1].
*   $O^{-1}$ adjusts for orientation [S1].
*   $M_S$ applies the VDS model matrix [S1].

**Classification Logic:**
The test models a ray $R = [(T_s, T_t, 0), (T_s, T_t, 1)]$ marching through the depth layers of the LDI [S1]. The parity $p_T$ starts at an initial value (typically `true` for outside) and swaps every time the ray crosses a layer of unique depth complexity [S1].

Mathematically, the test is formulated as:
$$ VPT(T, VDS^S) = p_T = \begin{cases} 1, & \exists d_i \in LDI_S : d_i \le T_r \le d_{i+1} \\ 0, & \text{otherwise} \end{cases} $$
*Note: The source text formula uses $d_i \le T_r \le d_{i+1}$, but the provided code implementation checks $T_r < d_i$ [S1]. The code logic implies that if the ray's depth $T_r$ is less than the stored depth $d_i$ of a layer, the ray has passed that layer, triggering a parity swap [S1].*

**Pseudocode:**
The following GLSL-like pseudocode implements the VPT [S1]. It requires Shader Model 3.0 compliant hardware due to dynamic flow control in the loop [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volumetricParityTest(T, LDI, depth, initParity):
    """
    Classifies point T in LDI-space against the VDS stored in LDI.
    
    Args:
        T (vec4): Coordinate in LDI-space. T.r is the depth component.
        LDI (sampler3D): The 3D texture representing the Layered Depth Image.
        depth (int): The depth complexity d_S of the VDS.
        initParity (bool): Initial parity (true = outside).
        
    Returns:
        bool: True if inside, False if outside.
    """
    parity = initParity
    # Calculate offset to address texture slices uniformly
    offset = 1.0 / float(depth)
    
    # March through each depth layer
    for i in range(0, depth):
        # Sample the depth value at the current layer
        # Texture coordinate: (T.st, offset * i)
        sampled_depth = texture3D(LDI, vec3(T.st, offset * i)).x
        
        # If the point's depth T.r is less than the layer's depth,
        # the ray has crossed this layer.
        if T.r < sampled_depth:
            parity = not parity
            
    return parity
```

**Parameters and Edge Cases:**
*   **Shader Model:** Requires Shader Model 3.0 or higher to support the dynamic flow control (loops) required for the ray march [S1].
*   **Initial Parity:** `initParity = true` corresponds to "outside" [S1].
*   **Precision:** The method does not consider the case where the coordinate lies exactly on the border of the shape for reasons of precision [S1].
*   **Depth Linearization:** Depth values in the LDI may need to be linearized using an inverse W-buffer [S1].
*   **Resolution Limits:** For complex non-convex solids, VPT might fail if the texture resolution is too low (e.g., $128^2$ pixels). A texture size of $1024^2$ pixels delivers fairly high quality, and larger resolutions do not significantly improve quality [S1].

## Operating characteristics
*   **Time Complexity:** The test consists of less than 20 assembler instructions per executed loop iteration [S1]. Performance depends on the depth complexity $d_S$ and the number of VDS samples [S1].
*   **Memory Complexity:** The key drawback is the high space complexity of the LDI 3D texture [S1].
*   **Performance:** On a GeForce 8800 GTS, the method achieves interactive frame rates. For example, rendering a "House" model with 10 samples took 74.4 FPS without the test and 73.2 FPS with the test; a "Crank" model with 80 samples took 8.6 FPS without and 7.5 FPS with the test [S1].
*   **Pre-processing:** Creating the VDS takes up to two seconds per shape, depending on depth complexity and texture dimension [S1].
*   **Strengths:** Enables pixel-precise clipping against multiple arbitrary-shaped meshes in a single rendering pass, unlike image-based CSG which requires multiple passes per frame [S1].
*   **Limitations:** High memory usage for the LDI; potential failure for complex non-convex solids at low resolutions [S1].

## Background
The Volumetric Parity Test was introduced by Matthias Trapp and Jürgen Döllner in 2008 [S1]. It was developed to address the need for fast, real-time volumetric tests in computer graphics and visualization, such as rendering 3D volumetric lenses or collision detection [S1]. Prior to this, such functionality relied on expensive image-based multi-pass rendering algorithms or lacked support for multiple arbitrary-shaped meshes [S1]. The method is implemented based on the Virtual Rendering System (VRS) [S1].

## Sources
[S1] Real-Time Volumetric Tests Using Layered Depth Images, https://doi.org/10.2312/egs.20081019, fetched 2026-10-26
