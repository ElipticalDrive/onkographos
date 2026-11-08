# Fog Animation
Also called: Solid Space Animation: Moving Point

## Overview
Fog Animation generates the appearance of moving gaseous volumes by perturbing the spatial coordinates of the evaluation point before sampling a static turbulence function, effectively moving the point through a "solid space" texture [S1, unofficial]. Unlike methods that animate the noise function itself, this approach creates motion by shifting the sampling domain, producing upward-swirling gas movement suitable for real-time fog and smoke [S1, unofficial]. It builds on Procedural Volume Rendering with Noise Perturbation [1370-procedural-volume-rendering-with-noise-perturbation.md] by applying coordinate perturbation specifically to transparency-based fog effects [S1, unofficial].

## How it works
**Inputs:**
- `pnt`: 3D point in world space ($x, y, z$).
- `frame_num`: Integer current animation frame.
- `transp`: Output pointer to a float representing transparency.

**Parameters:**
- `SWIRL_FRAMES`: Integer controlling the period of the helical swirl. Typical value: 126 [S1, unofficial].
- `SWIRL_AMOUNT`: Float controlling the speed of the swirling effect. Typical value: $2\pi/126$ (implied by modulo arithmetic for a full rotation) [S1, unofficial].
- `RAD1`: Float, radius of the helix in the X direction. Typical value: 0.5 [S1, unofficial].
- `RAD2`: Float, radius of the helix in the Z direction. Typical value: 2.0 [S1, unofficial].
- `DOWN_AMOUNT`: Float, downward movement speed per frame. Typical value: 0.01 [S1, unofficial].
- `turbulence(p, scale)`: Function returning a scalar noise value at point `p` with frequency `scale` [S1, unofficial].
- `noise_it(p)`: Function returning a scalar noise value at point `p` [S1, unofficial].

**Data Structures:**
- `direction`: 3D vector ($x, y, z$) representing the final sampling coordinate [S1, unofficial].
- `cyl`: 3D vector ($x, y, z$) representing cylindrical helical offset [S1, unofficial].
- `tmp`: Float scalar used for intermediate noise storage [S1, unofficial].

**Steps:**
1. **Perturb Input Point:** Modify the input point `pnt` to introduce local turbulence and drift [S1, unofficial].
   - $pnt.x \leftarrow pnt.x + 2.0 + \text{turbulence}(pnt, 0.1)$ [S1, unofficial]
   - $tmp \leftarrow \text{noise\_it}(pnt)$ [S1, unofficial]
   - $pnt.y \leftarrow pnt.y + 4.0 + tmp$ [S1, unofficial]
   - $pnt.z \leftarrow pnt.z - 2.0 - tmp$ [S1, unofficial]

2. **Calculate Helical Path Components:** Determine the position on a helix based on the current frame [S1, unofficial].
   - $\theta \leftarrow (\text{frame\_num} \pmod{\text{SWIRL\_FRAMES}}) \times \text{SWIRL\_AMOUNT}$ [S1, unofficial]
   - $cyl.x \leftarrow \text{RAD1} \times \cos(\theta)$ [S1, unofficial]
   - $cyl.z \leftarrow \text{RAD2} \times \sin(\theta)$ [S1, unofficial]

3. **Calculate Direction Vector:** Combine the perturbed point with the helical offset and a downward drift [S1, unofficial].
   - $direction.x \leftarrow pnt.x + cyl.x$ [S1, unofficial]
   - $direction.y \leftarrow pnt.y - (\text{frame\_num} \times \text{DOWN\_AMOUNT})$ [S1, unofficial]
   - $direction.z \leftarrow pnt.z + cyl.z$ [S1, unofficial]

