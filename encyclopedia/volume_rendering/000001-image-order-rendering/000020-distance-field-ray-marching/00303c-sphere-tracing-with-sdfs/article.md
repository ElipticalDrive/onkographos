# Sphere Tracing with SDFs

Also called: Raymarching with Signed Distance Functions, SDF Raymarching

## Overview
Sphere tracing with SDFs is a technique for rendering implicit surfaces by casting rays and stepping along them using the distance field as a guide [S1, unofficial][S2, unofficial]. It solves the problem of rendering complex, analytic geometry without explicit mesh data by replacing triangle intersection tests with distance queries [S1, unofficial]. It differs from standard triangle-based ray tracing by using the mathematical property of the SDF to guarantee no surface is missed, and from other volume rendering methods by focusing on surface intersection rather than volumetric integration [S2, unofficial].

## Shared principles
The method renders geometry by shooting a ray through each pixel and repeatedly querying a Signed Distance Function (SDF) [S2, unofficial]. An SDF is a formula that, given any point $p$ in space, returns the distance from $p$ to the nearest surface [S1, unofficial][S10][S14, unofficial]. The sign indicates position: negative inside, positive outside, zero exactly on the surface [S1, unofficial][S14, unofficial][S20, unofficial].

The core loop initializes a ray with an origin point $ro$ and a normalized direction vector $v$ [S1, unofficial][S10][S12, unofficial]. It maintains a current position $p$ (initially $ro$) [S10]. In each iteration:
1. The SDF is evaluated at $p$ to get the distance $d$ to the nearest surface [S1, unofficial][S10].
2. If $d < 0$, the ray has penetrated the surface; the algorithm returns a hit [S6, unofficial].
3. If $d \le \text{HIT\_THRESHOLD}$ (a small value such as 0.001), the ray has hit the surface [S1, unofficial][S3, unofficial]. The loop terminates.
4. Otherwise, the position is advanced: $p \leftarrow p + d \cdot v$ [S10]. This step size is guaranteed not to overshoot the surface because $d$ is the distance to the nearest surface [S2, unofficial].
5. The loop continues while the step count is less than $\text{MAX\_STEPS}$ [S1, unofficial][S7, unofficial].

If the loop terminates without a hit (e.g., $t$ exceeds a maximum range or $\text{MAX\_STEPS}$ is reached), the ray misses [S1, unofficial].

The scene is defined by combining primitive SDFs. For a sphere of radius $r$ centered at $c$, the distance function is $d(p) = \|p - c\| - r$ [S1, unofficial][S19, unofficial][S20, unofficial]. Union of two objects with distance fields $d1$ and $d2$ is $iso(p) = \min(d1(p), d2(p))$ [S1, unofficial][S10][S19, unofficial][S20, unofficial]. Intersection is $iso(p) = \max(d1(p), d2(p))$ [S1, unofficial][S10][S19, unofficial][S20, unofficial]. Subtraction of object $d2$ from $d1$ is $iso(p) = \max(d1(p), -d2(p))$ [S1, unofficial][S10][S19, unofficial][S20, unofficial].

Parameters include $\text{MAX\_STEPS}$ (a safety cap, typically 32 to 128 depending on implementation [S1, unofficial][S4, unofficial]) and $\text{HIT\_THRESHOLD}$ (epsilon) [S1, unofficial][S12, unofficial]. A maximum range or accumulated travel distance limit can also bound rays that miss [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march(ro, v, scene, max_steps=40, hit_threshold=0.001, max_range=100.0):
    p = ro
    for i in range(max_steps):
        d = scene(p)
        if d < 0:
            return p, True, dot(p - ro, v)
        if d <= hit_threshold:
            return p, True, dot(p - ro, v)
        p += v * d
        if dot(p - ro, v) > max_range:
            break
    return p, False, dot(p - ro, v)
```

## Subtopics

### [Zeno Implementation](1544-zeno-implementation.md)
The Zeno implementation is a specific software architecture for sphere tracing that minimizes the coding effort required to add new implicit surfaces by relying on a numerical gradient approximation for shading. It solves the problem of boilerplate code in ray tracers by allowing primitives to be defined solely by a signed distance function (SDF), automatically deriving normals via finite differences. It differs from standard sphere tracing implementations by explicitly decoupling the intersection logic from the shading logic, using a fixed six-sample stencil for gradients rather than analytical derivatives.

### [Raymarched SDF Volume Rendering](3663-raymarched-sdf-volume-rendering.md)
Raymarched SDF Volume Rendering is a technique to display a 2D projection of a 3D object defined by a mathematical distance function (SDF) by casting rays from the camera and stepping along them until they intersect the surface. It creates an image from equations rather than rasterized geometry, allowing for complex, analytic shapes without explicit mesh data. Unlike traditional Unity Raycasting, which is not available in shaders, this method implements its own ray-intersection logic using distance queries.

### [Uniform Sphere Tracing](4406-uniform-sphere-tracing.md)

### [Menger Sponge Raymarching](4875-menger-sponge-raymarching.md)
A raymarching technique rendering a Menger sponge fractal using a signed distance function (SDF) combined with a floor plane, utilizing GLSL 120.

### Mandelbulb Sphere Tracing
#### definition
- [S1, unofficial] A variation of sphere tracing that uses a Mandelbulb SDF (`distMandelBulb`) instead of primitive shapes, controlled by a `power` parameter.
- [S1, unofficial] Sphere tracing applied to the Mandelbulb fractal. Uses a specific distance estimator function `distMandelBulb` instead of a generic shape distance function.
#### step
- [S1, unofficial] If mode `m` is 3, increment `power` and call `RenderMandelTraceWithMultiThreading`.
- [S1, unofficial] Initialize step position to camera origin. Calculate minimum distance using `distMandelBulb(x, y, z)`.
- [S1, unofficial] Loop while minimum distance is greater than 0.01 and less than 2000. Increment iteration counter. Update step position by adding `minDist * cos(angle)` to each coordinate.
- [S1, unofficial] If final minimum distance is <= 0.01, the ray hit. Store distance and iteration count.
- [S1, unofficial] If final minimum distance > 0.01, the ray missed. Store distance as 10000 and iteration count.
#### parameter
- [S1, unofficial] mult: Multiplier used in an alternate rendering mode (commented out) for mu calculation.
- [S1, unofficial] threshhold: Threshold used in the alternate rendering mode.
- [S1, unofficial] Hit threshold: 0.01. If `minDist <= 0.01`, the surface is considered hit.
- [S1, unofficial] Max march distance: 2000.
#### code
- [S1, unofficial] Alternate rendering logic for Mandelbulb mu calculation (commented out in source).
- [S1, unofficial] Core sphere tracing loop for Mandelbulb.

## Background
The technique is implemented in the `3D-Sphere-Tracing` repository by Hayden Parker, which provides examples of sphere tracing with various SDF primitives including spheres, boxes, and tori, as well as fractals like the Mandelbulb [S1, unofficial]. The implementation demonstrates the use of SDL for window management and rendering, and includes multi-threaded rendering support for complex fractals [S1, unofficial].

## Sources
[S1] haydenparker3/3D-Sphere-Tracing, https://github.com/haydenparker3/3D-Sphere-Tracing, fetched 2026-10-26 (unofficial)
