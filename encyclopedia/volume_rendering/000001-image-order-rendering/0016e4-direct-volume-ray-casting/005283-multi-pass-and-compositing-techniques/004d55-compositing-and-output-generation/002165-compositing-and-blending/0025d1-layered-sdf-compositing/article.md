# Layered SDF Compositing

## Overview
Layered SDF Compositing composites Signed Distance Field (SDF) glyph layers (typically a face and one or more outlines) using alpha blending and pre-multiplied alpha handling [S1, unofficial]. It differs from standard ray-casting volume rendering by explicitly managing a fixed number of discrete layers (1, 2, or 4) using specific linear interpolation logic on pre-multiplied colors [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method takes a set of $N$ layers, where $N \in \{1, 2, 4\}$ [S1, unofficial]. Each layer $i$ provides a color $C_i$ (RGB) and an alpha $\alpha_i$ (A) [S1, unofficial]. The inputs are typically the raw sampled colors and alphas from the SDF atlas [S1, unofficial]. The output is a single final color $C_{final}$ (RGB) and accumulated alpha $\alpha_{final}$ (A) [S1, unofficial].

**Data Structures:**
The layers are processed in a specific order [S1, unofficial]. The notes imply a specific ordering for the `lerp` operations in Layer4: $C_0$ is the base (front?), $C_1$ is next, $C_2$ next, $C_3$ is the back [S1, unofficial]. However, the standard compositing logic described in the parent article is front-to-back [G]. The Layer4 formula `lerp(lerp(lerp(color3, color2, alpha.z), color1, alpha.y), color0, alpha.x)` suggests a specific hierarchical blending [S1, unofficial]. Let's analyze the Layer2 formula: `lerp(color1, color0, alpha.x)` [S1, unofficial]. If `alpha.x` is the alpha of layer 0, and we lerp between color1 and color0, this looks like blending layer 0 over layer 1 [S1, unofficial].

**Steps:**
1. **Pre-multiplication:** For each layer $i$, multiply its RGB color by its alpha: $C_i' = C_i \cdot \alpha_i$ [S1, unofficial].
2. **Layer-Specific Compositing:**
   - **Layer1:** Multiply `color0.a` by `alpha`; set `outColor = color0` [S1, unofficial].
   - **Layer2:** Multiply `color1.a` by `alpha.y`; compute `outColor = lerp(color1, color0, alpha.x)` [S1, unofficial].
   - **Layer4:** Multiply `color3.a` by `alpha.w`; compute `outColor = lerp(lerp(lerp(color3, color2, alpha.z), color1, alpha.y), color0, alpha.x)` [S1, unofficial].
3. **Un-premultiplication:** Divide `outColor.rgb` by `outColor.a` to obtain the final non-premultiplied color [S1, unofficial].

**Formulas:**
Let $C_i$ and $\alpha_i$ be the input color and alpha for layer $i$.
Let $C_i' = C_i \cdot \alpha_i$ be the pre-multiplied color.
The `lerp` function is defined as $\text{lerp}(A, B, t) = A(1-t) + Bt$.

**Layer1:**
$$
C_{out} = C_0'
$$
$$
\alpha_{out} = \alpha_0
$$
The source specifies `outColor = color0` after multiplying `color0.a` by `alpha` [S1, unofficial].

**Layer2:**
Inputs: $C_0, \alpha_0$ (Layer 0), $C_1, \alpha_1$ (Layer 1).
Parameters: $\alpha_x = \alpha_0$, $\alpha_y = \alpha_1$.
$$
C_{out} = C_1' (1 - \alpha_0) + C_0' \alpha_0
$$
$$
\alpha_{out} = \alpha_1 + \alpha_0 (1 - \alpha_1) \quad \text{[G: Standard alpha compositing for the resulting alpha]}
$$

**Layer4:**
Inputs: $C_0, \alpha_0$; $C_1, \alpha_1$; $C_2, \alpha_2$; $C_3, \alpha_3$.
$$
C_{out} = \text{lerp}(\text{lerp}(\text{lerp}(C_3', C_2', \alpha_2), C_1', \alpha_1), C_0', \alpha_0)
$$
$$
\alpha_{out} = 1 - (1-\alpha_0)(1-\alpha_1)(1-\alpha_2)(1-\alpha_3) \quad \text{[G: Standard alpha compositing for the resulting alpha]}
$$
Final Color:
$$
C_{final} = \frac{C_{out}}{\alpha_{out}}
$$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def layered_sdf_compositing(layers, num_layers):
    """
    layers: list of tuples (color: vec3, alpha: float)
    num_layers: int (1, 2, or 4)
    Returns: (final_color: vec3, final_alpha: float)
    """
    # Premultiply colors
    premultiplied = []
    for i in range(num_layers):
        c, a = layers[i]
        premultiplied.append((c * a, a))
    
    if num_layers == 1:
        c0_p, a0 = premultiplied[0]
        # Layer1: Multiply color0.a by alpha; set outColor = color0
        c_out = c0_p
        alpha_out = a0
        if alpha_out > 0:
            c_final = c_out / alpha_out
        else:
            c_final = vec3(0.0)
        return c_final, alpha_out

    elif num_layers == 2:
        c0_p, a0 = premultiplied[0]
        c1_p, a1 = premultiplied[1]
        
        # Layer2: Multiply color1.a by alpha.y; compute outColor = lerp(color1, color0, alpha.x)
        c_out = c1_p * (1.0 - a0) + c0_p * a0
        
        # Alpha accumulation (Standard)
        alpha_out = a1 + a0 * (1.0 - a1)
        
        if alpha_out > 0:
            c_final = c_out / alpha_out
        else:
            c_final = vec3(0.0)
        return c_final, alpha_out

    elif num_layers == 4:
        c0_p, a0 = premultiplied[0]
        c1_p, a1 = premultiplied[1]
        c2_p, a2 = premultiplied[2]
        c3_p, a3 = premultiplied[3]
        
        # Layer4: Multiply color3.a by alpha.w; compute outColor = lerp(lerp(lerp(color3, color2, alpha.z), color1, alpha.y), color0, alpha.x)
        c_temp1 = c3_p * (1.0 - a2) + c2_p * a2
        c_temp2 = c_temp1 * (1.0 - a1) + c1_p * a1
        c_out = c_temp2 * (1.0 - a0) + c0_p * a0
        
        # Alpha accumulation (Standard)
        transmittance = (1.0 - a0) * (1.0 - a1) * (1.0 - a2) * (1.0 - a3)
        alpha_out = 1.0 - transmittance
        
        if alpha_out > 0:
            c_final = c_out / alpha_out
        else:
            c_final = vec3(0.0)
        return c_final, alpha_out
    else:
        raise ValueError("Only 1, 2, or 4 layers are supported")
```

**Edge Cases:**
- Division by zero: If `alpha_out` is 0, the color is undefined. The code returns `(0,0,0)` [G].
- Precision: Premultiplication can lose precision if alphas are very small [G].

## Subtopics

### [SDF Underlay & Inner Underlay](5612-sdf-underlay-inner-underlay.md)
Renders a blurred, colored border either behind (underlay) or inside (inner underlay) the primary glyph face. It differs by offsetting the UV sampling in the SDF atlas and blending with the face color, allowing for "inner" outlines that appear within the character fill. Choose this when you need decorative or depth-indicating borders that are not strictly part of the glyph's stroke geometry.

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel, as the number of layers is fixed (1, 2, or 4) [S1, unofficial].
- **Memory Complexity:** $O(1)$ per pixel, storing only the layer colors and alphas [S1, unofficial].
- **Strengths:** Extremely fast; no ray casting or volume integration required; supports crisp, scalable text with colored outlines [S1, unofficial].
- **Limitations:** Limited to a fixed number of layers; the blending model is not standard optical compositing (it uses linear interpolation on pre-multiplied colors), which may produce different visual results than standard alpha blending [S1, unofficial].
- **When to use:** Rendering SDF text/icons with simple outline effects [S1, unofficial].
- **When not to use:** When complex lighting, shadows, or true volumetric transparency is required [S1, unofficial].

## Background
Originating from the Unity NanoVDB renderer implementation, this method is an unofficial technique for compositing SDF layers [S1, unofficial]. It is used in real-time rendering engines to render high-quality text and icons using Signed Distance Fields [S1, unofficial].

## Sources
[S1] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