4. **Calculate Transparency:** Sample turbulence at the direction vector and apply non-linear transformations to map the result to a transparency value [S1, unofficial].
   - $t \leftarrow \text{turbulence}(direction, 0.015)$ [S1, unofficial]
   - $t \leftarrow 1.0 - (t \times t \times 0.275)$ [S1, unofficial]
   - $\text{transp} \leftarrow t \times t \times t$ [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Parameters from source [S1, unofficial]
SWIRL_FRAMES = 126
SWIRL_AMOUNT = 2.0 * math.pi / SWIRL_FRAMES
RAD1 = 0.5
RAD2 = 2.0
DOWN_AMOUNT = 0.01

def turbulence(p, scale):
    """Returns a scalar turbulence value at point p with frequency scale."""
    pass  # Implementation depends on available noise library

def noise_it(p):
    """Returns a scalar noise value at point p."""
    pass  # Implementation depends on available noise library

class Vector3:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

def fog_animation(pnt, frame_num):
    # Step 1: Perturb input point
    pnt_x = pnt.x + 2.0 + turbulence(pnt, 0.1)
    tmp = noise_it(pnt)
    pnt_y = pnt.y + 4.0 + tmp
    pnt_z = pnt.z - 2.0 - tmp

    # Step 2: Calculate helical path components
    theta = (frame_num % SWIRL_FRAMES) * SWIRL_AMOUNT
    cyl_x = RAD1 * math.cos(theta)
    cyl_z = RAD2 * math.sin(theta)

    # Step 3: Calculate direction vector
    direction_x = pnt_x + cyl_x
    direction_y = pnt_y - (frame_num * DOWN_AMOUNT)
    direction_z = pnt_z + cyl_z

    # Step 4: Calculate transparency
    t = turbulence(Vector3(direction_x, direction_y, direction_z), 0.015)
    t = 1.0 - (t * t * 0.275)
    transp = t * t * t

    return transp
```

**Edge Cases and Pitfalls:**
- **Precision:** The cubic mapping of transparency ($t^3$) can result in very low values for small $t$, potentially causing fog to disappear if the turbulence amplitude is low [S1, unofficial].
- **Boundary Conditions:** The modulo operation on `frame_num` ensures the swirl repeats every `SWIRL_FRAMES` [S1, unofficial]. If `frame_num` is negative, ensure the modulo operator behaves correctly for the target language (e.g., Python's `%` vs C++'s `%`) [G].
- **Noise Function Dependency:** The method relies on `noise_it` and `turbulence` [S1, unofficial].

## Subtopics

### Marble Forming with Heat
A variant that animates marble textures by simulating heating and cooling effects [S1, unofficial]. It calculates a `glow_percent` based on `frame_num`, `start_frame`, `end_frame`, and `heat_length` [S1, unofficial]. If outside the heat range, `glow_percent=0` [S1, unofficial]. In the start heating phase, `glow_percent = 1.0 - ease(((start_frame+heat_length/2-frame_num)/heat_length), 0.4, 0.6)` [S1, unofficial]. In the end cooling phase, `glow_percent = ease(((frame_num-(end_frame-heat_length/2))/heat_length), 0.4, 0.6)` [S1, unofficial]. Otherwise, `glow_percent=1.0` [S1, unofficial]. Turbulence is applied with ease-in/ease-out: `turb_percent=ease(turb_percent, 0.3, 0.7)` [S1, unofficial]. The color is computed as `marble_color(sin((pnt.y + turb_percent*3.0*turbulence(pnt, .0125) - displacement)*M_PI))` and blended with a glow color (R:1.0, G:0.4, B:0.8) weighted by `0.5*glow_percent` [S1, unofficial].

### Moving Marble
A variant that moves the point along a helical path to create a swirling marble effect [S1, unofficial]. It calculates a displacement based on `frame_num` relative to `start_frame` and `end_frame` [S1, unofficial]. A helical path is computed using `theta = (frame_num%SWIRL_FRAMES)*SWIRL_AMOUNT`, with radii `RAD1` and `RAD2`, and downward drift `DOWN_AMOUNT` [S1, unofficial]. Randomness is added via `fast_noise(pnt)` [S1, unofficial]. The final color is `marble_color(sin((pnt.y + 3.0*turbulence(direction, .0125))*M_PI))` where `direction` is the sum of `pnt` and the helical path [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, assuming constant-time noise evaluation [S1, unofficial].
- **Memory Complexity:** $O(1)$, as it does not require storing volume data or large lookup tables, only static parameters [S1, unofficial].
- **Performance:** Suitable for real-time rendering due to its procedural nature and lack of ray-marching or grid traversal [S1, unofficial].
- **Strengths:** Simple to implement, low memory footprint, produces organic, swirling motion without animating the noise field itself [S1, unofficial].
- **Limitations:** The motion is constrained to the specific helical and drift patterns defined; less flexible than general noise animation [S1, unofficial].

## Background
The method originates from procedural modeling research on gases, fluids, and textures [S1, unofficial]. It was introduced as a technique for animating solid space functions to create realistic fog and smoke effects by moving the evaluation point through a static texture space [S1, unofficial]. The approach is similar to Gardner's cloud technique but uses turbulence for transparency control instead of Fourier synthesis [S1, unofficial].

## Sources
[S1] Procedural Modeling, Animation, and Rendering of Gases, Fluids, and Textures, https://www.researchgate.net/profile/David-Ebert-4/publication/2786953_Procedural_Modeling_Animation_and_Rendering_of_Gases_Fluids_and_Textures/links/00b7d51a73e42a3c4e000000/Procedural-Modeling-Animation-and-Rendering-of-Gases-Fluids-and-Textures.pdf, fetched 2026-10-31 (unofficial)
