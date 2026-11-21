# Zhukov Ambient Occlusion

Also called: Ambient Occlusion (Empirical Model), Geometric Obscurance, Obscurance

## Overview
Zhukov Ambient Occlusion is an empirical local illumination model that simulates indirect lighting by computing the geometric obscurance of a surface point, representing how open the point is to the environment [S1, unofficial]. It differs from radiosity by measuring the degree of occlusion (lack of secondary light) rather than the intensity of secondary reflections, making it computationally cheaper and aesthetically driven rather than physically correct [S1, unofficial]. It builds on [Ambient Occlusion via Shadow Rays](878-ambient-occlusion-via-shadow-rays.md) by approximating the hemisphere integral of visibility weighted by the cosine of the incident angle.

## How it works
**Inputs and Outputs:**
- **Input:** Surface point $x$, surface normal $n$, scene geometry (for visibility queries).
- **Output:** Ambient Occlusion factor $OA(x, n) \in [0, 1]$.

**Data Structures:**
- The visibility function $V(x, \omega)$ is typically implemented via ray casting or ray marching. For a given direction $\omega$, it returns 1 if an occluder is found within a certain distance or range, and 0 otherwise [S1, unofficial].

**Formula:**
The ambient occlusion factor is defined as the integral over the hemisphere $\Omega$ oriented with normal $n$ centered at $x$:

$$
OA(x, n) = \frac{1}{\pi} \int_{\Omega} V(x, \omega) |\omega \cdot n| d\omega
$$

Where:
- $V(x, \omega)$ is the visibility function (1 if occluder found, 0 otherwise) [S1, unofficial].
- $|\omega \cdot n|$ is the absolute value of the dot product between the direction vector $\omega$ and the normal $n$ [S1, unofficial]. Note: The original source notation uses a floor operator $\lfloor \omega \cdot n \rfloor$, but it is interpreted as the absolute value for consistency with standard hemisphere sampling formulations [S1, unofficial].
- $\Omega$ is the hemisphere oriented with $n$ centered at $x$ [S1, unofficial].

**Parameters:**
- $OA(x, n)$ range: 0 to 1. A value of 1 denotes full occlusion (repressed), and 0 denotes no occluding geometry influencing the point [S1, unofficial].

**Steps:**
1. For the surface point $x$ and normal $n$, define the hemisphere $\Omega$ of directions.
2. Sample directions $\omega$ uniformly or quasi-randomly over $\Omega$.
3. For each direction $\omega$, cast a ray from $x$ in direction $\omega$.
4. Determine $V(x, \omega)$:
   - If the ray intersects geometry within the scene, $V(x, \omega) = 1$.
   - Otherwise, $V(x, \omega) = 0$.
5. Compute the dot product $|\omega \cdot n|$.
6. Estimate the integral using Monte Carlo integration:
   $$
   OA(x, n) \approx \frac{1}{N} \sum_{i=1}^{N} V(x, \omega_i) |\omega_i \cdot n|
   $$
   Where $N$ is the number of samples.
7. Multiply the constant ambient term of the Phong model by $(1 - OA(x, n))$ to approximate global illumination effects [S1, unofficial].

**Edge Cases and Pitfalls:**
- The visibility function $V(x, \omega)$ is binary in this basic model, leading to potential aliasing if not sampled sufficiently [S1, unofficial].
- The interpretation of the dot product term as absolute value is an assumption based on consultation with standard practices, as the original paper's use of a floor operator is unclear [S1, unofficial].

## Subtopics

### [Zhukov Obscurance](4774-zhukov-obscurance.md)
Generalizes standard binary ambient occlusion by replacing the visibility function with a distance suitability function, providing a smooth gradient of darkening based on proximity.

## Operating characteristics
- **Time Complexity:** Depends on the number of samples $N$ and the cost of visibility queries (ray casting/marching).
- **Memory Complexity:** Low, as it typically computes values per vertex or pixel without storing large buffers, though results may be stored as vertex attributes or texture maps for later use [S1, unofficial].
- **Strengths:** Computationally cheaper than radiosity; aesthetically pleasing; simulates indirect illumination effects effectively for local geometry [S1, unofficial].
- **Limitations:** Not physically correct; ignores indirect bounces; considers only local geometry [S1, unofficial].
- **When to use:** When real-time or fast offline rendering is required, and aesthetic quality of local shadows/occlusion is prioritized over physical accuracy.
- **When not to use:** When physically accurate global illumination is required.

## Background
The method was introduced by Zhukov et al. in [ZIK98] as an empirical ambient light illumination model aiming to achieve illumination effects of occlusion similar or correct up to a certain degree to those acquired from radiosity but in a shorter time period [S1, unofficial]. It has been a basis for many other works, such as [IKSZ03] and [KL05], and has become quite popular in the movie industry under the influence of which it has undergone some refinement [S1, unofficial]. Kontkanen et al. mention that ambient occlusion is considered a special case, a simplification of the model proposed by Zhukov et al. [S1, unofficial].

## Sources
[S1] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
