# Zhukov Ambient Occlusion

Also called: Ambient Occlusion (Empirical Model), Geometric Obscurance, Obscurance

## Overview
Zhukov Ambient Occlusion is an empirical ambient light illumination model that simulates indirect illumination by computing the geometric obscurance of a surface point, representing how open the point is to the environment [S1, unofficial]. It approximates darkening effects in obscured areas by attenuating the constant ambient term of the Phong lighting model, rather than calculating the intensity of secondary reflections as in radiosity [S1, unofficial]. This approach is computationally cheaper and aesthetically driven, aiming to achieve illumination effects similar to radiosity in a shorter time period [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Surface point $x$, surface normal $n$, scene geometry (for ray intersection tests).
*   **Output:** Ambient Occlusion factor $OA(x, n)$, a value in $[0, 1]$.

**Data Structures:**
*   The method relies on hemisphere sampling. No specific complex data structures are defined in the source for the core integration; standard ray-scene intersection routines are assumed [S1, unofficial].

**Steps:**
1.  Define a hemisphere $\Omega$ oriented with normal $n$ centered at surface point $x$.
2.  Sample directions $\omega$ over the hemisphere $\Omega$.
3.  For each direction $\omega$, cast a ray from $x$ in direction $\omega$.
4.  Determine the visibility function $V(x, \omega)$:
    *   $V(x, \omega) = 1$ if an occluding surface is found in direction $\omega$.
    *   $V(x, \omega) = 0$ otherwise [S1, unofficial].
5.  Compute the integrand using the absolute value of the dot product $|\omega \cdot n|$ [S1, unofficial].
6.  Integrate over the hemisphere to compute $OA(x, n)$.
7.  Apply the factor to the Phong ambient term: $I_{ambient} = I_a k_a (1 - OA(x, n))$ [S1, unofficial].

**Formula:**
The ambient occlusion factor is defined as:
$$ OA(x, n) = \frac{1}{\pi} \int_{\Omega} V(x, \omega) |\omega \cdot n| d\omega $$
Where:
*   $x$: Surface point.
*   $n$: Surface normal.
*   $\Omega$: Hemisphere oriented with $n$ centered at $x$.
*   $V(x, \omega)$: Visibility function (1 if occluder found, 0 otherwise).
*   $|\omega \cdot n|$: Absolute value of the dot product between direction $\omega$ and normal $n$ [S1, unofficial].

**Parameters:**
*   $OA(x, n)$: Range $[0, 1]$. Value 1 denotes full occlusion (repressed), value 0 denotes no occluding geometry [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_zhukov_ao(x, n, scene, num_samples):
    """
    Computes Zhukov Ambient Occlusion for point x with normal n.
    """
    total_occlusion = 0.0
    
    # Sample directions over the hemisphere oriented by n
    for i in range(num_samples):
        # Generate a random direction omega on the hemisphere
        omega = sample_hemisphere(n)
        
        # Cast ray to find intersection
        hit_distance = scene.ray_intersect(x, omega)
        
        # Visibility function: 1 if occluder found, 0 otherwise
        # Note: Source defines V=1 if surface found, V=0 otherwise.
        # This implies V represents "occlusion presence".
        if hit_distance is not None:
            v = 1.0
        else:
            v = 0.0
            
        # Compute dot product term
        dot_product = abs(dot(omega, n))
        
        # Accumulate integrand
        total_occlusion += v * dot_product
        
    # Normalize by pi (area of hemisphere is 2*pi, but formula uses 1/pi)
    # The source formula divides by pi.
    oa = total_occlusion / (num_samples * pi)
    
    return oa
```
*[G] The normalization factor $1/\pi$ is taken directly from the source formula. Standard Monte Carlo integration over a hemisphere usually divides by the area ($2\pi$) or uses a specific PDF. The source explicitly writes $1/\pi$. The pseudocode assumes a uniform sampling strategy where the sum is divided by the number of samples, and the constant $1/\pi$ is applied. If the sampling PDF is $1/(2\pi)$, the estimator would be $\frac{1}{N} \sum \frac{V |\omega \cdot n|}{1/(2\pi)} = \frac{2\pi}{N} \sum V |\omega \cdot n|$. The source formula is an integral, not an estimator. The implementation detail of the PDF is not given in the sources. The pseudocode reflects the direct evaluation of the integral's structure as presented.]*

**Edge Cases and Pitfalls:**
*   **Precision:** The visibility function is binary. Small errors in ray origin (self-intersection) can cause incorrect occlusion.
*   **Boundaries:** The integration is strictly over the hemisphere defined by the normal.
*   **Degenerate Input:** If the point is inside geometry, the visibility function behavior depends on the ray intersection logic (first hit vs. any hit). The source assumes standard exterior surface points [S1, unofficial].

## Subtopics

### Zhukov Obscurance
Zhukov Obscurance is a generalization of ambient occlusion where the binary visibility function is replaced by a distance suitability function $\rho(d(x, \omega))$ [S1, unofficial]. It measures ambient light magnitude based on the distance to the first intersection, rather than just presence/absence [S1, unofficial].

**Definition:**
It computes an obscurance value $W(x, n)$ that maps distance to ambient light contribution [S1, unofficial].

**Formula:**
$$ W(x, n) = \frac{1}{\pi} \int_{\Omega} \rho(d(x, \omega)) |\omega \cdot n| d\omega $$
Where:
*   $W$: Obscurance value ($0 \le W \le 1$).
*   $d(x, \omega)$: Distance to the first intersection in direction $\omega$.
*   $\rho$: Distance suitability mapping function [S1, unofficial].

The reflected intensity is calculated as:
$$ I(x) = I_a k_a(x) W(x) $$
Where:
*   $I_a$: Ambient light intensity.
*   $k_a(x)$: Diffuse reflectance for ambient light [S1, unofficial].

**Distance Suitability Function:**
The function $\rho(d)$ is defined by the following properties:
1.  $\rho(0) = 0$ (fully occluded).
2.  $\rho(+\infty) = 1$ (fully open).
3.  $\rho'(d) > 0$ (monotonically increasing).
4.  $\rho''(d) < 0$ (monotonically decreasing derivative/smoothness) [S1, unofficial].

A suggested family of functions is:
$$ \rho(d) = 1 - e^{-\tau d} $$
Where $\tau$ is a positive parameter defining the decay rate [S1, unofficial].

**When to choose:**
Choose Zhukov Obscurance when a smooth falloff based on distance is preferred over binary occlusion, allowing for more nuanced ambient light attenuation [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   Time complexity is proportional to the number of samples and the cost of ray intersection tests. It is generally faster than full radiosity because it does not compute global light transport, only local geometric obscurance [S1, unofficial].
*   Memory complexity is low, typically storing the result as vertex attributes or texture maps for later use [S1, unofficial].

**Measured Performance:**
*   Not given in the sources.

**Scaling:**
*   Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Computationally cheaper than radiosity; aesthetically pleasing; simulates indirect illumination effects effectively [S1, unofficial].
*   **Limitations:** Not physically correct; relies on empirical parameters; does not account for light source directionality in the basic AO model (only ambient) [S1, unofficial].

**When to use it and when not:**
*   Use for static scenes where precomputed ambient occlusion maps are beneficial, or for real-time approximations of indirect lighting where physical accuracy is secondary to performance and aesthetics [S1, unofficial].
*   Do not use when physically accurate global illumination is required [S1, unofficial].

## Background
Zhukov Ambient Occlusion was introduced by Zhukov et al. in 1998 [S1, unofficial]. The method was designed to achieve illumination effects of occlusion similar to those acquired from radiosity but in a shorter time period [S1, unofficial]. It has become a basis for many other techniques, including those by Iones et al. [IKSZ03] and Kontkanen et al. [KL05] [S1, unofficial]. The technique has become quite popular in the movie industry and has undergone refinement since its introduction [S1, unofficial].

## Sources
[S1] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
