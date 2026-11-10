# Asymmetric Wing Stroke

Also called: Asymmetric Swim Cycle, Non-sinusoidal Flapping

## Overview
The Asymmetric Wing Stroke is a time-dependent function $p(t)$ that generates non-sinusoidal flapping or swimming motions for procedural volume rendering [S2, unofficial]. It avoids the mechanical appearance of symmetric sine waves by mimicking biological "snap and coast" kinematics, where the power stroke is rapid and the recovery stroke is slow [S1, unofficial]. It builds on [Sine-based procedural voxel generation](2477-sine-based-procedural-voxel-generation.md) and relates to [Procedural Tree Voxel Generation](4445-procedural-tree-voxel-generation.md) and [Simple Heightmap Voxel Generation](4649-simple-heightmap-voxel-generation.md).

## How it works
**Inputs and Outputs:**
- Input: Normalized cycle time $x \in [0, 1)$ [S1, unofficial][S2, unofficial].
- Output: Scalar amplitude $p(x) \in [0, 1]$ [S1, unofficial][S2, unofficial].

**Parameters:**
- Duty cycle threshold $\tau$: 0.28 for jellyfish contraction/glide [S1, unofficial]; 0.36 for butterfly downstroke/recovery [S2, unofficial].
- FLAP_HZ: Wingbeat frequency in Hertz. Value: 1.35 [S2, unofficial].
- TIP_LAG: Phase delay per unit of span. Value: 0.135 [S2, unofficial].

**Algorithm:**
1. Normalize absolute time $t_{\text{abs}}$ to $[0, 1)$ via $x = t_{\text{abs}} \pmod 1$ [S1, unofficial][S2, unofficial].
2. If $x < \tau$, compute $u = x / \tau$ and $p(x) = \text{smoothstep}(0.0, 1.0, u)$ [S1, unofficial][S2, unofficial].
3. If $x \ge \tau$, compute $u = (x - \tau) / (1.0 - \tau)$ and $p(x) = 1.0 - \text{smoothstep}(0.0, 1.0, u)$ [S1, unofficial][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def smoothstep(edge0, edge1, x):
    t = max(0.0, min(1.0, (x - edge0) / (edge1 - edge0)))
    return t * t * (3.0 - 2.0 * t)

def asymmetric_stroke(x, threshold):
    x = x % 1.0
    if x < threshold:
        progress = x / threshold
        return smoothstep(0.0, 1.0, progress)
    else:
        progress = (x - threshold) / (1.0 - threshold)
        return 1.0 - smoothstep(0.0, 1.0, progress)
```

**Application to Deformation:**
The jellyfish model drives radius $R$ and height $H$ in opposite directions: $R = 1.00 + 0.13\,p(t)$ and $H = 0.80 - 0.22\,p(t)$ [S1, unofficial]. The butterfly model drives the flap angle with a span-dependent phase lag: $\text{phase} = t \cdot \text{FLAP\_HZ} + \text{phase\_offset} - \text{span} \cdot \text{TIP\_LAG}$ [S2, unofficial].

**Edge Cases and Pitfalls:**
- The `smoothstep` function guarantees $C^1$ continuity at $\tau$ and the cycle boundaries, but $C^2$ is discontinuous, which may affect high-order shading derivatives [G].
- The modulo operation handles cycle wrap-around safely without division by zero [G].

## Operating characteristics
Not given in the sources.

## Background
The technique originates from GLSL shader implementations for procedural animation [S1, unofficial][S2, unofficial]. The "Asymmetric Swim Cycle" (28% contraction, 72% glide) was documented for jellyfish simulation to contrast with the equal-time symmetry of sine waves [S1, unofficial]. The "Asymmetric Wing Stroke" (36% downstroke, 64% recovery) was applied to butterfly flapping to replicate quick power strokes and slower recoveries [S2, unofficial]. Both methods are used in procedural voxel and raymarched rendering to animate static distance fields through time-dependent coefficient modulation [S1, unofficial][S2, unofficial].

## Sources
[S1] el-bakkali/manim-glsl-jellyfish, https://github.com/el-bakkali/manim-glsl-jellyfish, fetched 2026-10-26 (unofficial)
[S2] el-bakkali/manim-glsl-butterflies, https://github.com/el-bakkali/manim-glsl-butterflies, fetched 2026-10-26 (unofficial)
