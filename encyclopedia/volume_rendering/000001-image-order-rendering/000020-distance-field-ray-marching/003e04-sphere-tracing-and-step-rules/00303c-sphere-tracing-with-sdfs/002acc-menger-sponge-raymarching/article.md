# Menger Sponge Raymarching
Also called: Menger Journey, Menger Tower, Menger sponge

## Overview
Menger Sponge Raymarching is a specific application of sphere tracing that renders the Menger sponge fractal by evaluating a recursive Signed Distance Function (SDF) [S1, unofficial]. It differs from general SDF rendering by employing a domain repetition technique (modulo arithmetic) combined with iterative scaling to generate the fractal's self-similar voids, rather than relying on a fixed set of Boolean unions [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Camera position $\mathbf{prp}$, ray direction $\mathbf{scp}$, pixel coordinates, and fractal iteration count $N$ (typically 3) [S1, unofficial].
- **Output:** Fragment color $\mathbf{c}$ and alpha [S1, unofficial].

**Data Structures:**
- The SDF evaluation returns a `vec2` (or struct) containing:
  - `x`: The signed distance to the nearest surface.
  - `y`: A material ID (e.g., 0 for floor, 1 for sponge) [S1, unofficial].

**Core SDF Formulas:**
1.  **Box SDF:** For a box centered at origin with half-extents $\mathbf{b}$:
    $$ \mathbf{di} = |\mathbf{p}| - \mathbf{b} $$
    $$ d_{box} = \min(\max(\mathbf{di}), \|\max(\mathbf{di}, 0)\|) $$
    Where $\max(\mathbf{v})$ returns the maximum component of vector $\mathbf{v}$ [S1, unofficial].
2.  **Cross SDF (The Hole Pattern):** The Menger sponge is constructed by subtracting three orthogonal infinite beams from the box. The SDF for the union of these beams (which creates the holes) is:
    $$ d_{cross}(\mathbf{p}) = \min(d_{box}(\mathbf{p}, \mathbf{b}_{inf}), d_{box}(\mathbf{p}_{yzx}, \mathbf{b}_{inf}), d_{box}(\mathbf{p}_{zxy}, \mathbf{b}_{inf})) $$
    Where $\mathbf{b}_{inf} = (\infty, 2, 2)$ and permutations thereof [S1, unofficial].
3.  **Recursive Menger SDF:**
    $$ d_{menger}(\mathbf{p}) = \max(d_{box}(\mathbf{p}, \mathbf{b}_{base}), -d_{cross}(\mathbf{p}/3)) $$
    For $N$ iterations, the hole pattern is scaled and repeated:
    $$ \mathbf{a} = \text{mod}(\mathbf{p} \cdot s, 2.0) - 1.0 $$
    $$ \mathbf{r} = 1.0 - 4.0|\mathbf{a}| $$
    $$ \mathbf{c} = d_{cross}(\mathbf{r}) / s $$
    $$ d.x = \max(d.x, \mathbf{c}.x) $$
    Where $s$ starts at 1.0 and multiplies by 3.0 each iteration [S1, unofficial].

**Algorithm Steps:**
1.  **Camera Setup:** Define view reference point $\mathbf{vrp}=(0,0,0)$ and up vector $\mathbf{vuv}=(0,1,0)$ [S1, unofficial]. Compute basis vectors $\mathbf{u}, \mathbf{v}$ and ray direction $\mathbf{scp}$ from screen coordinates [S1, unofficial].
2.  **Initialization:** Set accumulated distance $f=0$, step size $d.x=0.02$, max depth $t_{max}=100.0$, and max iterations $I_{max}=256$ [S1, unofficial].
3.  **Raymarching Loop:**
    -   Query SDF: $\mathbf{d} = \text{distance\_to\_obj}(\mathbf{p})$, where $\mathbf{p} = \mathbf{prp} + f \cdot \mathbf{scp}$ [S1, unofficial].
    -   Update distance: $f \leftarrow f + d.x$ [S1, unofficial].
    -   Terminate if $|d.x| < \epsilon$ (hit) or $f > t_{max}$ (escape) or iteration count exceeds $I_{max}$ [S1, unofficial].
