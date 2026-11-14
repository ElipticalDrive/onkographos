# Inverted Sphere Parameterization

Also called: Inverted Sphere Parametrization, Outer volume parameterization

## Overview
Inverted Sphere Parameterization is a spatial parameterization scheme that allows Neural Radiance Fields to model 360° captures of unbounded scenes by separately representing foreground and background content [S1, unofficial]. It solves the parameterization failure of standard NeRF, which assumes the entire scene fits within a bounded volume, leading to either missing background elements or loss of detail due to limited sampling resolution in large-scale scenes [S1, unofficial]. It sets itself apart from its parent [NeRF++](6198-nerf.md) by providing the specific mathematical mapping for the outer volume.

## How it works
**Inputs and Outputs**
*   **Inputs:** Camera ray origin $\mathbf{o}$, camera ray direction $\mathbf{d}$, two MLPs: $\text{MLP}_{in}$ (for inner volume) and $\text{MLP}_{out}$ (for outer volume) [S1, unofficial].
*   **Outputs:** Pixel color $C(\mathbf{r})$ [S1, unofficial].

**Data Structures**
*   **Inner Volume:** Bounded by a unit sphere centered at the origin [S1, unofficial].
*   **Outer Volume:** The complement of the unit sphere ($r > 1$) [S1, unofficial].
*   **MLP Inputs:** $\text{MLP}_{in}$ takes standard Euclidean coordinates $(x, y, z)$ and viewing direction $\mathbf{d}$ [S1, unofficial]. $\text{MLP}_{out}$ takes re-parameterized coordinates $(x_0, y_0, z_0, 1/r)$ and viewing direction $\mathbf{d}$ [S1, unofficial].

**Parameters**
*   **Inner Volume Boundary:** Unit sphere radius $R=1$ [S1, unofficial].
*   **Outer Volume Parameter Range:** $1/r \in [0, 1]$ [S1, unofficial].
*   **Sampling:** A finite number of points sampled from the interval $[0, 1]$ for the variable $1/r$ in the outer volume integration [S1, unofficial].

