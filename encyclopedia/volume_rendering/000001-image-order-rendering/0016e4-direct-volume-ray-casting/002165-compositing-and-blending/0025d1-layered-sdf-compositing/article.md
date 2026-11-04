# Layered SDF Compositing


## Overview
Layered SDF Compositing is a method for rendering Signed Distance Field (SDF) glyphs by compositing multiple pre-multiplied color layers (typically a face and one or more outline layers) using alpha blending. It solves the problem of rendering crisp, anti-aliased text or icons with variable-width outlines by treating each visual layer as a separate alpha channel in a single pass. It differs from standard [Alpha Blending](5750-alpha-blending.md) by explicitly managing pre-multiplied alpha arithmetic across multiple distinct layers to support complex outline structures, and from [SDF Underlay & Inner Underlay](5612-sdf-underlay-inner-underlay.md) by compositing discrete layers rather than sampling a blurred border offset.

## How it works
**Inputs and Outputs:**
- Input: A set of $N$ layers (where $N \in \{1, 2, 4\}$), each consisting of a color vector $\mathbf{c}_i$ (RGB) and an alpha value $\alpha_i$. The alphas are packed into a single vector $\boldsymbol{\alpha} = (\alpha.x, \alpha.y, \alpha.z, \alpha.w)$ corresponding to layers $0, 1, 2, 3$.
- Output: A final pre-multiplied color $\mathbf{c}_{out}$ and a final alpha $\alpha_{out}$.

**Data Structures:**
The method assumes the input colors are not yet pre-multiplied. The algorithm performs pre-multiplication internally.

**Steps:**
1. **Pre-multiply and Scale:** For each layer $i$, multiply the color $\mathbf{c}_i$ by its corresponding alpha component $\alpha_i$.
2. **Composite:** Compute the weighted sum of the pre-multiplied colors using the alpha components as weights.
3. **Normalize:** Divide the resulting RGB color by the total accumulated alpha to obtain the final pre-multiplied output color.

**Formulas:**
Let $N$ be the number of layers.
For $N=1$ (Layer1_float):
$$ \mathbf{c}_{out} = \mathbf{c}_0 $$
$$ \alpha_{out} = \alpha_0 $$

For $N=2$ (Layer2_float):
$$ \mathbf{c}_{out} = \frac{\mathbf{c}_1 \alpha_1 + \mathbf{c}_0 \alpha_0}{\alpha_1 + \alpha_0} $$
Note: The source specifies `lerp(color1, color0, alpha.x)` which implies $\mathbf{c}_{out} = \mathbf{c}_1 (1 - \alpha_0) + \mathbf{c}_0 \alpha_0$ if interpreted as standard lerp, but the text explicitly says "premultiply color0.rgb by color0.a and color1.rgb by color1.a" and then "divide outColor.rgb by outColor.a". The standard alpha compositing formula for layer 1 over layer 0 is $\mathbf{c}_{out} = \mathbf{c}_1 \alpha_1 + \mathbf{c}_0 \alpha_0 (1 - \alpha_1)$. However, the source text for Layer2 says: `outColor = lerp(color1, color0, alpha.x)`. In many graphics APIs, `lerp(a, b, t)` is $a(1-t) + bt$. If `alpha.x` is $\alpha_0$, this is $\mathbf{c}_1(1-\alpha_0) + \mathbf{c}_0 \alpha_0$. This is non-standard compositing unless $\alpha_1 = 1 - \alpha_0$. Given the explicit instruction to "premultiply... and divide by outColor.a", the most faithful implementation of the *text* is:
$$ \mathbf{c}_{pre} = \mathbf{c}_1 \alpha_1 + \mathbf{c}_0 \alpha_0 $$
$$ \alpha_{out} = \alpha_1 + \alpha_0 $$
$$ \mathbf{c}_{out} = \frac{\mathbf{c}_{pre}}{\alpha_{out}} $$
*Correction*: The source says `lerp(color1, color0, alpha.x)`. If we assume standard alpha compositing where layer 1 is on top of layer 0, the formula is $\mathbf{c}_{out} = \mathbf{c}_1 \alpha_1 + \mathbf{c}_0 \alpha_0 (1 - \alpha_1)$. The source text is slightly ambiguous regarding the exact lerp parameters vs standard compositing. However, the Layer4 description uses nested lerp. We will follow the explicit "premultiply all... compute outColor = lerp... divide by outColor.a" pattern.

