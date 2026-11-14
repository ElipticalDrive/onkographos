# Discrete Volume Rendering Integral (VRI)
Also called: Pass 2 Fragment Shader VRI Evaluation

## Overview
The Discrete Volume Rendering Integral (VRI) is the fundamental numerical approximation used in direct volume ray casting to compute the color and opacity of a pixel by integrating the continuous Volume Rendering Equation along a camera ray [S1, unofficial]. It solves the problem of rendering 3D scalar or vector fields by discretizing the ray into fixed or variable segments, sampling the volume data at each step, and compositing the results front-to-back [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Entry texture coordinate $s_{in}$, exit texture coordinate $s_{out}$, slice spacing in world coordinates $\Delta x$, spatial dimensions of the bounding box $x_{dim}, y_{dim}, z_{dim}$, attenuation values $\tau(s)$, emitted color $C(s)$, and backlight $L_{backlight}$ [S1, unofficial].
*   **Output:** Final pixel color $L_{out}$ [S1, unofficial].

**Data Structures:**
*   **3D Volume Texture:** Stores attenuation values $\tau(s)$ [S1, unofficial].
*   **1D Transfer Function Texture:** Maps attenuation $\tau'$ to color $k_d$ (if present) [S1, unofficial].
*   **Gradient Texture:** Stores volume gradients for shading (if present) [S1, unofficial].

**Numbered Steps:**
1.  **Compute $\Delta s$:** Calculate the step size in texture coordinates. The slice spacing $\Delta x$ is a world-space measure [S1, unofficial]. Compute the spatial locations of the entry and exit points from their texture coordinates. Divide the spatial segment between these points into pieces of length $\Delta x$. The difference in texture coordinates corresponding to one piece of length $\Delta x$ is $\Delta s$ [S1, unofficial].
2.  **Initialize Accumulators:** Set output color $L_{out} = (0, 0, 0)$ and accumulated transparency $t = 1$ [S1, unofficial].
3.  **Iterate Along Ray:** Loop from $s = s_{in}$ to $s_{out}$ with step $\Delta s$ [S1, unofficial].
    a.  **Sample Attenuation:** Lookup $\tau(s)$ from the 3D volume texture. If a transfer function exists, lookup $\tau'$ and $k_d$ from the 1D texture [S1, unofficial].
    b.  **Compute Opacity:** Calculate $\alpha = \tau(s) \; \Delta x$ [S1, unofficial].
    c.  **Compute Color:** Determine $C(s)$ using volume gradients (e.g., Phong equation) or other shading methods, potentially using gradient textures and $k_d$ [S1, unofficial].
    d.  **Composite:** Update $L_{out} = L_{out} + t \; C(s) \; \alpha$ [S1, unofficial].
    e.  **Update Transparency:** Update $t = t \; (1 - \alpha)$ [S1, unofficial].
4.  **Add Backlight:** Return $L_{out} + t \; L_{backlight}$, where $L_{backlight}$ is the light coming from behind the volume [S1, unofficial].

**Formulas:**
The VRI is defined as:
$$L_\textrm{out} = \displaystyle \sum_{i=0}^n C_i\ \alpha_i\ \left(\ \prod_{j=0}^{i-1}\ (1-\alpha_j)\ \right)$$
where $t_i = \prod_{j=0}^{i-1}\ (1-\alpha_j)$ denotes the total transparency up to step $i$, representing the attenuation of light generated in the $i^\textrm{th}$ interval before leaving the volume through the entry point toward the eye [S1, unofficial]. At the entry point, $t_0 = 1$ [S1, unofficial].

**Parameters:**
*   $\Delta x$: Slice spacing in world coordinates [S1, unofficial].
*   $\Delta s$: Spacing in texture coordinates along the ray, derived from $\Delta x$ and spatial dimensions [S1, unofficial].
*   $L_{backlight}$: Light coming from behind the volume [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Computing $\Delta s$:** A common pitfall is confusing world-space spacing with texture-coordinate spacing. $\Delta s$ must be computed by mapping the world-space slice spacing $\Delta x$ to the texture coordinate domain based on the specific ray's orientation and the bounding box dimensions [S1, unofficial].
*   **Precision:** Accumulating transparency $t$ can suffer from precision loss if $\alpha$ is very small or large over many steps [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_vri(s_in, s_out, delta_x, x_dim, y_dim, z_dim, volume_texture, transfer_function, gradient_texture, backlight):
    # Step 1: Compute delta_s
    # Map texture coordinates to world space (bounding box centered at origin)
    def tex_to_world(s):
        return ((s[0] - 0.5) * x_dim, (s[1] - 0.5) * y_dim, (s[2] - 0.5) * z_dim)

    spatial_in = tex_to_world(s_in)
    spatial_out = tex_to_world(s_out)
    spatial_dir = (spatial_out[0] - spatial_in[0], spatial_out[1] - spatial_in[1], spatial_out[2] - spatial_in[2])
    spatial_len = (spatial_dir[0]**2 + spatial_dir[1]**2 + spatial_dir[2]**2)**0.5

    tex_dir = (s_out[0] - s_in[0], s_out[1] - s_in[1], s_out[2] - s_in[2])
    tex_len = (tex_dir[0]**2 + tex_dir[1]**2 + tex_dir[2]**2)**0.5

    delta_s = (delta_x * tex_dir[0] / spatial_len, delta_x * tex_dir[1] / spatial_len, delta_x * tex_dir[2] / spatial_len)
    num_steps = int((spatial_len / delta_x) + 0.999999)  # ceil

    # Step 2: Initialize Accumulators
    L_out = (0.0, 0.0, 0.0)
    t = 1.0

    # Step 3: Iterate Along Ray
    for i in range(num_steps):
        s = (s_in[0] + i * delta_s[0], s_in[1] + i * delta_s[1], s_in[2] + i * delta_s[2])

        # Step 3a: Sample Attenuation
        tau = volume_texture.sample(s)
        if transfer_function is not None:
            tau_prime, k_d = transfer_function.sample(tau)
        else:
            tau_prime = tau
            k_d = (1.0, 1.0, 1.0)

        # Step 3b: Compute Opacity
        alpha = tau * delta_x

        # Step 3c: Compute Color
        grad = gradient_texture.sample(s)
        # Standard Phong-like shading using gradient and transfer function color [G]
        light_dir = (0.0, 0.0, 1.0)
        grad_len = (grad[0]**2 + grad[1]**2 + grad[2]**2)**0.5
        if grad_len > 1e-6:
            norm_grad = (grad[0]/grad_len, grad[1]/grad_len, grad[2]/grad_len)
        else:
            norm_grad = (0.0, 0.0, 1.0)
        diffuse = max(0.0, norm_grad[0]*light_dir[0] + norm_grad[1]*light_dir[1] + norm_grad[2]*light_dir[2])
        C_s = (k_d[0] * (0.2 + 0.8 * diffuse), k_d[1] * (0.2 + 0.8 * diffuse), k_d[2] * (0.2 + 0.8 * diffuse))

        # Step 3d: Composite
        L_out = (L_out[0] + t * C_s[0] * alpha, L_out[1] + t * C_s[1] * alpha, L_out[2] + t * C_s[2] * alpha)

        # Step 3e: Update Transparency
        t = t * (1.0 - alpha)

    # Step 4: Add Backlight
    return (L_out[0] + t * backlight[0], L_out[1] + t * backlight[1], L_out[2] + t * backlight[2])
```

## Subtopics

### [Discrete Volume Rendering Integral](1334-discrete-volume-rendering-integral.md)
The Discrete Volume Rendering Integral (VRI) is the fundamental numerical approximation used in direct volume ray casting to compute the color and opacity of a pixel by integrating the continuous Volume Rendering Equation (VRE) along a camera ray. It solves the problem of rendering 3D scalar or vector fields by discretizing the ray into fixed or variable segments, sampling the volume data at each step, and compositing the results front-to-back. It sets itself apart from analytical methods like [Self-Attenuation Integral Evaluation](1085-self-attenuation-integral-evaluation.md) by relying on Riemann sums rather than closed-form solutions, and from higher-order integration schemes like [Runge-Kutta 4 (RK4) Integration](3158-runge-kutta-4-rk4-integration.md) by assuming constant density and color within each segment (Zero Order Integration).

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] CISC/CMPE 454, GPU Volume Rendering, https://watkins.cs.queensu.ca/~jstewart/454/notes/44/44-gpu-volume-rendering.html, fetched 2026-10-30 (unofficial)