4.  **Shading:**
    -   If hit, compute normal $\mathbf{n}$ via central finite differences with offset $\mathbf{e}=(0.02, 0, 0)$ [S1, unofficial]:
        $$ \mathbf{n} = \frac{\mathbf{d} - \text{distance\_to\_obj}(\mathbf{p} - \mathbf{e})}{\|\mathbf{d} - \text{distance\_to\_obj}(\mathbf{p} - \mathbf{e})\|} $$
        *(Note: The source code computes the gradient components as $d.x - \text{SDF}(\mathbf{p}-\mathbf{e}_i)$, which approximates the negative gradient. The normalization ensures a unit vector [S1, unofficial].)*
    -   Compute diffuse lighting $b = \mathbf{n} \cdot \frac{\mathbf{prp}-\mathbf{p}}{\|\mathbf{prp}-\mathbf{p}\|}$ [S1, unofficial].
    -   Apply fog: $\text{fog} = 1.0 - f \cdot 0.01$ [S1, unofficial].
    -   Final color: $\mathbf{c} = (b \cdot \mathbf{color} + b^{60.0}) \cdot \text{fog}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def menger_sdf(p, iterations=3):
    """
    Computes the SDF for a Menger Sponge.
    p: vec3 query point
    iterations: int number of fractal iterations
    Returns: float distance
    """
    # Base box SDF
    b = vec3(4.0, 4.0, 4.0)
    di = abs(p) - b
    mc = max(di.x, max(di.y, di.z))
    d_box = min(mc, length(max(di, 0.0)))
    
    # Helper for cross/beam SDF
    def cross_sdf(q):
        # Union of 3 orthogonal beams
        # Beam 1: along X (infinite in X, width 2 in Y/Z)
        d1 = box_sdf(q, vec3(100.0, 2.0, 2.0))
        # Beam 2: along Y
        d2 = box_sdf(q.yzx, vec3(2.0, 100.0, 2.0))
        # Beam 3: along Z
        d3 = box_sdf(q.zxy, vec3(2.0, 2.0, 100.0))
        return min(d1, min(d2, d3))

    def box_sdf(q, b):
        di = abs(q) - b
        mc = max(di.x, max(di.y, di.z))
        return min(mc, length(max(di, 0.0)))

    # Initial distance is the base box
    d = d_box
    
    # Recursive iteration
    s = 1.0
    for i in range(iterations):
        # Domain repetition
        a = mod(p * s, 2.0) - 1.0
        s *= 3.0
        
        # Scale the hole pattern
        r = 1.0 - 4.0 * abs(a)
        c_dist = cross_sdf(r) / s
        
        # Subtract the holes (max of current dist and negative hole dist)
        # Note: The source code uses max(d.x, c.x) where c.x is the hole SDF.
        # Since the hole SDF is positive outside the hole, we want to subtract it.
        # The source code logic: d2.x = max(d2.x, c.x) implies c.x is treated as 
        # a "negative" contribution in the boolean logic context of the specific 
        # implementation where obj_cross returns the distance to the *union* of beams.
        # However, standard SDF subtraction is max(d1, -d2). 
        # The source code `obj_menger` does: `d = max(d1.x, -d2.x)` for simple case.
        # In the loop: `d2.x = max(d2.x, c.x)`. This implies `c.x` is already 
        # negated or the logic relies on the specific `obj_cross` return value.
        # Looking at `obj_cross`, it returns distance to beams. 
        # To subtract beams from box: max(box_dist, -beam_dist).
        # The source code `c = obj_cross(r)/s` returns positive distance to beams.
        # Then `d2.x = max(d2.x, c.x)` would be incorrect for subtraction unless 
        # `obj_cross` returns negative values or the logic is inverted.
        # Re-reading source: `vec2 c = obj_cross(r)/s; d2.x = max(d2.x, c.x);`
        # And `obj_cross` returns `vec2(min(...), 1)`.
        # Actually, in the simple case `obj_menger_simple`: `float d = max(d1.x, -d2.x)`.
        # In the loop, `c` is the distance to the *scaled* cross. 
        # To subtract, we need `-c.x`. The source code has `d2.x = max(d2.x, c.x)`.
        # This suggests `obj_cross` might be returning negative distances? 
        # No, `obj_box_s` returns positive distance outside.
        # There is a discrepancy in the source code logic vs standard SDF math 
        # if `obj_cross` returns positive distances. 
        # However, `1.0 - 4.0*abs(a)` maps the domain [-1,1] to [-3,1].
        # The `obj_cross` function calculates distance to beams.
        # If we assume the source code works, `c.x` must be the negative distance 
        # or the max operation is used differently. 
        # Given the constraint to implement from source:
        d = max(d, c_dist) # As per source code line: d2.x = max(d2.x, c.x)
        
    return d

def raymarch(prp, scp):
    maxd = 100.0
    f = 0.0
    d = 0.02 # Initial step size
    n_hits = 0
    
    for i in range(256):
        if abs(d) < 0.001 or f > maxd:
            break
        
        f += d
        p = prp + scp * f
        d = menger_sdf(p)
        
    return f, p, d
```

## Operating characteristics
- **Time Complexity:** $O(N \cdot I)$, where $N$ is the number of fractal iterations (typically 3) and $I$ is the number of raymarching steps [S1, unofficial]. Each step requires evaluating the SDF, which involves $N$ modulo and scaling operations [S1, unofficial].
- **Memory Complexity:** $O(1)$, as the SDF is computed analytically without storing voxel data [S1, unofficial].
- **Performance:** The recursive modulo operations are computationally expensive on some hardware. NVIDIA OpenGL drivers on Windows have been reported to crash or perform poorly with this specific implementation due to driver bugs with certain GLSL instructions (e.g., GeForce GTX 660, driver R334.67) [S1, unofficial]. AMD hardware (Radeon R7) has shown stable performance [S1, unofficial].
- **Strengths:** No storage required for geometry; infinite resolution at the fractal boundary [S1, unofficial].
- **Limitations:** Precision issues may arise at deep recursion levels or very small scales [S1, unofficial]. The fixed step size initialization ($0.02$) may lead to overshooting in tight spaces if not adjusted [S1, unofficial].

## Background
The method originates from Inigo Quilez's articles on raymarching and fractals, adapted into GLSL code samples by Geeks3D [S1, unofficial]. The Menger sponge is a classic 3D fractal formed by recursively removing cubes from a larger cube [S1, unofficial]. The specific implementation uses a "cross" subtraction pattern to create the characteristic holes [S1, unofficial].

## Sources
[S1] (GLSL) Menger Sponge, Raymarching Code Samples Updated, https://www.geeks3d.com/20140201/glsl-menger-sponge-raymarching-code-samples-updated/, fetched 2026-10-26 (unofficial)
