# Ray Marching with SDFs and Soft Accumulation

Also called: Jellyfish Ray Marching with Soft Accumulation, Soft Accumulation for Thin Geometry

## Overview
A rendering technique for hair-thin geometry (such as jellyfish tentacles) that prevents flickering by replacing hard surface intersection with soft accumulation. Instead of stopping at the first hit, each ray march step adds a contribution based on the distance to the surface, fading in smoothly via an exponential falloff. This sets it apart from standard SDF ray marching, which typically terminates on hard intersection, and from [Aggressive Ray Marching with SDF](2326-aggressive-ray-marching-with-sdf.md), which focuses on step size optimization rather than anti-aliasing thin strands.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origin $o$, ray direction $d$, time $t$, phase offset $\phi$, scene parameters (bell radius, tentacle length, school count), and SDF functions for the bell ($sdBell$), tentacles ($sdTentacles$), and arms ($sdArms$).
*   **Outputs:** Final color $col$ for the pixel.

**Data Structures:**
*   **Ray State:** `travel` (float, accumulated distance), `trans` (float, transparency, initialized to 1.0), `rad` (vec3, accumulated radial color), `strands` (float, accumulated strand contribution).
*   **Scene Constants:** `MARCH_STEPS` (120), `TENT_LEN` (3.0), `ARM_LEN` (1.5), `SCHOOL` (5), `GLOW`, `HOT` (lighting intensity parameters).

**Step-by-Step Method:**

1.  **Initialization:**
    For each jellyfish in the school:
    *   Compute the jellyfish center with sinusoidal drift.
    *   Transform the ray origin $o$ relative to the center: $o = (ro - centre) / data.w$.
    *   Apply tilt rotations to $o$:
        $$o_{xy} = \text{rot}(\text{tilt}.x) \cdot o_{xy}$$
        $$o_{zy} = \text{rot}(\text{tilt}.y) \cdot o_{zy}$$
    *   Compute ray-sphere intersection for marching bounds using the bell center $bo = o - \text{vec3}(0.0, -1.4, 0.0)$:
        $$b = \text{dot}(bo, d)$$
        $$\text{disc} = b^2 - \text{dot}(bo, bo) + 2.7^2$$
        $$t_{max} = -b + \sqrt{\text{disc}}$$
    *   Initialize marching state:
        $$pc = \text{pulse}(t \cdot 0.42 + \phi \cdot 0.37)$$
        $$\text{travel} = \max(-b - \text{sq}, 0.0) + \text{dither}$$
        $$\text{trans} = 1.0$$
        $$\text{strands} = 0.0$$
        $$\text{rad} = \text{vec3}(0.0)$$

2.  **Ray Marching Loop:**
    Loop $s$ from $0$ to `MARCH_STEPS` - 1:
    *   **Termination Check:** If $\text{travel} > t_{max}$ or $\text{trans} < 0.05$, break.
    *   **Position:** $p = o + d \cdot \text{travel}$.
    *   **SDF Evaluation:**
        $$db = sdBell(p, pc, t, \phi)$$
        $$dt = sdTentacles(p, pc, t, \phi)$$
        $$da = sdArms(p, t, \phi)$$
    *   **Step Size:**
        $$\text{stride} = \text{clamp}(\min(db, \min(dt, da)) \cdot 0.70, 0.010, 0.18)$$
    *   **Soft Accumulation for Strands:**
        Calculate taper factor:
        $$\text{taper} = 1.0 - \text{smoothstep}(0.55, 1.05, -p.y / \text{TENT\_LEN})$$
        Add tentacle contribution:
        $$\text{strands} += \exp(-\max(dt, 0.0) \cdot 95.0) \cdot \text{stride} \cdot 8.5 \cdot \text{taper}$$
        Add arm contribution:
        $$\text{strands} += \exp(-\max(da, 0.0) \cdot 45.0) \cdot \text{stride} \cdot 3.0$$
    *   **Bell Surface Shading:**
        If $db < 0.012$:
        *   Compute normal $n$ via finite differences with $e = \text{vec2}(0.0035, 0.0)$.
        *   Compute Fresnel term:
            $$\text{fres} = \text{pow}(1.0 - |\text{dot}(n, -d)|, 2.4)$$
        *   Compute lighting angle:
            $$\text{lam} = 0.35 + 0.65 \cdot |\text{dot}(n, \text{normalize}(\text{vec3}(-0.35, 1.0, 0.45)))|$$
        *   Compute rib pattern:
            $$\text{ribs} = \text{pow}(0.5 + 0.5 \cdot \cos(16.0 \cdot \text{atan}(p.z, p.x)), 14.0)$$
        *   Compute opacity:
            $$\text{op} = 0.075 + 0.42 \cdot \text{fres} + \text{ribs} \cdot 0.12$$
        *   Accumulate radial color:
            $$\text{rad} += \text{trans} \cdot \text{op} \cdot (\text{GLOW} \cdot (0.75 + 1.15 \cdot \text{lam}) + \text{HOT} \cdot (\text{fres} \cdot 1.30 + \text{ribs} \cdot 0.60))$$
        *   Update transparency:
            $$\text{trans} *= 1.0 - \text{op}$$
        *   Force small step for bell surface detail:
            $$\text{stride} = 0.026$$
    *   **Advance:**
        $$\text{travel} += \text{stride}$$

