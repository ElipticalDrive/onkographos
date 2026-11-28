# Transparency Compositing Formula

Also called: Alpha Blending Formula, Alpha Compositing (1-A)*B, Blend1 Compositing, C_out = C_in * (1 - α) + C * α, Over-Under Blend

## Overview
The Transparency Compositing Formula computes the resulting color when a foreground voxel blends with a background accumulation, typically used for over-compositing in volume rendering where the foreground alpha represents opacity [S2, unofficial]. It sets itself apart from sibling methods like [CTF Color Blending](14808-ctf-color-blending.md) by applying the standard recursive accumulation along rays rather than perceptual harmonization models [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   $C_{in}$: Incoming intensity/color from previous voxels along the ray [S1, unofficial].
*   $C(X_i)$: Shading/color calculated for the current voxel $X_i$, derived from the local gradient and lighting [S1, unofficial].
*   $\alpha(X_i)$: Opacity (light impermeability) of the current voxel, calculated from the underlying data value (e.g., CT value) [S1, unofficial].
*   $C_{out}$: Outgoing intensity/color after compositing the current voxel [S1, unofficial].

**Data Structures:**
The method operates on scalar or vector values representing color and opacity [G]. In grid-based implementations, values may be scaled by grid-specific multipliers and combined with a constant `ONE` of the grid's `ValueT` type [S2, unofficial].

**Formula:**
The standard compositing equation is:
$$ C_{out} = C_{in} \cdot (1 - \alpha(X_i)) + C(X_i) \cdot \alpha(X_i) $$

In grid-based implementations using multipliers:
$$ \text{out} = (\text{ONE} - \text{aMult} \cdot a) \cdot \text{bMult} \cdot b $$
Where `aMult` is a multiplier for grid A value (default 1.0) and `bMult` is a multiplier used in the grid formula [S2, unofficial].

**Steps:**
1.  Initialize accumulated color $C_{acc} = 0$ and accumulated opacity $\alpha_{acc} = 0$ [G].
2.  For each voxel $X_i$ along the ray:
    a. Sample color $C(X_i)$ and opacity $\alpha(X_i)$ [S1, unofficial].
    b. Compute transmission: $T = 1 - \alpha(X_i)$ [G].
    c. Update accumulated color: $C_{acc} = C_{acc} \cdot T + C(X_i) \cdot \alpha(X_i)$ [S1, unofficial].
    d. Update accumulated opacity: $\alpha_{acc} = \alpha_{acc} + \alpha(X_i) \cdot (1 - \alpha_{acc})$ [G].
    e. If $\alpha(X_i) = 0$ (e.g., hitting light-impermeable tissue like bone), terminate recursion [S1, unofficial].
3.  Return $C_{acc}$ as $C_{out}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transparency_compositing(ray_voxels):
    c_acc = 0.0
    alpha_acc = 0.0
    for voxel in ray_voxels:
        c_v = voxel.color
        a_v = voxel.opacity
        if a_v == 0.0:
            break
        t = 1.0 - a_v
        c_acc = c_acc * t + c_v * a_v
        alpha_acc = alpha_acc + a_v * (1.0 - alpha_acc)
    return c_acc
```

**Parameters:**
*   `aMult`: Multiplier for grid A value. Default 1.0 [S2, unofficial].
*   `ONE`: Constant value 1.0 of the grid's ValueT type [S2, unofficial].

**Edge Cases and Pitfalls:**
*   Precision: Floating-point precision errors can accumulate during recursive application along long rays [G].
*   Boundaries: The formula assumes the ray enters the volume with $C_{in}=0$ and $\alpha_{in}=0$ [G].
*   Implementation Gap: Some raycasters do not implement this transparency formula, offering only images without transparency [S1, unofficial].

## Subtopics

### Additive Color Blending
A simple additive blending mode for combining brush color and canvas color, primarily used in 3D painting applications rather than volume rendering compositing [S3, unofficial]. It differs from the standard over-operator by defining output opacity as the sum of opacities and modulating color mixing based on relative alpha weights [S3, unofficial].
**Formula:**
Output Color: $C_{out} = B_{rgb} \cdot m + (1 - m) \cdot C_{rgb}$ [S3, unofficial].
Output Opacity: $C_{out, \alpha} = B_{\alpha} + C_{\alpha}$ [S3, unofficial].
Where $m = B_{\alpha} / (B_{\alpha} + C_{\alpha})$, $B_{rgb}, B_{\alpha}$ are brush RGB and alpha, and $C_{rgb}, C_{\alpha}$ are canvas RGB and alpha [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Does not account for subsurface scattering or complex light transport; requires accurate opacity estimation [G].
*   **When to use:** Standard volume visualization where translucency is desired [G].
*   **When not to use:** When additive mixing is preferred (e.g., certain painting tools) or when performance constraints prevent recursive accumulation [G].

## Background
The formula is widely adopted in computer graphics for alpha blending. In volume rendering, it accumulates color and opacity along rays cast through volumetric data [S1, unofficial]. Grid-based implementations may apply multipliers (`aMult`, `bMult`) to grid values [S2, unofficial]. The recursive application along rays until opacity reaches 0 is a common optimization, though not all implementations support it [S1, unofficial].

## Sources
[S1] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
[S2] NVIDIA-Omniverse/ext-openvdb, https://github.com/NVIDIA-Omniverse/ext-openvdb, fetched 2026-10-27 (unofficial)
[S3] CanvoX: High-resolution VR Painting in Large Volumetric Canvas, https://archive.org/details/arxiv-1704.02724, fetched 2026-10-29 (unofficial)
