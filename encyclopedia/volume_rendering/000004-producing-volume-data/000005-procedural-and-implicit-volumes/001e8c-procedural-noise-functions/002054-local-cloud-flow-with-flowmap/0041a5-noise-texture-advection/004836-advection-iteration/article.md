# Advection Iteration

Also called: LIC-like Advection

## Overview
Advection Iteration is a GPU-based technique for visualizing tensor fields by iteratively advecting a noise texture along eigenvector directions. It solves the problem of generating coherent flow-like structures from static tensor data by leveraging the depth buffer for edge detection, thereby avoiding the overhead of separate depth textures. Unlike its parent method, Noise Texture Advection, which uses simple Euler integration, this method specifically handles the lack of orientation in eigenvectors by sampling in both forward and backward directions and averaging the results.

## How it works
The method operates in two phases: a CPU-controlled loop that manages texture ping-ponging and a GPU shader that performs the per-pixel advection logic.

### Inputs and Outputs
**Inputs:**
*   `edgeTexture`: Output from the silhouette detection step.
    *   Channel G: Depth value.
    *   Channel R: Edge intensity.
    *   Channel B: Original noise value.
*   `eigenvectorColormap`: Output from the projection step.
    *   Channels R, G: First eigenvector ($ev_1$) components.
    *   Channels B, A: Second eigenvector ($ev_2$) components.
*   `advectionInput`: Texture from the previous iteration (or initial noise).
    *   Channel R: Advected noise for $ev_1$.
    *   Channel G: Advected noise for $ev_2$.

**Outputs:**
*   `advectionOutput`: Texture for the current iteration.
    *   Channel R: Final advected noise for $ev_1$.
    *   Channel G: Final advected noise for $ev_2$.
    *   Channel B: Unused (0.0).
    *   Channel A: 1.0.

### Data Structures and Layout
The implementation uses a ping-pong buffer strategy with two textures, `advection1` and `advection2`, to alternate between input and output roles.

*   **edgeTexture**: `RGBA` texture. `G` stores depth, `R` stores edge intensity, `B` stores original noise.
*   **eigenvectorColormap**: `RGBA` texture. `(R, G)` stores $ev_1$, `(B, A)` stores $ev_2$. Values are stored in $[0, 1]$ range.
*   **advectionInput/Output**: `RGBA` textures. `R` stores $ev_1$ noise, `G` stores $ev_2$ noise.

### Parameters
*   `MAX`: Maximum number of advection iterations per frame. [S1, unofficial]
*   `k`: Ratio parameter from Definition 4.9, controlling the blend between advected noise and original noise. [S1, unofficial]
*   `edgeBlend`: Constant defining the edge intensity contribution during advection. [S1, unofficial]
*   `s`: Step size used for sampling positions along eigenvectors. [S1, unofficial]

### Algorithm Steps

#### CPU Initialization
1.  Bind `edgeTexture` and `eigenvectorColormap`.
2.  Create two textures: `advection1` and `advection2` for ping-ponging.
3.  Set `advectionInput` to the initial noise texture (e.g., `reactionDiffusionTexture`). [S1, unofficial]

#### CPU Loop
Iterate $i$ from 1 to `MAX`:
1.  Bind `advectionInput` as the input texture.
2.  Determine the output texture based on the iteration parity:
    *   If $i$ is even: Set `advection1` as the output texture. Update `advectionInput` to `advection1` for the next iteration.
    *   If $i$ is odd: Set `advection2` as the output texture. Update `advectionInput` to `advection2` for the next iteration.
3.  Render a full-screen quad to the Framebuffer Object (FBO) associated with the selected output texture. [S1, unofficial]

**Note:** If the iteration is broken early for framerate reasons, the textures and framebuffer objects must not be destroyed at the end of the frame to allow continuation in the next frame. [S1, unofficial]

#### GPU Shader (Per Pixel)
For each pixel at texture coordinate $t$:

