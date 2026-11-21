# Deferred Shading for Isosurfaces

Also called: Curvature Estimation, Deferred Shading with Differential Properties, Differential Surface Properties Computation, Gradient and Hessian Estimation, Image-Space Shading

## Overview
Deferred Shading for Isosurfaces is a multi-pass image-space technique that computes surface properties—position, gradient, Hessian, and curvature—in separate rendering passes, decoupling shading from volume sampling [S1]. It solves the problem of efficiently applying complex lighting and material models to isosurfaces extracted from scalar volume data by calculating differential properties only for visible pixels, making its complexity independent of the underlying grid size [S1]. It differs from direct volume rendering by computing shading only at the single intersection point where the scalar field exceeds a threshold, rather than integrating opacity and color along the entire ray path [S1].

## How it works
The method operates on the intersection point $\mathbf{p}$ found by the parent method [G]. The core mechanism involves computing surface attributes at $\mathbf{p}$ and applying a shading model [G].

### Inputs and Outputs
*   **Input:** 3D scalar volume texture $f(\mathbf{x})$, ray origin and direction per pixel (from ray-casting pass).
*   **Intermediate Outputs:**
    *   Position image $\mathbf{pos}$ (RGB floating point) [S1].
    *   Gradient image $\mathbf{g}$ (RGB floating point) [S1].
    *   Hessian image $\mathbf{H}$ (stored in two RGB floating point images) [S1].
    *   Curvature images: Principal curvatures $\kappa_1, \kappa_2$ and directions $\mathbf{e}_1, \mathbf{e}_2$ (stored in two floating point targets) [S1].
*   **Final Output:** Shaded image.

### Data Structures
*   **Position Image:** Stores the 3D world coordinates $\mathbf{p}$ of the isosurface intersection for each pixel [S1].
*   **Gradient Image:** Stores the 3D gradient vector $\nabla f(\mathbf{p})$ as RGB components [S1].
*   **Hessian Image:** Stores the 6 unique components of the Hessian matrix $\nabla^2 f(\mathbf{p})$ (due to symmetry) distributed across two RGB textures [S1].
*   **Curvature Images:** Store the scalar principal curvatures and the 3D principal direction vectors [S1].

### Numbered Steps

1.  **Ray-Casting (Position Pass):**
    *   Input: Volume $f$.
    *   Output: Position image $\mathbf{pos}$.
    *   Perform ray casting to find the first intersection with the isosurface $f(\mathbf{x}) = c$ [S1]. Store the intersection point $\mathbf{p}$ in the position image [S1].
2.  **Gradient Computation (Gradient Passes):**
    *   Input: Position image $\mathbf{pos}$, Volume $f$.
    *   Output: Gradient image $\mathbf{g}$.
    *   Compute the gradient $\mathbf{g} = \nabla f$ in three rendering passes (one per component $g_x, g_y, g_z$) [S1].
    *   In each pass, evaluate a tri-cubic B-spline convolution sum to compute one first-order partial derivative [S1].
    *   This requires eight texture fetches from the 3D volume texture and three fetches from 1D filter weight textures [S1].
    *   Store the resulting gradient vector $\mathbf{g}$ in a single RGB floating point image [S1].
3.  **Hessian Computation (Hessian Passes):**
    *   Input: Position image $\mathbf{pos}$, Volume $f$.
    *   Output: Hessian image $\mathbf{H}$.
    *   Compute the Hessian matrix $\mathbf{H} = \nabla \mathbf{g}$ in six rendering passes (one for each unique second partial derivative) [S1].
    *   Due to symmetry, only six unique components are calculated [S1].
    *   Each pass uses either eleven or fourteen texture fetches [S1].
    *   Store the six coefficients in two RGB floating point images [S1].
4.  **Curvature Computation (Curvature Pass):**
    *   Input: Gradient image $\mathbf{g}$, Hessian image $\mathbf{H}$.
    *   Output: Principal curvatures $\kappa_1, \kappa_2$ and directions $\mathbf{e}_1, \mathbf{e}_2$.
    *   Compute the shape operator $\mathbf{S}$, defined as the tangent space projection of the normalized Hessian [S1]:
        $$ \mathbf{S} = \mathbf{P}^T \frac{\mathbf{H}}{|\mathbf{g}|} \mathbf{P} $$
        where $\mathbf{P} = \mathbf{I} - \frac{\mathbf{g}\mathbf{g}^T}{|\mathbf{g}|^2}$ is the projection matrix onto the tangent plane [S1].
    *   The shape operator $\mathbf{S}$ is a $2 \times 2$ matrix $\mathbf{A}$ in the tangent basis [S1]. Compute the principal curvature magnitudes $\kappa_{1,2}$ as the eigenvalues of $\mathbf{A}$:
        $$ \kappa_{1,2} = \frac{1}{2} \left( \text{trace}(\mathbf{A}) \pm \sqrt{\text{trace}(\mathbf{A})^2 - 4 \det(\mathbf{A})} \right) $$
        [S1]
    *   Compute the principal directions $\mathbf{e}_i$ using the formula:
        $$ \mathbf{e}_i = \kappa_i \mathbf{u} + (\kappa_i + a_{22} - a_{11}) \mathbf{v} $$
        where $\mathbf{u}, \mathbf{v}$ are the basis vectors of the tangent plane and $a_{11}, a_{22}$ are diagonal elements of $\mathbf{A}$ [S1].
    *   Render the curvature magnitudes and directions to two floating point targets [S1].