For $N=4$ (Layer4_float):
$$ \mathbf{c}_{out} = \frac{\mathbf{c}_3 \alpha_3 + \mathbf{c}_2 \alpha_2 (1-\alpha_3) + \mathbf{c}_1 \alpha_1 (1-\alpha_3)(1-\alpha_2) + \mathbf{c}_0 \alpha_0 (1-\alpha_3)(1-\alpha_2)(1-\alpha_1)}{\alpha_{total}} $$
The source specifies: `outColor = lerp(lerp(lerp(color3, color2, alpha.z), color1, alpha.y), color0, alpha.x)`.
This corresponds to:
1. $t_1 = \text{lerp}(\mathbf{c}_3, \mathbf{c}_2, \alpha_2) = \mathbf{c}_3(1-\alpha_2) + \mathbf{c}_2 \alpha_2$
2. $t_2 = \text{lerp}(t_1, \mathbf{c}_1, \alpha_1) = t_1(1-\alpha_1) + \mathbf{c}_1 \alpha_1$
3. $t_3 = \text{lerp}(t_2, \mathbf{c}_0, \alpha_0) = t_2(1-\alpha_0) + \mathbf{c}_0 \alpha_0$
4. $\mathbf{c}_{out} = t_3 / \alpha_{total}$

**Parameters:**
- Number of layers: 1, 2, or 4.