1.  **Data Extraction**:
    *   Extract depth: $d = \text{edgeTexture}[t].G$
    *   Extract edge intensity: $e = \text{edgeTexture}[t].R$
    *   Extract tensor eigenvectors: $V = \text{eigenvectorColormap}[t].RGBA$

2.  **Vector Unscaling**:
    Unscale tensor values from $[0, 1]$ to $[-1, 1]$:
    $$ V' = (V - 0.5) \times 2.0 $$

3.  **Vector Scaling to Texture Space**:
    Scale eigenvectors to texture space coordinates. Let $W$ be texture width and $H$ be texture height.
    $$ ev_1 = \left( \frac{V'.R}{W}, \frac{V'.G}{H} \right) $$
    $$ ev_2 = \left( \frac{V'.B}{W}, \frac{V'.A}{H} \right) $$

4.  **Discard Check**:
    If `discardable(d, ev_1, ev_2)` returns true, output black $(0, 0, 0, 1)$ and terminate.
    *   `discardable` checks if both directions and anti-directions of $ev_1$ and $ev_2$ point to fragments without geometry using the depth buffer. [S1, unofficial]

5.  **Position Calculation**:
    Calculate forward and backward sample positions for both eigenvectors. Since eigenvectors lack orientation, advection occurs in both directions.
    $$ a_1 = t - s \cdot ev_1 $$
    $$ a_2 = t - s \cdot ev_2 $$
    $$ a_{1n} = t + s \cdot ev_1 $$
    $$ a_{2n} = t + s \cdot ev_2 $$

6.  **Noise Sampling**:
    Sample `advectionInput` at the calculated positions. Average the forward and backward samples for each eigenvector.
    $$ \text{aNoise}_1 = 0.5 \times (\text{advectionInput}[a_1].R + \text{advectionInput}[a_{1n}].R) $$
    $$ \text{aNoise}_2 = 0.5 \times (\text{advectionInput}[a_2].G + \text{advectionInput}[a_{2n}].G) $$

7.  **Final Output**:
    Blend the averaged advected noise with the original noise (weighted by $k$) and add the edge contribution (weighted by `edgeBlend`).
    $$ \text{out} = (\text{aNoise}_1, \text{aNoise}_2, 0.0, 1.0) \times (1.0 - k) + (e, e, e, 1.0) \times k + (e, e, e, 1.0) \times \text{edgeBlend} $$
    
    *Correction based on source formula:* The source formula is `out = (vec4(aNoise1, aNoise2, 0.0, 1.0) * (1.0-k)) + vec4(noise * k) + vec4(edge * edgeBlend)`. Here `noise` is `edgeTexture[t].B`.
    
    $$ \text{out} = \begin{pmatrix} \text{aNoise}_1 \\ \text{aNoise}_2 \\ 0.0 \\ 1.0 \end{pmatrix} (1.0 - k) + \begin{pmatrix} \text{edgeTexture}[t].B \\ \text{edgeTexture}[t].B \\ \text{edgeTexture}[t].B \\ 1.0 \end{pmatrix} k + \begin{pmatrix} e \\ e \\ e \\ 1.0 \end{pmatrix} \text{edgeBlend} $$

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# CPU Side
def init_advection():
    edge_texture = load_or_get_edge_texture()
    eigenvector_colormap = load_or_get_eigenvector_colormap()
    advection1 = create_texture()
    advection2 = create_texture()
    advection_input = reaction_diffusion_texture # Initial noise
    return edge_texture, eigenvector_colormap, advection1, advection2, advection_input

def run_advection_loop(edge_texture, eigenvector_colormap, advection1, advection2, advection_input, MAX, s, k, edge_blend):
    current_input = advection_input
    
    for i in range(1, MAX + 1):
        bind_texture(current_input)
        
        if i % 2 == 0:
            output_texture = advection1
            next_input = advection1
        else:
            output_texture = advection2
            next_input = advection2
            
        render_quad_offscreen(output_texture, edge_texture, eigenvector_colormap, s, k, edge_blend)
        current_input = next_input
        
    return current_input