3.  **Final Compositing:**
    *   Apply fog:
        $$\text{fog} = \exp(-0.055 \cdot \max(\text{length}(ro - centre) - 3.0, 0.0))$$
    *   Add accumulated color:
        $$col += (\text{rad} + \text{GLOW} \cdot \text{strands} \cdot 0.16 + \text{HOT} \cdot \text{strands}^2 \cdot 0.006) \cdot \text{fog}$$

4.  **Post-Processing:**
    *   Tone mapping:
        $$col = \text{vec3}(1.0) - \exp(-col \cdot 1.85)$$
    *   Black point adjustment:
        $$col = \max(col - \text{vec3}(0.0175, 0.0160, 0.0060), \text{vec3}(0.0))$$
    *   Gamma correction:
        $$\text{output} = \text{pow}(col, \text{vec3}(1.0 / 2.2))$$

**Pulse Function:**
The swim cycle pulse function $pulse(x)$ models a quick contraction followed by a long relaxed glide:
$$pulse(x) = \text{fract}(x)$$
If $x < 0.28$:
$$\text{return } \text{smoothstep}(0.0, 1.0, x / 0.28)$$
Else:
$$\text{return } 1.0 - \text{smoothstep}(0.0, 1.0, (x - 0.28) / 0.72)$$

**Pitfalls:**
*   Hard intersection of thin geometry causes flickering as strands slip between sample points. Soft accumulation solves this by fading in smoothly [S1, unofficial].
*   Hair-thin tentacles and arms alias severely across frames if rendered with hard SDF hits. Solved by soft accumulation using exponential falloff based on distance to SDF surface, multiplied by stride and a taper factor that fades near the base [S1, unofficial].

## Subtopics

### [Butterfly Ray Marching with SDFs](5009-butterfly-ray-marching-with-sdfs.md)
Renders a swarm of butterflies by combining SDFs for individual anatomical parts with ray marching in a transformed local coordinate space to handle dynamic wing flapping and orientation. It sets itself apart from standard SDF ray marching by using a non-rigid "un-bend" transform to animate wings and a specific stability clamping strategy for step sizes, building on the soft accumulation techniques of [Ray Marching with SDFs and Soft Accumulation](5001-ray-marching-with-sdfs-and-soft-accumulation.md).

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of march steps per ray ($O(MARCH\_STEPS)$) and the number of objects in the school ($O(SCHOOL)$).
*   **Memory Complexity:** Constant per ray, storing only a few floating-point accumulators.
*   **Performance:** Roughly one minute of rendering per 10 seconds of output on integrated graphics [S1, unofficial]. Single frames are near-instant, allowing for rapid iteration.
*   **Strengths:** Eliminates flickering and aliasing artifacts common in rendering hair-thin geometry with hard SDF intersections. Provides smooth, fluid motion for dynamic thin structures.
*   **Limitations:** Requires careful tuning of exponential falloff constants (e.g., 95.0 for tentacles) to balance visibility and performance. The soft accumulation approach may result in slightly lower contrast or "ghosting" if not properly attenuated by transparency.
*   **When to Use:** Ideal for rendering thin, strand-like geometry (hair, tentacles, grass) where traditional hard intersection causes temporal instability.
*   **When Not to Use:** Not suitable for solid, opaque objects where precise surface definition is required without transparency artifacts.

## Background
Originating from the `manim-glsl-jellyfish` project by el-bakkali, this technique was developed to render a school of jellyfish using signed distance functions for the bell, tentacles, and arms [S1, unofficial]. It addresses the specific challenge of animating thin, dynamic structures without the visual artifacts associated with discrete ray marching steps. The method is implemented in GLSL shaders and integrated with the Manim animation engine for timeline control [S1, unofficial].

## Sources
[S1] el-bakkali/manim-glsl-jellyfish, https://github.com/el-bakkali/manim-glsl-jellyfish, fetched 2026-10-26 (unofficial)
