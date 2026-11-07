# Discrete Volume Rendering Integral (VRI)
Also called: Pass 2 Fragment Shader VRI Evaluation

## Overview
The Discrete Volume Rendering Integral (VRI) computes the final pixel color by discretizing the continuous volume rendering equation into a Riemann sum along a ray segment [S1, unofficial]. It projects a 3D scalar field onto a 2D image by iterating from the entry texture coordinate $s_{in}$ to the exit texture coordinate $s_{out}$, sampling attenuation and color, and compositing results front-to-back [S1, unofficial]. What sets it apart from [Ray Marching Volume Rendering](1735-ray-marching-volume-rendering.md) and [Ray Evaluation Stepping DVR](3461-ray-evaluation-stepping-dvr.md) is its specific implementation as a fragment shader operation that relies on pre-computed entry/exit texture coordinates and a strict fixed-step $\Delta s$ derived from world-space slice spacing, rather than adaptive stepping [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Entry texture coordinate $s_{in}$, exit texture coordinate $s_{out}$, slice spacing in world coordinates $\Delta x$, 3D volume texture (attenuation), optional 1D transfer function, optional gradient texture, backlight color $L_{backlight}$ [S1, unofficial].
*   **Output:** Final pixel color $L_{out}$ [S1, unofficial].

**Data Structures:**
*   **3D Volume Texture:** Stores attenuation values $\tau(s)$ at 3D texture coordinates [S1, unofficial].
*   **1D Transfer Function (Optional):** Maps attenuation $\tau$ to opacity $\tau'$ and color coefficient $k_d$ [S1, unofficial].
*   **Gradient Texture (Optional):** Stores volume gradients for shading [S1, unofficial].

**Parameters:**
*   $\Delta x$: Slice spacing in world coordinates [S1, unofficial].
*   $L_{backlight}$: Light coming from behind the volume, added at the end of the loop [S1, unofficial].

**Steps:**
1.  **Compute $\Delta s$:** Calculate the texture coordinate step size $\Delta s$ corresponding to the world-space step $\Delta x$ [S1, unofficial].
    *   Determine the spatial locations of $s_{in}$ and $s_{out}$ using the bounding box dimensions $x_{dim}, y_{dim}, z_{dim}$ [S1, unofficial].
    *   Calculate the total spatial length of the ray segment inside the volume.
    *   Divide this length by $\Delta x$ to find the number of steps $N$ [S1, unofficial].
    *   $\Delta s$ is the difference in texture coordinates corresponding to one $\Delta x$ step, computed as $(s_{out} - s_{in}) / N$ [S1, unofficial].
2.  **Initialize Accumulators:** Set $L_{out} \leftarrow (0, 0, 0)$ and $t \leftarrow 1$ (accumulated transmittance) [S1, unofficial].
3.  **Iterate along Ray:** For $s$ from $s_{in}$ to $s_{out}$ with step $\Delta s$:
    *   **Sample Attenuation:** Lookup $\tau(s)$ from the 3D volume texture at coordinate $s$ [S1, unofficial].
    *   **Apply Transfer Function (Optional):** If a transfer function exists, lookup $\tau'$ (opacity) and $k_d$ (color coefficient) using $\tau(s)$ [S1, unofficial]. Otherwise, $\tau' = \tau(s)$ and $k_d = 1$ [S1, unofficial].
    *   **Compute Opacity:** $\alpha = \tau(s) \; \Delta x$ [S1, unofficial].
    *   **Compute Color Contribution $C(s)$:** Compute $C(s)$ using the volume gradient (e.g., Phong equation) or other methods, potentially using the gradient texture and transfer function $k_d$ [S1, unofficial].
    *   **Accumulate Color:** $L_{out} \leftarrow L_{out} + t \; C(s) \; \alpha$ [S1, unofficial].
    *   **Update Transmittance:** $t \leftarrow t \; (1 - \alpha)$ [S1, unofficial].
4.  **Add Backlight:** $L_{out} \leftarrow L_{out} + t \; L_{backlight}$ [S1, unofficial].
5.  **Return:** $L_{out}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_vri(s_in, s_out, delta_x, volume_texture, transfer_function, gradient_texture, backlight_color):
    # Step 1: Compute delta_s
    # Convert texture coordinates to spatial positions (linear mapping from [0,1]^3 to bounding box)
    p_in = s_in * [x_dim, y_dim, z_dim]
    p_out = s_out * [x_dim, y_dim, z_dim]
    spatial_dist = sqrt(sum((p_out - p_in)**2))
    N = round(spatial_dist / delta_x)
    if N <= 0:
        return backlight_color
    ds = (s_out - s_in) / N
    
    L_out = (0.0, 0.0, 0.0)
    t = 1.0
    
    # Step 3: Iterate
    for i in range(N):
        s = s_in + i * ds
        tau = volume_texture.sample_3d(s)
        
        if transfer_function is not None:
            tau_prime, k_d = transfer_function.sample_1d(tau)
        else:
            tau_prime = tau
            k_d = 1.0
            
        alpha = tau * delta_x
        
        # C(s) computation: sources state it uses volume gradient (Phong equation) 
        # or other methods, potentially using gradient texture and k_d. Exact shading math is not provided in sources.
        # C_s = ... # Not given in sources [S1, unofficial]
        
        L_out = L_out + t * C_s * alpha
        t = t * (1.0 - alpha)
        
    # Step 4: Add backlight
    L_out = L_out + t * backlight_color
    return L_out
```

**Edge Cases and Pitfalls:**
*   **Computing $\Delta s$:** A common pitfall is using a fixed texture coordinate step instead of deriving it from the world-space slice spacing $\Delta x$ [S1, unofficial]. $\Delta s$ must be calculated by dividing the spatial segment into pieces of length $\Delta x$ and finding the corresponding texture coordinate difference [S1, unofficial].
*   **Precision:** Accumulating small $\alpha$ values over many steps can lead to precision issues. Using high-precision accumulators is recommended [G].
*   **Boundary Conditions:** Ensure $s_{in}$ and $s_{out}$ are correctly clamped to the volume bounds to avoid sampling outside the texture [G].

## Subtopics

### [Discrete Volume Rendering Integral](1334-discrete-volume-rendering-integral.md)
Classical Ray Tracing Integration is a global, image-order rendering approach that determines visible objects at the pixel level by searching for the nearest visible object from the eye point, contrasting with rasterization. It solves the problem of rendering participating media by tracing a ray from the camera through the volume, computing illumination in small segments along the ray while attenuating light by the density between the camera and the segment. It explicitly integrates the volume rendering integral using ray-object intersection logic to define the integration bounds, rather than relying solely on fixed-step discretization of a pre-defined volume boundary.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to Use:** Not given in the sources.
*   **When Not to Use:** Not given in the sources.

## Background
The Discrete Volume Rendering Integral (VRI) formulation originates from GPU-based volume rendering techniques, specifically as implemented in fragment shaders for direct volume rendering [S1, unofficial]. It is a discretization of the continuous volume rendering equation, adapted for efficient rasterization-based rendering pipelines [S1, unofficial]. The method is widely used in medical imaging and scientific visualization for real-time exploration of 3D scalar fields [S1, unofficial].

## Sources
[S1] CISC/CMPE 454, GPU Volume Rendering, https://watkins.cs.queensu.ca/~jstewart/454/notes/44/44-gpu-volume-rendering.html, fetched 2026-10-30 (unofficial)
