# Layered SDF Compositing


## Overview
Layered SDF Compositing is a technique for rendering Signed Distance Field (SDF) glyphs by compositing multiple pre-multiplied alpha layers (typically a face and one or more outline layers) into a final pixel color. It solves the problem of rendering crisp, scalable text or icons with anti-aliased edges and colored outlines by treating each visual component as a distinct alpha-blended layer. It differs from standard SDF rendering by explicitly managing a fixed number of discrete layers (1, 2, or 4) using specific linear interpolation logic on pre-multiplied colors, rather than a continuous ray-casting integration [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method takes a set of $N$ layers, where $N \in \{1, 2, 4\}$. Each layer $i$ provides a color $C_i$ (RGB) and an alpha $\alpha_i$ (A). The inputs are typically the raw sampled colors and alphas from the SDF atlas. The output is a single final color $C_{final}$ (RGB) and accumulated alpha $\alpha_{final}$ (A).

**Data Structures:**
The layers are processed in a specific order, usually from back to front or front to back depending on the interpolation logic. The notes imply a specific ordering for the `lerp` operations in Layer4: $C_0$ is the base (front?), $C_1$ is next, $C_2$ next, $C_3$ is the back. However, the standard compositing logic described in the parent article is front-to-back. The Layer4 formula `lerp(lerp(lerp(color3, color2, alpha.z), color1, alpha.y), color0, alpha.x)` suggests a specific hierarchical blending. Let's analyze the Layer2 formula: `lerp(color1, color0, alpha.x)`. If `alpha.x` is the alpha of layer 0, and we lerp between color1 and color0, this looks like blending layer 0 over layer 1.

**Steps:**
1.  **Pre-multiplication:** For each layer $i$, multiply its RGB color by its alpha: $C_i' = C_i \cdot \alpha_i$.
2.  **Layer-Specific Compositing:**
    *   **Layer1:** The output color is simply the pre-multiplied color of the single layer.
    *   **Layer2:** Blend Layer 1 over Layer 0.
    *   **Layer4:** Blend Layer 3 over Layer 2, then that result over Layer 1, then that result over Layer 0.
3.  **Un-premultiplication:** Divide the resulting RGB color by the resulting alpha to get the final non-premultiplied color.

**Formulas:**
Let $C_i$ and $\alpha_i$ be the input color and alpha for layer $i$.
Let $C_i' = C_i \cdot \alpha_i$ be the pre-multiplied color.

**Layer1:**
$$
C_{out} = C_0'
$$
$$
\alpha_{out} = \alpha_0
$$
(Note: The source says `outColor = color0` after multiplying `color0.a` by alpha. It implies the input `color0` might be un-premultiplied, so we premultiply it. The output `outColor` is the result. The division step is mentioned for Layer2/4 but not explicitly for Layer1, but consistency suggests un-premultiplication if the final output is expected to be non-premultiplied. However, the source says "divide outColor.rgb by outColor.a" for Layer2/4. For Layer1, it just sets `outColor`. We will assume the final step applies to all if a non-premultiplied result is needed, or that Layer1 is already in the correct format. Given the explicit division in L2/L4, we apply it to L1 as well for consistency of the "method" definition, or note it's optional if the consumer handles premultiplied alpha. The source says "divide outColor.rgb by outColor.a" for L2 and L4. It does NOT say it for L1. We will follow the source strictly.)

**Layer2:**
Inputs: $C_0, \alpha_0$ (Layer 0), $C_1, \alpha_1$ (Layer 1).
Parameters: $\alpha_x = \alpha_0$, $\alpha_y = \alpha_1$.
1.  Premultiply: $C_0' = C_0 \cdot \alpha_0$, $C_1' = C_1 \cdot \alpha_1$.
2.  Interpolate:
    $$
    C_{temp} = \text{lerp}(C_1', C_0', \alpha_x) = C_1' \cdot (1 - \alpha_x) + C_0' \cdot \alpha_x
    $$
    Wait, `lerp(a, b, t)` is usually $a(1-t) + bt$. The source says `lerp(color1, color0, alpha.x)`. So $a=color1, b=color0, t=\alpha.x$.
    $$
    C_{temp} = C_1' (1 - \alpha_0) + C_0' \alpha_0
    $$
3.  Resulting Alpha: The source doesn't explicitly state the alpha calculation for the intermediate step, but standard alpha compositing applies. However, the `lerp` on colors is done on pre-multiplied values. The resulting alpha $\alpha_{out}$ must be calculated.
    Standard compositing of $\alpha_0$ over $\alpha_1$:
    $$
    \alpha_{out} = \alpha_1 + \alpha_0 (1 - \alpha_1)
    $$
    The source says `outColor = lerp(...)`. It does not explicitly give the alpha formula for the intermediate blend, but the final division requires `outColor.a`.
    Let's look at the Layer4 structure. It nests lerp.
    $$
    C_{temp} = \text{lerp}(C_3', C_2', \alpha_2)
    $$
    $$
    C_{temp2} = \text{lerp}(C_{temp}, C_1', \alpha_1)
    $$
    $$
    C_{out} = \text{lerp}(C_{temp2}, C_0', \alpha_0)
    $$
    This `lerp` logic on pre-multiplied colors is equivalent to standard alpha compositing IF the interpolation factor is the alpha of the *top* layer.
    Let's verify Layer2: `lerp(C1', C0', alpha0)`.
    Standard compositing of $C_0$ over $C_1$:
    $C_{out} = C_1' + C_0' (1 - \alpha_1)$? No, standard is $C_{out} = C_1' + C_0' (1 - \alpha_1)$ if $C_1$ is bottom.
    Wait, standard front-to-back: $C_{out} = C_{bottom}' + C_{top}' (1 - \alpha_{bottom})$.
    The source uses `lerp(C1, C0, alpha0)`.
    If $C_1$ is bottom and $C_0$ is top:
    $C_{out} = C_1 (1 - \alpha_0) + C_0 \alpha_0$.
    This is NOT standard alpha compositing. Standard alpha compositing of $C_0$ over $C_1$ is $C_1 + C_0(1-\alpha_1)$? No.
    Standard: $C_{out} = C_1' + C_0' (1 - \alpha_1)$ is wrong.
    Standard: $C_{out} = C_1' + C_0' (1 - \alpha_1)$?
    Let's check: $C_{out} = C_1' + C_0' (1 - \alpha_1)$?
    Actually, $C_{out} = C_1' + C_0' (1 - \alpha_1)$ is correct if $C_1$ is behind $C_0$.
    The source formula: $C_{out} = C_1' (1 - \alpha_0) + C_0' \alpha_0$.
    This implies a different blending model, possibly "soft" blending or specific SDF outline blending where the alpha acts as a weight for the top layer against the bottom layer, rather than strict opacity occlusion.
    However, the source says "Composites SDF layers... using alpha blending".
    Let's stick to the source's explicit formulas.

    **Layer2 Formula:**
    $$
    C_{out} = C_1' (1 - \alpha_0) + C_0' \alpha_0
    $$
    $$
    \alpha_{out} = \alpha_1 + \alpha_0 (1 - \alpha_1) \quad \text{[G: Standard alpha compositing for the resulting alpha, as the color blend is non-standard]}
    $$
    *Correction*: If the color blend is `lerp(C1, C0, alpha0)`, and we divide by `outColor.a`, what is `outColor.a`?
    If we use standard alpha compositing for alpha: $\alpha_{out} = \alpha_1 + \alpha_0 - \alpha_1 \alpha_0$.
    The source does not specify the alpha calculation for the intermediate layers, only the final division. We must assume standard alpha accumulation for the alpha channel to make the division valid (avoiding division by zero or incorrect normalization).

    **Layer4 Formula:**
    Inputs: $C_0, \alpha_0$; $C_1, \alpha_1$; $C_2, \alpha_2$; $C_3, \alpha_3$.
    Premultiply all: $C_i' = C_i \cdot \alpha_i$.
    1.  $C_{temp} = \text{lerp}(C_3', C_2', \alpha_2) = C_3' (1 - \alpha_2) + C_2' \alpha_2$
    2.  $C_{temp2} = \text{lerp}(C_{temp}, C_1', \alpha_1) = C_{temp} (1 - \alpha_1) + C_1' \alpha_1$
    3.  $C_{out} = \text{lerp}(C_{temp2}, C_0', \alpha_0) = C_{temp2} (1 - \alpha_0) + C_0' \alpha_0$
    4.  $\alpha_{out}$: Calculated via standard alpha compositing of all layers? Or derived from the color?
        The source says "divide outColor.rgb by outColor.a". It implies `outColor.a` is the accumulated alpha.
        Standard alpha compositing for 4 layers:
        $$
        \alpha_{out} = 1 - (1-\alpha_0)(1-\alpha_1)(1-\alpha_2)(1-\alpha_3)
        $$
    5.  Final Color:
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
        # Layer1: outColor = color0 (premultiplied)
        # Alpha is a0
        # If non-premultiplied output is required:
        if a0 > 0:
            c_final = c0_p / a0
        else:
            c_final = vec3(0.0)
        return c_final, a0

    elif num_layers == 2:
        c0_p, a0 = premultiplied[0]
        c1_p, a1 = premultiplied[1]
        
        # Layer2: lerp(color1, color0, alpha.x) where alpha.x = alpha0
        # c_temp = c1_p * (1 - a0) + c0_p * a0
        c_temp = c1_p * (1.0 - a0) + c0_p * a0
        
        # Alpha accumulation (Standard)
        alpha_out = a1 + a0 * (1.0 - a1)
        
        if alpha_out > 0:
            c_final = c_temp / alpha_out
        else:
            c_final = vec3(0.0)
        return c_final, alpha_out

    elif num_layers == 4:
        c0_p, a0 = premultiplied[0]
        c1_p, a1 = premultiplied[1]
        c2_p, a2 = premultiplied[2]
        c3_p, a3 = premultiplied[3]
        
        # Layer4: nested lerp
        # lerp(color3, color2, alpha.z) -> alpha.z = a2
        c_temp1 = c3_p * (1.0 - a2) + c2_p * a2
        
        # lerp(temp1, color1, alpha.y) -> alpha.y = a1
        c_temp2 = c_temp1 * (1.0 - a1) + c1_p * a1
        
        # lerp(temp2, color0, alpha.x) -> alpha.x = a0
        c_temp3 = c_temp2 * (1.0 - a0) + c0_p * a0
        
        # Alpha accumulation (Standard)
        # alpha_out = 1 - (1-a0)(1-a1)(1-a2)(1-a3)
        transmittance = (1.0 - a0) * (1.0 - a1) * (1.0 - a2) * (1.0 - a3)
        alpha_out = 1.0 - transmittance
        
        if alpha_out > 0:
            c_final = c_temp3 / alpha_out
        else:
            c_final = vec3(0.0)
        return c_final, alpha_out
    else:
        raise ValueError("Only 1, 2, or 4 layers are supported")
```

**Edge Cases:**
- Division by zero: If `alpha_out` is 0, the color is undefined. The code returns `(0,0,0)`.
- Precision: Premultiplication can lose precision if alphas are very small.

## Subtopics

### [SDF Underlay & Inner Underlay](5612-sdf-underlay-inner-underlay.md)
Renders a blurred, colored border either behind (underlay) or inside (inner underlay) the primary glyph face. It differs by offsetting the UV sampling in the SDF atlas and blending with the face color, allowing for "inner" outlines that appear within the character fill. Choose this when you need decorative or depth-indicating borders that are not strictly part of the glyph's stroke geometry.

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel, as the number of layers is fixed (1, 2, or 4).
- **Memory Complexity:** $O(1)$ per pixel, storing only the layer colors and alphas.
- **Strengths:** Extremely fast; no ray casting or volume integration required; supports crisp, scalable text with colored outlines.
- **Limitations:** Limited to a fixed number of layers; the blending model is not standard optical compositing (it uses linear interpolation on pre-multiplied colors), which may produce different visual results than standard alpha blending.
- **When to use:** Rendering SDF text/icons with simple outline effects.
- **When not to use:** When complex lighting, shadows, or true volumetric transparency is required.

## Background
Originating from the Unity NanoVDB renderer implementation, this method is an unofficial technique for compositing SDF layers. It is used in real-time rendering engines to render high-quality text and icons using Signed Distance Fields.

## Sources
[S1] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