**Edge Cases:**
- Division by zero: If $\alpha_{total} = 0$, the output color is undefined. Typically, return black or transparent.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def layered_sdf_compositing(colors, alphas, num_layers):
    """
    colors: list of tuples (r, g, b) for each layer
    alphas: list of floats for each layer
    num_layers: 1, 2, or 4
    """
    # Pre-multiply colors
    premultiplied = []
    for i in range(num_layers):
        r, g, b = colors[i]
        a = alphas[i]
        premultiplied.append((r * a, g * a, b * a, a))

    if num_layers == 1:
        c0, a0 = premultiplied[0]
        return (c0, a0)

    elif num_layers == 2:
        c1, a1 = premultiplied[1]
        c0, a0 = premultiplied[0]
        # lerp(color1, color0, alpha.x) -> color1 * (1 - alpha0) + color0 * alpha0
        # Note: Source says premultiply then lerp then divide.
        # Standard alpha compositing is usually c1*a1 + c0*a0*(1-a1).
        # The source text is specific: lerp(color1, color0, alpha.x).
        # Assuming alpha.x is alpha0.
        r = c1[0] * (1 - a0) + c0[0] * a0
        g = c1[1] * (1 - a0) + c0[1] * a0
        b = c1[2] * (1 - a0) + c0[2] * a0
        alpha_out = a1 + a0 # Or 1.0 if fully opaque? Source says divide by outColor.a
        # Wait, source says "divide outColor.rgb by outColor.a".
        # If we use lerp, the resulting alpha is not simply sum.
        # Let's stick to the source's explicit steps:
        # 1. Premultiply.
        # 2. Lerp.
        # 3. Divide by outColor.a.
        # This implies outColor.a is the result of the lerp's alpha?
        # Lerp of premultiplied colors doesn't track alpha linearly in the same way.
        # Re-reading S1: "compute outColor = lerp(color1, color0, alpha.x); divide outColor.rgb by outColor.a"
        # This suggests outColor.a is computed separately or is the sum?
        # In Layer4, it's nested lerp.
        # Let's assume outColor.a is the accumulated alpha for normalization.
        alpha_out = a1 + a0 # Approximation, or 1.0 if opaque.
        # Actually, if we premultiply, we lose the original alpha for the lerp weight?
        # No, alpha.x is the original alpha.
        # Let's implement exactly as written:
        r_out = c1[0] * (1 - a0) + c0[0] * a0
        g_out = c1[1] * (1 - a0) + c0[1] * a0
        b_out = c1[2] * (1 - a0) + c0[2] * a0
        # What is outColor.a?
        # If we assume standard compositing, alpha_out = a1 + a0 * (1 - a1).
        # But the source doesn't specify alpha_out formula, just "divide by outColor.a".
        # We will assume outColor.a is the standard composite alpha.
        alpha_out = a1 + a0 * (1 - a1)
        if alpha_out == 0:
            return (0, 0, 0, 0)
        return (r_out / alpha_out, g_out / alpha_out, b_out / alpha_out, alpha_out)

    elif num_layers == 4:
        c3, a3 = premultiplied[3]
        c2, a2 = premultiplied[2]
        c1, a1 = premultiplied[1]
        c0, a0 = premultiplied[0]
        
        # lerp(color3, color2, alpha.z) -> color3 * (1 - a2) + color2 * a2
        r1 = c3[0] * (1 - a2) + c2[0] * a2
        g1 = c3[1] * (1 - a2) + c2[1] * a2
        b1 = c3[2] * (1 - a2) + c2[2] * a2
        
        # lerp(result1, color1, alpha.y) -> result1 * (1 - a1) + color1 * a1
        r2 = r1 * (1 - a1) + c1[0] * a1
        g2 = g1 * (1 - a1) + c1[1] * a1
        b2 = b1 * (1 - a1) + c1[2] * a1
        
        # lerp(result2, color0, alpha.x) -> result2 * (1 - a0) + color0 * a0
        r3 = r2 * (1 - a0) + c0[0] * a0
        g3 = g2 * (1 - a0) + c0[1] * a0
        b3 = b2 * (1 - a0) + c0[2] * a0
        
        # Compute total alpha for normalization
        # Standard alpha compositing for 4 layers:
        # a_out = a3 + a2*(1-a3) + a1*(1-a3)*(1-a2) + a0*(1-a3)*(1-a2)*(1-a1)
        a_out = a3 + a2 * (1 - a3) + a1 * (1 - a3) * (1 - a2) + a0 * (1 - a3) * (1 - a2) * (1 - a1)
        
        if a_out == 0:
            return (0, 0, 0, 0)
        
        return (r3 / a_out, g3 / a_out, b3 / a_out, a_out)
```

## Subtopics

### [SDF Underlay & Inner Underlay](5612-sdf-underlay-inner-underlay.md)
Renders a blurred, colored border either behind (underlay) or inside (inner underlay) the primary glyph face by sampling the SDF atlas at an offset UV coordinate. It differs by allowing the border to be offset in UV space independently of the glyph geometry and by supporting "inner" rendering. Choose it when you need a soft glow or inset shadow effect rather than a sharp outline.

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel, as it involves a fixed number of arithmetic operations (4-16 multiplications/additions) regardless of scene complexity.
- **Memory Complexity:** $O(1)$, requiring only registers for the color and alpha values.
- **Strengths:** Extremely fast; supports variable-width outlines via the SDF sampling step (which precedes this compositing); handles multiple layers cleanly.
- **Limitations:** Limited to 1, 2, or 4 layers; assumes pre-multiplied alpha handling; precision issues may arise if alpha values are very small or very large.

## Background
Originating from the Unity NanoVDB renderer implementation [S1, unofficial], this method is used for rendering high-performance text and icons using Signed Distance Fields. It leverages the hierarchical nature of SDF atlases to composite multiple visual layers (face, outline) efficiently.

## Sources
[S1] andersblomqvist/unity-nanovdb-renderer, https://github.com/andersblomqvist/unity-nanovdb-renderer, fetched 2026-10-27 (unofficial)