5.  **Shading Pass:**
    *   Input: Position image $\mathbf{pos}$, Gradient image $\mathbf{g}$, Curvatures $\kappa_{1,2}$, Directions $\mathbf{e}_{1,2}$ [S1].
    *   Output: Final shaded image.
    *   Compute the surface normal $\mathbf{n} = \frac{\mathbf{g}}{|\mathbf{g}|}$ [S1].
    *   Apply lighting models (e.g., Blinn-Phong) using $\mathbf{n}$ [S1].
    *   Apply curvature-based effects:
        *   **Curvature Color Mapping:** Map $\kappa_1, \kappa_2$, mean curvature $(\kappa_1+\kappa_2)/2$, or Gaussian curvature $\kappa_1\kappa_2$ to colors via 1D or 2D lookup textures [S1].
        *   **Accessibility Shading:** Use a 1D curvature transfer function to darken areas with large negative maximum curvature [S1].
        *   **Flow Advection:** Project 3D curvature direction vectors to the 2D image plane and advect flow on a per-pixel basis [S1].
        *   **Non-Photorealistic Effects:** Implement silhouette outlining (using curvature to control thickness) and ridge/valley line depiction [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dot(a, b): return sum(x*y for x,y in zip(a,b))
def norm(v): return sum(x*x for x in v)**0.5
def normalize(v): l=norm(v); return [x/l for x in v] if l>0 else [0,0,0]
def outer(v1, v2): return [[v1[i]*v2[j] for j in range(3)] for i in range(3)]
def mat_mul(A, B):
    C=[[0.0]*3 for _ in range(3)]
    for i in range(3):
        for j in range(3):
            for k in range(3): C[i][j]+=A[i][k]*B[k][j]
    return C
def transpose(M): return [[M[j][i] for j in range(3)] for i in range(3)]
def mat_vec(M, v): return [dot(M[i], v) for i in range(3)]
def tangent_basis(g):
    # Construct two orthogonal vectors u, v to g
    if abs(g[0]) > abs(g[1]):
        u = normalize([-g[1], g[0], 0.0])
    else:
        u = normalize([0.0, -g[2], g[1]])
    v = normalize([g[1]*u[2]-g[2]*u[1], g[2]*u[0]-g[0]*u[2], g[0]*u[1]-g[1]*u[0]])
    return u, v

def ray_casting_pass(volume, rays, width, height):
    pos = [[[0.0]*3 for _ in range(width)] for _ in range(height)]
    for y in range(height):
        for x in range(width):
            t = find_intersection(rays[y][x], volume)
            if t is not None: pos[y][x] = [rays[y][x][i]+t*rays[y][x][i+3] for i in range(3)]
    return pos

def gradient_pass(volume, pos, comp, width, height):
    grad = [[[0.0] for _ in range(width)] for _ in range(height)]
    for y in range(height):
        for x in range(width):
            grad[y][x][0] = eval_tricubic_deriv(volume, pos[y][x], comp)
    return grad

def compute_gradient(volume, pos, width, height):
    gx, gy, gz = gradient_pass(volume, pos, 0, width, height), gradient_pass(volume, pos, 1, width, height), gradient_pass(volume, pos, 2, width, height)
    return [[[gx[y][x][0], gy[y][x][0], gz[y][x][0]] for x in range(width)] for y in range(height)]

def compute_hessian(volume, pos, width, height):
    h1 = [[[0.0]*3 for _ in range(width)] for _ in range(height)]
    h2 = [[[0.0]*3 for _ in range(width)] for _ in range(height)]
    pairs = [(0,0),(1,1),(2,2),(0,1),(0,2),(1,2)]
    for i, (a,b) in enumerate(pairs):
        val = hessian_pass(volume, pos, (a,b), width, height)
        if i < 3: h1 = [[[val[y][x][0] if j==i else h1[y][x][j] for j in range(3)] for x in range(width)] for y in range(height)]
        else: h2 = [[[val[y][x][0] if j==i-3 else h2[y][x][j] for j in range(3)] for x in range(width)] for y in range(height)]
    return h1, h2

def compute_curvature(grad, h1, h2, width, height):
    mag = [[[0.0]*2 for _ in range(width)] for _ in range(height)]
    dir_ = [[[0.0]*3 for _ in range(width)] for _ in range(height)]
    for y in range(height):
        for x in range(width):
            g = grad[y][x]
            ng = norm(g)
            if ng == 0: continue
            H = [[h1[y][x][0] if i==0 and j==0 else h1[y][x][1] if i==1 and j==1 else h1[y][x][2] if i==2 and j==2 else h2[y][x][0] if i==0 and j==1 else h2[y][x][1] if i==0 and j==2 else h2[y][x][2] for j in range(3)] for i in range(3)]
            P = [[1.0 if i==j else 0.0 for j in range(3)] for i in range(3)]
            for i in range(3):
                for j in range(3): P[i][j] -= g[i]*g[j]/(ng*ng)
            S = mat_mul(mat_mul(transpose(P), [[H[i][j]/ng for j in range(3)] for i in range(3)]), P)
            u, v = tangent_basis(g)
            A = [[dot(u, mat_vec(S, u)), dot(u, mat_vec(S, v))], [dot(v, mat_vec(S, u)), dot(v, mat_vec(S, v))]]
            tr, det = A[0][0]+A[1][1], A[0][0]*A[1][1]-A[0][1]*A[1][0]
            disc = tr**2 - 4*det
            if disc < 0: continue
            sq = disc**0.5
            k1, k2 = 0.5*(tr+sq), 0.5*(tr-sq)
            e1 = normalize([k1*u[i]+(k1+A[1][1]-A[0][0])*v[i] for i in range(3)])
            e2 = normalize([k2*u[i]+(k2+A[1][1]-A[0][0])*v[i] for i in range(3)])
            mag[y][x] = [k1, k2]
            dir_[y][x] = e1
    return mag, dir_
```

### Parameters and Typical Values
*   **Isosurface Threshold:** Not given in the sources.
*   **B-Spline Order:** Tri-cubic (order 4) [S1].
*   **Texture Fetches for Gradient:** 8 from 3D volume, 3 from 1D weights [S1].
*   **Texture Fetches for Hessian:** 11 or 14 per pass [S1].

### Edge Cases and Pitfalls
*   **Flow Advection Seams:** Advecting flow along curvature directions may produce visible seams because the orientation of curvature direction vectors is not uniquely defined [S1].
*   **Precision:** The method relies on floating point images for intermediate results (gradient, Hessian, curvature) to maintain precision [S1].
*   **Degenerate Input:** If the gradient magnitude is zero (flat region), the normal and curvature computation may be undefined. The sources do not specify a guard for this case [G].

## Subtopics

### [Surface shading for thresholded volumes](1159-surface-shading-for-thresholded-volumes.md)
Surface shading for thresholded volumes applies illumination models to isosurfaces extracted from scalar volume data by determining the first hit of a ray against a thresholded boundary. It distinguishes itself from direct volume rendering by computing shading only at the single intersection point where the scalar field exceeds a threshold, rather than integrating opacity and color along the entire ray path. This method is a specific implementation within the broader framework of deferred shading for isosurfaces, focusing on efficient surface property estimation.

### [Normal-based Contextual Shading](6617-normal-based-contextual-shading.md)
Normal-based Contextual Shading is an object-space shading technique applied after finding closest intersection points to achieve better results than image-space depth-gradient shading. It solves the problem of inaccurate normal estimation by fitting a linear or biquadratic function to the set of points that belong to the same iso-surface. It differs from its parent, Deferred Shading for Isosurfaces, by operating directly on the discrete intersection points rather than computing gradients in volume space.

## Operating characteristics
*   **Time Complexity:** Image-space complexity, independent of the grid size [S1]. The total number of passes is 14 (3 for position, 3 for gradient, 6 for Hessian, 1 for curvature, 1 for shading) [S1].
*   **Memory Complexity:** Requires storage for multiple floating point images (position, gradient, Hessian, curvature) [S1].
*   **Measured Performance:** On a NVIDIA GeForce 6800, each property calculation pass renders a single screen-aligned quad [S1].
*   **Strengths:** Decouples shading from volume sampling; allows for complex lighting and curvature-based effects; real-time rates for flow advection [S1].
*   **Limitations:** Requires multiple rendering passes; seams in flow advection due to non-unique curvature directions [S1].
*   **When to Use:** When high-quality shading and curvature visualization of isosurfaces are required, and the hardware supports multiple render targets and floating point textures [S1].
*   **When Not to Use:** When memory bandwidth is a bottleneck due to the large number of texture fetches and intermediate buffers [G].

## Background
The method was introduced in "Real-Time Ray-Casting and Advanced Shading of Discrete Isosurfaces" by [S1]. It builds on the principles of deferred shading and differential geometry applied to volume data. The technique exploits tri-cubic B-spline reconstruction filters to compute accurate gradients and Hessians in image space [S1].

## Sources
[S1] Real-Time Ray-Casting and Advanced Shading of Discrete Isosurfaces, https://cgl.ethz.ch/people/archive/siggc/publications/eg05.pdf, fetched 2026-10-26
[S2] sebastianlipponer/surface_splatting, https://github.com/sebastianlipponer/surface_splatting, fetched 2026-10-28 (unofficial)