**Step-by-Step Method**
1.  **Partition Space:** Define the scene space as an inner unit sphere (containing cameras and foreground) and an outer volume (containing the background environment) [S1, unofficial].
2.  **Compute Inner Volume Contribution (Terms i and ii):** Raycast the ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ within the unit sphere using standard Euclidean parameterization [S1, unofficial]. Find the intersection point $t'$ where the ray exits the unit sphere by solving $|\mathbf{o} + t'\mathbf{d}| = 1$ [S1, unofficial]. Compute the accumulated color and transmittance for $t \in [0, t']$ using $\text{MLP}_{in}$ [S1, unofficial].
3.  **Compute Outer Volume Contribution (Term iii):** For the outer volume ($t > t'$), apply the inverted sphere parameterization to map 3D points to a bounded 4D space [S1, unofficial]. Calculate point $\mathbf{a}$, the intersection of the ray with the unit sphere: Solve $|\mathbf{o} + t_a\mathbf{d}| = 1$ for $t_a$ [S1, unofficial]. Calculate point $\mathbf{b}$, the midpoint of the chord formed by the ray passing through the sphere: Solve $\mathbf{d}^T(\mathbf{o} + t_b\mathbf{d}) = 0$ for $t_b$ [S1, unofficial]. For a sample inverse radius $u = 1/r \in (0, 1]$, determine the corresponding 3D direction $(x_0, y_0, z_0)$ by rotating vector $\mathbf{a}$ along the axis $\mathbf{b} \times \mathbf{d}$ by the angle $\omega = \arcsin(|\mathbf{b}|) - \arcsin(|\mathbf{b}| \cdot u)$ [S1, unofficial]. Query $\text{MLP}_{out}$ with input $(x_0, y_0, z_0, u, \mathbf{d})$ to obtain density $\sigma_{out}$ and color $c_{out}$ [S1, unofficial]. Integrate over $u \in [0, 1]$ to compute term (iii) [S1, unofficial].
4.  **Composite:** Combine the inner and outer contributions using the split volumetric rendering integral [S1, unofficial].

**Formulas**
The volumetric rendering integral $C(\mathbf{r})$ is split into two parts: integration inside the inner volume (terms i and ii) and integration in the outer volume (term iii) [S1, unofficial]:
$$C(\mathbf{r}) = \underbrace{\int_{t=0}^{t'} \sigma_{in} \cdot c_{in} \cdot e^{-\int_{s=0}^t \sigma_{in} ds} dt}_{\text{(i)}} + \underbrace{e^{-\int_{s=0}^{t'} \sigma_{in} ds}}_{\text{(ii)}} \cdot \underbrace{\int_{t=t'}^{\infty} \sigma_{out} \cdot c_{out} \cdot e^{-\int_{s=t'}^t \sigma_{out} ds} dt}_{\text{(iii)}}$$
A 3D point $(x, y, z)$ with radius $r = \sqrt{x^2 + y^2 + z^2} > 1$ is re-parameterized into a quadruple $(x_0, y_0, z_0, 1/r)$, where $(x_0, y_0, z_0)$ is a unit vector along the same direction as $(x, y, z)$ and $0 < 1/r < 1$ is the inverse radius [S1, unofficial]. All values are bounded: $x_0, y_0, z_0 \in [-1, 1]$ and $1/r \in [0, 1]$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_outer_params(o, d, num_samples=128):
    """
    Computes the geometric mapping for the outer volume of NeRF++.
    Args:
        o: Ray origin (3,)
        d: Ray direction (3,)
        num_samples: Number of samples to take for the outer volume integration.
    Returns:
        points_out: Array of shape (num_samples, 4) containing (x0, y0, z0, 1/r)
    """
    # 1. Intersection with unit sphere (point a)
    # Solve |o + t_a * d|^2 = 1 => t_a^2 + 2(o·d)t_a + |o|^2 - 1 = 0
    a_coeff = 1.0
    b_coeff = 2.0 * np.dot(o, d)
    c_coeff = np.dot(o, o) - 1.0
    discriminant = b_coeff**2 - 4 * a_coeff * c_coeff
    if discriminant < 0:
        return np.empty((0, 4))
    sqrt_disc = math.sqrt(discriminant)
    t_a = (-b_coeff - sqrt_disc) / (2 * a_coeff)
    a = o + t_a * d

    # 2. Midpoint of chord (point b)
    # Solve d·(o + t_b*d) = 0 => t_b = -(o·d) / |d|^2
    t_b = -np.dot(o, d)
    b = o + t_b * d
    b_norm = np.linalg.norm(b)

    # 3. Sample u = 1/r in [0, 1]
    u_samples = np.linspace(0.0, 1.0, num_samples)
    points_out = []
    for u in u_samples:
        # Rotation angle
        arg1 = min(1.0, max(-1.0, b_norm))
        arg2 = min(1.0, max(-1.0, b_norm * u))
        omega = math.asin(arg1) - math.asin(arg2)

        # Rotation axis
        axis = np.cross(b, d)
        axis_norm = np.linalg.norm(axis)

        if axis_norm < 1e-8:
            x0, y0, z0 = a
        else:
            axis = axis / axis_norm
            cos_w = math.cos(omega)
            sin_w = math.sin(omega)
            k_cross_v = np.cross(axis, a)
            k_dot_v = np.dot(axis, a)
            x0 = a[0]*cos_w + k_cross_v[0]*sin_w + axis[0]*k_dot_v*(1-cos_w)
            y0 = a[1]*cos_w + k_cross_v[1]*sin_w + axis[1]*k_dot_v*(1-cos_w)
            z0 = a[2]*cos_w + k_cross_v[2]*sin_w + axis[2]*k_dot_v*(1-cos_w)
        points_out.append([x0, y0, z0, u])
    return np.array(points_out)

