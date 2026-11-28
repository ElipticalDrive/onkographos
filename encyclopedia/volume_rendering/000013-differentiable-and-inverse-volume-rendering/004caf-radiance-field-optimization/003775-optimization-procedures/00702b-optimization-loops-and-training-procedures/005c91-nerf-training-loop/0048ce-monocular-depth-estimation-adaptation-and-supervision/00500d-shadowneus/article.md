# ShadowNeuS

Also called: Neural SDF Reconstruction by Shadow Ray Supervision

## Overview
ShadowNeuS reconstructs a neural Signed Distance Field (SDF) of a scene from single-view images under multiple lighting conditions by supervising shadow rays rather than relying solely on camera rays [S1]. It solves the problem that standard NeRF does not model rays from the scene to the light source, missing shadow information which contains critical geometry cues via light visibility [S1]. Unlike NeRF which uses multi-view cameras as ray sensors, this method uses a single-view camera and shadow rays to provide supervision, making it applicable to real capture setups with limited views [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Single-view RGB images or binary shadow images captured under multiple known light directions. The scene is assumed to be opaque. To bound the scene, camera rays that do not intersect the SDF are projected onto a ground plane, whose position and orientation are assumed known to resolve scale ambiguity [S1].
*   **Outputs:** A neural SDF network $f$ representing the scene geometry, and optionally material properties (diffuse and specular coefficients) if RGB inputs are used [S1].

**Data Structures:**
*   **SDF Network $f$:** An MLP that takes a 3D position $\mathbf{x}$ and outputs a scalar signed distance value $\text{sdf}(\mathbf{x})$ [S1]. For RGB inputs, it also outputs a 256-dimensional feature vector [S1].
*   **Material Network $g$:** An MLP that takes 3D position $\mathbf{x}$ and surface normal $\mathbf{n}$ to regress material properties: diffuse albedo $\rho_d$ and specular coefficients $\mathbf{y}$ [S1].
*   **Light Source:** Defined as either a directional light (uniform direction $\mathbf{l}$ and intensity $L$) or a point light (location $\mathbf{q}$, scalar intensity $L_p$) [S1].

**Parameters:**
*   **Samples per image:** 256 pixel positions are sampled as supervision signals [S1].
*   **Batch size:** 4 images randomly selected in each batch [S1].
*   **Light sampling:** 100 directional lights and 100 point lights selected randomly on the upper hemisphere [S1].
*   **Training iterations:** 150k [S1].
*   **Input resolution:** 800x800 for synthetic datasets [S1].
*   **SDF Sigmoid Parameter $s$:** A learnable scalar controlling the transition between volume and surface rendering [S1].

**Step-by-Step Method:**

1.  **Scene Representation:** Represent the scene as the zero level set of the SDF network $f$, i.e., $S = \{ \mathbf{u} \in \mathbb{R}^3 \mid f(\mathbf{u}) = 0 \}$ [S1].

2.  **Camera Ray Intersection:** For each camera ray $\mathbf{r} = (\mathbf{o}, \mathbf{v})$ starting at camera center $\mathbf{o}$ with direction $\mathbf{v}$, compute the intersection point $\mathbf{x}$ with the SDF using ray marching [S1].
    *   *Pitfall Handling:* If the camera ray does not intersect the SDF, it is set to intersect with the ground plane to bound the scene [S1].

3.  **Differentiable Intersection:** To make the intersection point $\mathbf{x}$ differentiable for backpropagation, apply implicit differentiation [S1]:
    $$ \tilde{\mathbf{x}} = \mathbf{x} - \frac{\mathbf{v}}{\mathbf{n} \cdot \mathbf{v}} \text{sdf}(\mathbf{x}) $$
    where $\mathbf{n} = \nabla_{\mathbf{x}} f(\mathbf{x})$ is the surface normal derived from the SDF network [S1].

4.  **Boundary Detection:** Identify boundary pixels using a surface walk procedure [S1]. This yields a pixel subset $\Omega$ corresponding to surface boundaries and a differentiable area ratio $w$ for each boundary pixel [S1].

5.  **Shadow Ray Setup:** Illuminate the scene with a light source in direction $\mathbf{l}$ [S1].
    *   For **directional lights**, $\mathbf{l}$ and $L$ are known and uniform [S1].
    *   For **point lights** at location $\mathbf{q}$ with intensity $L_p$, calculate direction and intensity at point $\tilde{\mathbf{x}}$:
        $$ L = \frac{L_p}{\| \mathbf{q} - \tilde{\mathbf{x}} \|_2^2}, \quad \mathbf{l} = \frac{\mathbf{q} - \tilde{\mathbf{x}}}{\| \mathbf{q} - \tilde{\mathbf{x}} \|_2} $$
        [S1]

6.  **Shadow Ray Rendering (Incoming Irradiance):** Compute the incoming irradiance at the surface point $\tilde{\mathbf{x}}$ by marching shadow rays from $\tilde{\mathbf{x}}$ in direction $\mathbf{l}$ [S1].
    *   **Opacity Calculation:** Derive opacity $\alpha_i$ from the SDF using the CDF of SDF values at adjacent samples $t_i$ and $t_{i+1}$ along the shadow ray:
        $$ \alpha_i = \max \left( 1 - \frac{\Phi_s(\text{sdf}(\mathbf{p}(t_{i+1})))}{\Phi_s(\text{sdf}(\mathbf{p}(t_i)))}, 0 \right) $$
        where $\Phi_s(x) = (1 + e^{-sx})^{-1}$ is the sigmoid function and $s$ is a learnable scalar [S1].
    *   **Irradiance Formula:** The estimated incoming irradiance is:
        $$ \text{irradiance}(\tilde{\mathbf{x}}, \mathbf{l}) = L \prod_{i=1}^{N} (1 - \alpha_i) $$
        [S1]

7.  **Boundary Handling:** For pixels in the boundary subset $\Omega$, a single shadow ray may lead to incorrect gradients because the pixel spans disconnected regions at different depths [S1].
    *   Locate two intersections $\mathbf{x}_n$ (near) and $\mathbf{x}_f$ (far) at different depths within the pixel [S1].
    *   Compute their incoming irradiance $\text{irradiance}(\tilde{\mathbf{x}}_n, \mathbf{l})$ and $\text{irradiance}(\tilde{\mathbf{x}}_f, \mathbf{l})$ [S1].
    *   Average them using the area ratio $w$:
        $$ \text{irradiance}_{\text{pix}} = \begin{cases} \text{irradiance}(\tilde{\mathbf{x}}, \mathbf{l}) & \text{pixel} \notin \Omega \\ w \cdot \text{irradiance}(\tilde{\mathbf{x}}_n, \mathbf{l}) + (1-w) \cdot \text{irradiance}(\tilde{\mathbf{x}}_f, \mathbf{l}) & \text{pixel} \in \Omega \end{cases} $$
        [S1]

8.  **Outgoing Radiance (RGB Inputs Only):** If inputs are RGB, decompose the outgoing radiance using inverse rendering. Model the non-Lambertian BRDF as a diffuse component $\rho_d$ and a specular component $\rho_s$ [S1].
    *   Regress material properties $(\rho_d, \mathbf{y}) = g(\mathbf{x})$ using the material network $g$ at surface location $\mathbf{x}$ [S1].
    *   Model specular component using spherical Gaussian basis: $\rho_s = \mathbf{y}^T D(\mathbf{h}, \mathbf{n})$, where $\mathbf{h} = \frac{\mathbf{l} - \mathbf{v}}{\| \mathbf{l} - \mathbf{v} \|}$ is the half-vector and $D$ is the specular basis [S1].
    *   Compute outgoing radiance:
        $$ C(\tilde{\mathbf{x}}, -\mathbf{v}) = (\rho_d + \rho_s) \cdot \text{irradiance}(\tilde{\mathbf{x}}, \mathbf{l}) \cdot (\mathbf{l} \cdot \mathbf{n}) $$
        [S1]

9.  **Loss Computation:**
    *   **Eikonal Loss:** Regularize the SDF to output valid distances:
        $$ \mathcal{L}_{\text{eik}} = \frac{1}{M} \sum_{i}^{M} (\| \nabla f(\mathbf{p}_i) \|_2 - 1)^2 $$
        [S1]
    *   **Shadow Loss (Binary Inputs):**
        $$ \mathcal{L}_{\text{shadow}} = \| \text{irradiance}_{\text{pix}} - I_{\text{shadow}} \|_1 $$
        where $I_{\text{shadow}}$ is the binary shadow image pixel value [S1].
    *   **RGB Loss (RGB Inputs):**
        $$ \mathcal{L}_{\text{rgb}} = \| \widehat{C} - I_{\text{rgb}} \|_1 $$
        where $\widehat{C}$ is the computed outgoing radiance and $I_{\text{rgb}}$ is the RGB image pixel value [S1].

10. **Optimization:** Minimize the total loss ($\mathcal{L}_{\text{eik}} + \mathcal{L}_{\text{shadow}}$ or $\mathcal{L}_{\text{eik}} + \mathcal{L}_{\text{rgb}}$) to update the SDF network $f$ and material network $g$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class ShadowNeuS:
    def __init__(self, sdf_net, mat_net, s_init=10.0, num_shadow_samples=64):
        self.sdf_net = sdf_net
        self.mat_net = mat_net
        self.s = nn.Parameter(torch.tensor(s_init))
        self.num_shadow_samples = num_shadow_samples

    def sigmoid_cdf(self, sdf_val):
        """Phi_s(x) = (1 + exp(-s*x))^-1"""
        return 1.0 / (1.0 + torch.exp(-self.s * sdf_val))

    def compute_opacity(self, sdf_near, sdf_far):
        """alpha = max(1 - Phi_s(sdf_far) / Phi_s(sdf_near), 0)"""
        phi_near = self.sigmoid_cdf(sdf_near)
        phi_far = self.sigmoid_cdf(sdf_far)
        phi_near = torch.clamp(phi_near, min=1e-6)
        alpha = 1.0 - (phi_far / phi_near)
        return torch.clamp(alpha, min=0.0, max=1.0)

    def implicit_differentiation(self, x, v, sdf_val, grad_sdf):
        """x_tilde = x - (v / (n . v)) * sdf(x)"""
        n = grad_sdf
        dot_nv = torch.sum(n * v, dim=-1, keepdim=True)
        dot_nv = torch.clamp(dot_nv, min=1e-6)
        return x - (v / dot_nv) * sdf_val

    def ray_march_intersection(self, origin, direction, max_steps=128):
        """Standard ray marching to find SDF zero crossing."""
        t = 0.0
        step_size = 0.01
        for _ in range(max_steps):
            point = origin + t * direction
            sdf_val = self.sdf_net(point)
            if sdf_val < 0.0:
                grad_sdf = torch.autograd.grad(sdf_val.sum(), point, create_graph=True)[0]
                return point, sdf_val, grad_sdf
            t += step_size
        point = origin + t * direction
        sdf_val = self.sdf_net(point)
        grad_sdf = torch.autograd.grad(sdf_val.sum(), point, create_graph=True)[0]
        return point, sdf_val, grad_sdf

    def march_shadow_ray(self, origin, direction, light_intensity):
        """Compute irradiance = L * prod(1 - alpha_i) along shadow ray."""
        t_vals = torch.linspace(0.0, 1.0, self.num_shadow_samples)
        ray_points = origin.unsqueeze(0) + direction.unsqueeze(0) * t_vals.unsqueeze(-1)
        sdf_vals = self.sdf_net(ray_points)
        alphas = self.compute_opacity(sdf_vals[:-1], sdf_vals[1:])
        one_minus_alpha = 1.0 - alphas
        total_transmittance = torch.prod(one_minus_alpha)
        return light_intensity * total_transmittance

    def get_light_params(self, light_type, light_pos, light_intensity, point_x):
        if light_type == 'directional':
            return light_pos, light_intensity
        else:
            diff = light_pos - point_x
            dist_sq = torch.sum(diff**2, dim=-1)
            L = light_intensity / (dist_sq + 1e-6)
            l_dir = diff / (torch.sqrt(dist_sq) + 1e-6)
            return l_dir, L

    def compute_material(self, x, n):
        """Regress diffuse and specular coefficients."""
        input_feat = torch.cat([x, n], dim=-1)
        output = self.mat_net(input_feat)
        rho_d = output[..., :1]
        y = output[..., 1:]
        return rho_d, y

    def compute_specular(self, y, l_dir, v_dir, n):
        """rho_s = y^T D(h, n). D is a predefined specular basis."""
        diff = l_dir - v_dir
        norm_diff = torch.norm(diff, dim=-1, keepdim=True) + 1e-6
        h = diff / norm_diff
        # [G] Standard implementation evaluates basis functions D_k(h, n) and dots with y.
        # Here we assume a simple evaluation for completeness.
        basis_vals = torch.ones_like(y)
        return torch.sum(y * basis_vals, dim=-1, keepdim=True)

    def find_boundary_intersections(self, origin, direction, is_boundary):
        """Simple ray search to locate two intersections for boundary pixels."""
        t_vals = torch.linspace(0.0, 1.0, 128)
        ray_pts = origin.unsqueeze(0) + direction.unsqueeze(0) * t_vals.unsqueeze(-1)
        sdf_vals = self.sdf_net(ray_pts)
        crossings = torch.where(sdf_vals[:-1] * sdf_vals[1:] < 0)[0]
        if len(crossings) >= 2:
            xn = ray_pts[crossings[0]]
            xf = ray_pts[crossings[1]]
            return xn, xf, torch.tensor(0.5)
        return origin, origin, torch.tensor(0.5)

    def forward(self, camera_rays, light_data, shadow_images=None, rgb_images=None, boundary_mask=None, area_ratio=None):
        B, H, W = camera_rays.shape[:3]
        total_loss = 0.0
        num_samples = 256
        indices = torch.randint(0, H*W, (B, num_samples))

        for b in range(B):
            rays_o = camera_rays[b, :, :, 0]
            rays_v = camera_rays[b, :, :, 1]
            flat_o = rays_o.view(-1, 3)
            flat_v = rays_v.view(-1, 3)
            sampled_o = flat_o[indices[b]]
            sampled_v = flat_v[indices[b]]

            x, sdf_val, grad_sdf = self.ray_march_intersection(sampled_o, sampled_v)
            x_tilde = self.implicit_differentiation(x, sampled_v, sdf_val, grad_sdf)

            l_dir, L = self.get_light_params(light_data['type'], light_data['pos'], light_data['intensity'], x_tilde)
            irradiance = self.march_shadow_ray(x_tilde, l_dir, L)

            is_boundary = boundary_mask[b].view(-1)[indices[b]]

            if is_boundary.any():
                xn, xf, w = self.find_boundary_intersections(sampled_o, sampled_v, is_boundary)
                irradiance_near = self.march_shadow_ray(xn, l_dir, L)
                irradiance_far = self.march_shadow_ray(xf, l_dir, L)
                w = area_ratio[b].view(-1)[indices[b]]
                irradiance_final = w * irradiance_near + (1 - w) * irradiance_far
            else:
                irradiance_final = irradiance

            if shadow_images is not None:
                target = shadow_images[b].view(-1)[indices[b]]
                loss = torch.abs(irradiance_final - target).mean()
                total_loss += loss
            elif rgb_images is not None:
                n = grad_sdf
                rho_d, y = self.compute_material(x_tilde, n)
                rho_s = self.compute_specular(y, l_dir, sampled_v, n)
                l_dot_n = torch.sum(l_dir * n, dim=-1, keepdim=True)
                outgoing_radiance = (rho_d + rho_s) * irradiance_final * l_dot_n
                target = rgb_images[b].view(-1, 3)[indices[b]]
                loss = torch.abs(outgoing_radiance - target).mean()
                total_loss += loss

            random_points = torch.randn_like(sampled_o) * 2.0
            sdf_random = self.sdf_net(random_points)
            grad_random = torch.autograd.grad(sdf_random.sum(), random_points, create_graph=True)[0]
            eikonal = (torch.norm(grad_random, dim=-1) - 1.0)**2
            total_loss += eikonal.mean()

        return total_loss / B
```

## Operating characteristics
*   **Time Complexity:** Training takes approximately 24 hours on a single RTX 2080Ti for 150k iterations [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Evaluated on synthetic datasets (800x800 resolution) and real capture setups [S1].
*   **Strengths:**
    *   Reconstructs geometry from scratch using shadows, unlike methods that use MVS first [S1].
    *   Fully differentiable shadow ray supervision, unlike concurrent works using non-differentiable root finding [S1].
    *   Applicable to single-view setups, unlike NeRF which requires multi-view [S1].
    *   Handles RGB inputs by modeling material, avoiding the need for explicit shadow segmentation [S1].
*   **Limitations:**
    *   Requires multiple lighting conditions to obtain sufficient shadow observations [S1].
    *   Assumes the ground plane position and orientation are known to resolve scale ambiguity [S1].
    *   Iterative updating strategy is insufficient for gradients at surface boundaries with abrupt depth changes without the boundary handling mechanism [S1].
    *   Transparency and inter-reflections are disabled as these effects are beyond the method's assumptions [S1].

## Background
ShadowNeuS was proposed in the paper "ShadowNeuS: Neural SDF Reconstruction by Shadow Ray Supervision" [S1]. It addresses the limitation of NeRF, which models camera rays but ignores shadow rays (rays from the light source to the scene) [S1]. The method draws inspiration from NeuS for SDF representation and differentiable rendering techniques [S1]. It is used for single-view 3D reconstruction from binary shadow images or RGB images under multiple lights [S1].

## Sources
[S1] ShadowNeuS: Neural SDF Reconstruction by Shadow Ray Supervision, https://doi.org/10.1109/cvpr52729.2023.00025, fetched 2026-10-26