# GPU Side (Fragment Shader)
def advect_iteration_shader(t, edge_texture, eigenvector_colormap, advection_input, s, k, edge_blend, texture_width, texture_height):
    # 1. Data Extraction
    depth = edge_texture[t].G
    edge = edge_texture[t].R
    tensor = eigenvector_colormap[t].RGBA
    
    # 2. Vector Unscaling
    tensor_unscaled = (tensor - 0.5) * 2.0
    
    # 3. Vector Scaling to Texture Space
    ev1 = vec2(tensor_unscaled.R / texture_width, tensor_unscaled.G / texture_height)
    ev2 = vec2(tensor_unscaled.B / texture_width, tensor_unscaled.A / texture_height)
    
    # 4. Discard Check
    if discardable(depth, ev1, ev2):
        return vec4(0.0, 0.0, 0.0, 1.0)
        
    # 5. Position Calculation
    a1 = t - s * ev1
    a2 = t - s * ev2
    a1n = t + s * ev1
    a2n = t + s * ev2
    
    # 6. Noise Sampling
    # Average forward and backward samples
    aNoise1 = 0.5 * (advection_input[a1].R + advection_input[a1n].R)
    aNoise2 = 0.5 * (advection_input[a2].G + advection_input[a2n].G)
    
    # 7. Final Output
    noise = edge_texture[t].B
    
    out_r = aNoise1 * (1.0 - k) + noise * k + edge * edge_blend
    out_g = aNoise2 * (1.0 - k) + noise * k + edge * edge_blend
    out_b = 0.0
    out_a = 1.0
    
    return vec4(out_r, out_g, out_b, out_a)

def discardable(depth, ev1, ev2):
    # Check if both directions and anti-directions point to fragments without geometry
    # This uses the depth buffer to determine if the sampled positions are empty space
    # Implementation detail: [G] Standard depth comparison logic would go here.
    # If all 4 directions (ev1, -ev1, ev2, -ev2) lead to empty space, return True.
    pass
```

### Edge Cases and Pitfalls
*   **Texture Management**: When breaking iteration early for framerate, textures and framebuffer objects must not be destroyed at the end of each frame to allow continuation in the next frame. [S1, unofficial]
*   **Input Binding Count**: Due to data merging in the prior step, only three input texture bindings are needed per iteration (depth/edge, eigenvector colormap, previous advection output) instead of four. [S1, unofficial]
*   **Eigenvector Orientation**: Eigenvectors lack orientation. The method must sample in both forward and backward directions and average the results to avoid directional bias. [S1, unofficial]

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of iterations `MAX` and the number of pixels. Each iteration involves a constant number of texture lookups and arithmetic operations per pixel.
*   **Memory Complexity**: Requires storage for two ping-pong textures (`advection1`, `advection2`) plus the input textures (`edgeTexture`, `eigenvectorColormap`).
*   **Performance**: Allows breaking iteration after a fixed number of passes to ensure higher framerates and a smoother user experience. [S1, unofficial]
*   **Strengths**: Avoids separate depth texture overhead by using the depth buffer directly for edge detection. Efficiently handles tensor field visualization without complex geometry mapping for near-planar geometry. [S1, unofficial]
*   **Limitations**: Dependent on the quality of the eigenvector colormap and silhouette detection. The step size `s` must be chosen carefully to balance coherence and detail.

## Background
Advection Iteration is described in the context of image-space tensor field visualization using a LIC-like (Line Integral Convolution) method. It is part of a pipeline that includes silhouette detection, projection, and advection. The method was introduced in the paper "Image-Space Tensor Field Visualization Using a LIC-like Method". [S1, unofficial]

## Sources
[S1] Image-Space Tensor Field Visualization Using a LIC-like Method, https://doi.org/10.1007/978-3-642-21608-4_11, fetched 2026-10-27 (unofficial)