def render_ray(o, d, mlp_in, mlp_out, num_inner=64, num_outer=128):
    """
    Renders a single ray using NeRF++ with Inverted Sphere Parameterization.
    Args:
        o: Ray origin (3,)
        d: Ray direction (3,)
        mlp_in: MLP function for inner volume
        mlp_out: MLP function for outer volume
        num_inner: Number of samples for inner volume
        num_outer: Number of samples for outer volume
    Returns:
        color: Rendered color (3,)
    """
    # Inner volume integration
    a_coeff = 1.0
    b_coeff = 2.0 * np.dot(o, d)
    c_coeff = np.dot(o, o) - 1.0
    disc = b_coeff**2 - 4 * a_coeff * c_coeff
    if disc < 0:
        t_max = 0.0
    else:
        t_max = (-b_coeff - math.sqrt(disc)) / (2 * a_coeff)
    t_max = max(t_max, 0.0)

    if t_max > 0:
        t_inner = np.linspace(0, t_max, num_inner)
        t_inner = (t_inner[:-1] + t_inner[1:]) / 2.0
        pts_inner = o + np.outer(t_inner, d)
        sigma_in, color_in = mlp_in(pts_inner, d)
        delta_t = t_max / num_inner
        alpha_in = 1.0 - np.exp(-sigma_in * delta_t)
        T_in = np.exp(-np.cumsum(sigma_in * delta_t))
        T_in = np.concatenate(([1.0], T_in[:-1]))
        color_inner = np.sum(T_in * alpha_in * color_in, axis=0)
        transmittance_at_boundary = T_in[-1]
    else:
        color_inner = np.zeros(3)
        transmittance_at_boundary = 1.0

    # Outer volume integration
    outer_params = compute_outer_params(o, d, num_outer)
    if outer_params.size == 0:
        return color_inner

    x0, y0, z0, u = outer_params[:, 0], outer_params[:, 1], outer_params[:, 2], outer_params[:, 3]
    pts_out_xyz = np.stack([x0, y0, z0], axis=1)
    sigma_out, color_out = mlp_out(pts_out_xyz, u, d)

    # [G] The source states to sample points from [0, 1] for 1/r to compute term (iii),
    # but does not provide the Jacobian |dt/du| required to convert the integral from t-space to u-space.
    # An implementer must derive dt/du from the geometric mapping t(u) to weight the samples correctly.
    du = 1.0 / num_outer
    alpha_out = 1.0 - np.exp(-sigma_out * du)
    T_out = np.exp(-np.cumsum(sigma_out * du))
    T_out = np.concatenate(([1.0], T_out[:-1]))
    color_outer = np.sum(T_out * alpha_out * color_out, axis=0)

    return color_inner + transmittance_at_boundary * color_outer
```

## Operating characteristics
*   **Time Complexity:** Training is time-consuming. Training NeRF++ on a node with 4 RTX 2080 Ti GPUs takes approximately 24 hours [S1, unofficial].
*   **Memory Complexity:** Memory-intensive due to two separate NeRFs (MLPs) for large-scale scenes [S1, unofficial].
*   **Rendering Performance:** Rendering a single 1280x720 image on one RTX 2080 Ti GPU takes approximately 30 seconds at test time [S1, unofficial].
*   **Strengths:** Enables photorealistic synthesis of 360° captures in unbounded scenes [S1, unofficial]. Improves numeric stability by bounding all input values to $[-1, 1]$ [S1, unofficial]. Respects the prior that farther objects should get less resolution (via $1/r$) [S1, unofficial].
*   **Limitations:** Rendering is far from real-time [S1, unofficial].
*   **When to use:** Use for real-world captures from datasets like Tanks and Temples or light field datasets where the scene is unbounded and 360° [S1, unofficial].

## Background
*   **Origin:** The method was introduced in the paper "NeRF++: Analyzing and Improving Neural Radiance Fields" by Yu et al. [S1, unofficial].
*   **History:** It was developed to address the failure modes of standard NeRF when applied to 360° captures of unbounded scenes, specifically the trade-off between modeling foreground detail and background coverage [S1, unofficial].
*   **Usage:** It has been applied to real-world captures from the Tanks and Temples dataset (Knapitsch et al., 2017) and the light field dataset of Yucer et al. (2016) [S1, unofficial].

## Sources
[S1] NeRF++: Analyzing and Improving Neural Radiance Fields, https://doi.org/10.48550/arxiv.2010.07492, fetched 2026-10-26 (unofficial)
