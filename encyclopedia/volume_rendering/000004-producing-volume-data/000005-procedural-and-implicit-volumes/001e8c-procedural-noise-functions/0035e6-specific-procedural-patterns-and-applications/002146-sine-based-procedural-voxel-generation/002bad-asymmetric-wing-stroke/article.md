# Asymmetric Wing Stroke

Also called: Asymmetric Swim Cycle, Non-sinusoidal Flapping

## Overview
The Asymmetric Wing Stroke is a time-dependent parametric function $p(t)$ that generates non-sinusoidal, biologically plausible flapping or swimming motions for procedural volume rendering. It solves the "robotic" appearance of symmetric sine waves by mimicking the "snap and coast" kinematics of real organisms (jellyfish bells or insect wings), where the power stroke is rapid and the recovery stroke is slow. It differs from standard sinusoidal oscillation by enforcing a specific duty cycle asymmetry, and from rigid-body animation by providing a continuous scalar field driver that can be mapped to deformation parameters like radius or span angle.

## How it works
**Inputs and Outputs:**
- Input: Normalized time $x \in [0, 1)$ representing the position within one full cycle.
- Output: A scalar value $p(x) \in [0, 1]$ representing the stroke amplitude.

**Data Structures:**
No persistent data structures are required. The function is stateless and pure.

**Parameters:**
- **Duty Cycle:** The fraction of the period spent in the contraction/downstroke phase.
  - Jellyfish (Swim Cycle): 28% contraction, 72% glide [S1, unofficial].
  - Butterfly (Wing Stroke): 36% downstroke, 64% recovery [S2, unofficial].
- **FLAP_HZ:** Wingbeat frequency in Hertz (used to compute $x$ from absolute time $t$). Value: 1.35 [S2, unofficial].
- **TIP_LAG:** Phase delay per unit of span to simulate wing flexion. Value: 0.135 [S2, unofficial].

**Algorithm:**
The function uses `smoothstep` to create a smooth, continuous transition between the two phases. `smoothstep(edge0, edge1, x)` is defined as $t = \max(0, \min(1, (x - edge0) / (edge1 - edge0)))$; $t * t * (3 - 2 * t)$.

1. Normalize the input time $t_{abs}$ to the cycle range $[0, 1)$ using the modulo operator or fract function: $x = t_{abs} \pmod 1$.
2. Determine the phase threshold $\tau$ based on the organism model (0.28 for jellyfish, 0.36 for butterflies).
3. If $x < \tau$ (power stroke):
   - Compute normalized progress: $u = x / \tau$.
   - Result: $p(x) = \text{smoothstep}(0.0, 1.0, u)$.
4. If $x \ge \tau$ (recovery/coast):
   - Compute normalized progress: $u = (x - \tau) / (1.0 - \tau)$.
   - Result: $p(x) = 1.0 - \text{smoothstep}(0.0, 1.0, u)$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def smoothstep(edge0, edge1, x):
    # Clamp x to [0, 1] relative to the edge range
    t = max(0.0, min(1.0, (x - edge0) / (edge1 - edge0)))
    return t * t * (3.0 - 2.0 * t)

def asymmetric_stroke(x, threshold):
    """
    Computes the asymmetric stroke amplitude.
    x: normalized time in [0, 1)
    threshold: fraction of cycle for the 'fast' phase (0.28 or 0.36)
    """
    # Ensure x is in [0, 1)
    x = x % 1.0
    
    if x < threshold:
        # Fast phase: 0 -> 1
        progress = x / threshold
        return smoothstep(0.0, 1.0, progress)
    else:
        # Slow phase: 1 -> 0
        progress = (x - threshold) / (1.0 - threshold)
        return 1.0 - smoothstep(0.0, 1.0, progress)
```

**Application to Deformation:**
In the jellyfish model, this scalar $p(t)$ drives radius $R$ and height $H$ in opposite directions to conserve volume/shape:
$$R = 1.00 + 0.13\,p(t)$$
$$H = 0.80 - 0.22\,p(t)$$
[S1, unofficial]

In the butterfly model, the stroke drives the flap angle, with a phase lag applied across the wing span to simulate flexibility:
$$\text{phase} = t \cdot \text{FLAP\_HZ} + \text{phase\_offset} - \text{span} \cdot \text{TIP\_LAG}$$
[S2, unofficial]

**Edge Cases and Pitfalls:**
- **Discontinuity:** The derivative is continuous at the threshold $\tau$ because `smoothstep` has zero derivative at 0 and 1. However, the second derivative is not continuous, which may cause minor artifacts in high-order shading derivatives if not smoothed further.
- **Boundary Conditions:** Ensure $x$ is strictly less than 1.0 before evaluation to avoid division by zero or incorrect phase selection at the cycle wrap-around.

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel/pixel. The calculation involves basic arithmetic and a cubic polynomial evaluation.
- **Memory Complexity:** $O(1)$. No auxiliary data structures.
- **Strengths:** Produces natural-looking motion with minimal computational cost. Avoids the "robotic" symmetry of sine waves. Easy to tune via the threshold parameter.
- **Limitations:** Does not model physical elasticity or fluid dynamics directly; it is a kinematic approximation. The "snap" may appear too sharp if the smoothstep range is not sufficient for the visual style.
- **When to use:** Procedural animation of biological volumes (jellyfish, insects) where performance is critical and physical simulation is unnecessary.
- **When not to use:** When physical accuracy of fluid-structure interaction is required, or when a perfectly sinusoidal motion is desired.

## Background
The concept originates from procedural animation techniques in GLSL shaders for real-time graphics. The "Asymmetric Swim Cycle" was popularized in the context of jellyfish simulation by el-bakkali, who noted that real jellyfish spend 28% of the cycle contracting and 72% gliding, contrasting with the equal-time symmetry of sine waves [S1, unofficial]. The "Asymmetric Wing Stroke" was similarly applied to butterfly wing flapping, using a 36%/64% split to mimic the quick downstroke and slower recovery of real insects [S2, unofficial]. These methods are part of a broader trend in procedural voxel and raymarched rendering to inject "life" into static distance fields through time-dependent coefficient modulation.

## Sources
[S1] el-bakkali/manim-glsl-jellyfish, https://github.com/el-bakkali/manim-glsl-jellyfish, fetched 2026-10-26 (unofficial)
[S2] el-bakkali/manim-glsl-butterflies, https://github.com/el-bakkali/manim-glsl-butterflies, fetched 2026-10-26 (unofficial)
