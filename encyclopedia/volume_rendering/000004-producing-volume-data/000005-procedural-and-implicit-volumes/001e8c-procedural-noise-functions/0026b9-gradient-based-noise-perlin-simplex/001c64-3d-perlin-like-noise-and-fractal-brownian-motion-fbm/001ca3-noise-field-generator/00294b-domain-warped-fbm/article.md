# Domain Warped FBM

Also called: 2D Clouds, Domain Warping, Inigo Quilez Domain Warping

## Overview
Domain Warped FBM is a procedural noise technique where the output of a noise function is fed back as a coordinate offset to distort the sampling domain, creating organic, fluid-like patterns [S1]. It differs from standard FBM by introducing non-linear spatial distortion that mimics turbulent flow, whereas standard FBM simply layers noise at different scales [S1].

## How it works
**Inputs and Outputs:**
- Input: 2D coordinate `q` (or `uv`), time `t` (optional), warp strength `WARP_STRENGTH` [S1].
- Output: Scalar value `f` representing the noise intensity [S1].

**Data Structures:**
- `fbm(p)`: Multi-octave noise summation returning a scalar [S1].
- `fbm4(p)`: 4-octave FBM returning a scalar [S1].
- `fbm4_2(p)`: Returns a `vec2` from two 4-octave noise sums with phase offsets [S1].
- `fbm6_2(p)`: Returns a `vec2` from two 6-octave noise sums [S1].

**Parameters:**
- `WARP_STRENGTH`: Controls the intensity of the distortion [S1].
- `time`: A scalar for animation, often scaled by small factors (e.g., 0.1) [S1].

**Steps:**
The method follows a cascade structure where noise outputs offset subsequent sampling coordinates [S1]. The classic three-level cascade proceeds as follows [S1]:

1. **Compute Base Warp Field (`o`):**
   Calculate two orthogonal noise values to form a 2D offset vector `o` [S1].
   $$ o = 0.5 + 0.5 \times \text{fbm4\_2}(q) $$
   Where $\text{fbm4\_2}(q)$ returns:
   $$ \text{vec2}(\text{fbm4}(q + \text{vec2}(1.0)), \text{fbm4}(q + \text{vec2}(6.2))) $$

2. **Compute Secondary Warp Field (`n`):**
   Calculate a second 2D offset vector `n` using the result `o` [S1].
   $$ n = \text{fbm6\_2}(4.0 \times o) $$

3. **Compute Warped Coordinate (`p`):**
   Offset the original coordinate $q$ by the secondary warp field `n` [S1].
   $$ p = q + 2.0 \times n + \text{vec2}(1.0) $$

4. **Compute Final Noise Value (`f`):**
   Sample the final noise field using the warped coordinate $p$ [S1].
   $$ f = 0.5 + 0.5 \times \text{fbm4}(2.0 \times p) $$

5. **Modulate Output:**
   Mix the final value with a cubic function weighted by the magnitude of the secondary warp field `n` [S1].
   $$ f = \text{mix}(f, f^3 \times 3.5, f \times |n.x|) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def domain_warped_fbm(q):
    # Step 1: Compute base warp field o
    o_x = fbm4(q + vec2(1.0))
    o_y = fbm4(q + vec2(6.2))
    o = vec2(0.5 + 0.5 * o_x, 0.5 + 0.5 * o_y)

    # Step 2: Compute secondary warp field n
    n = fbm6_2(4.0 * o)

    # Step 3: Compute warped coordinate p
    p = q + 2.0 * n + vec2(1.0)

    # Step 4: Compute final noise value f
    f = 0.5 + 0.5 * fbm4(2.0 * p)

    # Step 5: Modulate output
    weight = f * abs(n.x)
    f = mix(f, f * f * f * 3.5, weight)

    return f

# Standard FBM helper [G]
def fbm(p, octaves):
    val = 0.0
    amp = 0.5
    freq = 1.0
    for _ in range(octaves):
        val += amp * noise(p * freq)
        amp *= 0.5
        freq *= 2.0
    return val

def fbm4(p):
    return fbm(p, 4)

def fbm6_2(p):
    # Returns two orthogonal 6-octave noise sums [G]
    return vec2(fbm(p, 6), fbm(p + vec2(1.0), 6))

def vec2(x, y):
    return (x, y)

def mix(a, b, t):
    return a + t * (b - a)

def abs(x):
    return x if x >= 0 else -x

def noise(p):
    # Base gradient noise implementation [G]. Not given in the sources.
    pass
```

**Edge Cases and Pitfalls:**
- **Precision:** The cascade involves multiple nested noise evaluations [S1]. Not given in the sources.
- **Boundaries:** The addition of constants (e.g., `+ 1.0`, `+ vec2(1.0)`) shifts the sampling domain [S1]. Not given in the sources.
- **Degenerate Input:** If `WARP_STRENGTH` is zero, the method degrades to standard FBM [S1].

## Operating characteristics
- **Time Complexity:** Proportional to the total number of FBM evaluations across the cascade levels [S1]. Not given in the sources.
- **Memory Complexity:** $O(1)$ auxiliary memory, as it relies on the noise function's internal state [G].
- **Strengths:** Produces highly organic, fluid-like structures suitable for clouds, smoke, and lava [S1].
- **Limitations:** Not given in the sources.

## Background
Domain Warped FBM is a variant of Fractal Brownian Motion that introduces non-linear spatial distortion to the noise field [S1]. It is implemented in shader programming for procedural textures [S1].

## Sources
[S1] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
