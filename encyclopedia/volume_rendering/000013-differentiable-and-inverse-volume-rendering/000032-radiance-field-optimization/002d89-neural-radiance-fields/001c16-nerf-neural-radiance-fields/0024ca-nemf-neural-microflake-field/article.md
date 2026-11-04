# NeMF (Neural Microflake Field)
Also called: Neural Microflake Field

## Overview
NeMF is a differentiable inverse volume rendering method that represents a scene as a distribution of microflakes at every point in space, enabling the recovery of physical attributes (shape, reflectance, illumination) from multi-view images under unknown natural illumination [S1]. Unlike standard NeRF which models view-dependent radiance, NeMF models light transportation via microflake normals and roughness, allowing it to handle complex geometry and translucent objects better than surface-based methods [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-view images of a scene, camera poses, and a bounding box defining the scene volume.
*   **Outputs:** A continuous volumetric representation defined by the implicit microflake field $\Phi$, which outputs volume density $\sigma$, albedo $a$, microflake normal $\boldsymbol{\omega}_m$, and roughness $\tau_m$ for any 3D position $\mathbf{p}$ [S1]. The method also outputs optimized Spherical Gaussian (SG) parameters for environment illumination and visibility fields [S1].

**Data Structures:**
The core data structure is the implicit microflake field, implemented via coordinate-based neural networks.
1.  **Geometry Network ($\mathcal{F}_g$):** An MLP that takes position $\mathbf{p}$ and outputs volume density $\sigma$ and microflake normal $\boldsymbol{\omega}_m$. This builds on the NeRF structure [S1].
2.  **Appearance Network ($\mathcal{D}_m$):** A U-shaped MLP that takes a latent code $\mathbf{z}$ (derived from position $\mathbf{p}$) to output albedo $a$ and roughness $\tau_m$. A sparsity constraint is applied to the latent code $\mathbf{z}$ [S1].
3.  **Visibility Network ($\mathcal{V}$):** An MLP that takes position $\mathbf{p}$ and outputs visibility SG parameters for each spatial position [S1].
4.  **Illumination Parameters:** A set of 128-dimensional Spherical Gaussians (SGs) representing the environment light [S1].

**Mathematical Definitions:**
The microflake model assumes space is filled with infinite small flakes. The probability of normals in direction $\boldsymbol{\omega}$ on the ellipsoid surface is modeled by the Normal Distribution Function (NDF) [S1]:
$$D(\boldsymbol{\omega}) = \frac{1}{\pi\sqrt{|S|}(\boldsymbol{\omega}^T S^{-1}\boldsymbol{\omega})^2}$$
where $|S|$ is the determinant of the shape matrix $S$ [S1].

The diffuse phase function is defined as [S1]:
$$f_p^d(\boldsymbol{\omega}_i, \boldsymbol{\omega}_l) = \frac{1}{\pi \tau(\boldsymbol{\omega}_i)} \int_{\Omega} \langle \boldsymbol{\omega}_l, \boldsymbol{\omega} \rangle \langle \boldsymbol{\omega}_i, \boldsymbol{\omega} \rangle D(\boldsymbol{\omega}) d\boldsymbol{\omega}$$
where $\langle \cdot \rangle$ is the dot product, $\Omega$ is the hemisphere centered at $\boldsymbol{\omega}_m$, and $\tau(\boldsymbol{\omega})$ is the projected size of the ellipsoid along direction $\boldsymbol{\omega}$ [S1].

The specular phase function is defined as [S1]:
$$f_p^s(\boldsymbol{\omega}_i, \boldsymbol{\omega}_l) = \frac{D(\boldsymbol{\omega}_h)}{4\tau(\boldsymbol{\omega}_i)}$$
where $\boldsymbol{\omega}_h$ is the half-angle between the input light direction $\boldsymbol{\omega}_l$ and viewing direction $\boldsymbol{\omega}_i$ [S1].

**Algorithm Steps:**
1.  **Initialization:** Initialize the geometry network $\mathcal{F}_g$, appearance network $\mathcal{D}_m$, visibility network $\mathcal{V}$, and SG illumination parameters. Pre-train a standard NeRF to initialize the density field and provide initial visibility estimates [S1].
2.  **Sampling:** For each ray $\mathbf{r}$, sample discrete positions $\mathbf{p}_k$ along the ray. [S1]
3.  **Feature Extraction:**
    *   Query $\mathcal{F}_g(\mathbf{p}_k)$ to get density $\sigma_k$ and normal $\boldsymbol{\omega}_{m,k}$ [S1].
    *   Encode $\mathbf{p}_k$ into a latent code $\mathbf{z}_k$ (e.g., via positional encoding or a shared encoder) and query $\mathcal{D}_m(\mathbf{z}_k)$ to get albedo $a_k$ and roughness $\tau_{m,k}$ [S1].
    *   Query $\mathcal{V}(\mathbf{p}_k)$ to get visibility SG parameters [S1].
4.  **Visibility Calculation:**
    *   To compute visibility of a point $\mathbf{p}_k$ with respect to light direction $\boldsymbol{\omega}_l$, march a ray from $\mathbf{p}_k$ towards the light in the background.
    *   Calculate opacity $\eta$ using the pre-trained NeRF density field [S1].
    *   Encode this visibility using SG parameterization [S1].
5.  **Radiance Computation:**
    *   Compute the diffuse radiance component $\nu_d(\mathbf{p}_k)$ by summing the product of environment lighting and visibility. The sources provide a specific approximation for faster computation:
        $$\nu_d(x) = \frac{\pi}{\pi} \sum_{\boldsymbol{\omega}_l \in \Omega} \{ [V(x, \boldsymbol{\omega}_l) Y(\boldsymbol{\omega}_l)] \otimes [L^{\text{sg}}(\boldsymbol{\omega}_l) Y(\boldsymbol{\omega}_l)] \cdot \langle \boldsymbol{\omega}_l \cdot \boldsymbol{\omega}_m \rangle \}$$
        (Note: The exact formula in the source text contains a typo $\frac{\pi}{\pi}$ which simplifies to 1, and the summation term structure is complex; implementers should refer to Eqn 10 and surrounding text in [S1] for the full integration of diffuse and specular components) [S1].
    *   Compute the specular radiance component $\nu_s(\mathbf{p}_k)$. Check visibility of the point w.r.t. light direction during ray sampling; set radiance to 0 if the light is invisible to the point [S1].
    *   Total radiance at $\mathbf{p}_k$ is $\nu^*(\mathbf{p}_k, \boldsymbol{\omega}_i) = \nu_d(\mathbf{p}_k) + \nu_s(\mathbf{p}_k)$ [S1].
6.  **Volume Rendering:**
    *   Replace integrals with summations over discrete sampling positions [S1].
    *   Perform importance ray sampling of light directions for calculating the final radiance, as elaborated in Algorithm 1 of Heitz et al. [22] (cited in [S1]) [S1].
    *   Render the image color $C(\mathbf{r})$ using standard volume rendering integration along the ray [S1].
7.  **Loss Calculation:**
    *   **Photometric Loss:** $\mathcal{L}_c = \sum_{\mathbf{r}, I} \|C(\mathbf{r}) - I(\mathbf{r})\|_2^2$ [S1].
    *   **Density Field Loss:** $\mathcal{L}_\sigma = \sum_k \eta_k (\|\boldsymbol{\omega}_m - \Delta \sigma\|_2^2 + \max(0, \langle \boldsymbol{\omega}_m, \boldsymbol{\omega}_i \rangle))$, where $\Delta \sigma$ is the gradient of $\sigma$ [S1].
    *   **Latent Sparsity Loss:** $\mathcal{L}_z = \sum_j KL(\rho \| \hat{\rho}_j)$, where $\hat{\rho}_j$ is the average of the $j$-th channel of $\mathbf{z}$ over the batch input, and $\rho$ is set to 0.05 [S1].
    *   **Smoothness Loss:** $\mathcal{L}_s = \|\mathcal{N}(\mathcal{F}_g(\mathbf{p})) - \mathcal{N}(\mathcal{F}_g(\mathbf{p} + \epsilon))\|_1 + \|\mathcal{D}_m(\mathbf{z}) - \mathcal{D}_m(\mathbf{z} + \epsilon)\|_1$, where $\epsilon$ is a small random variable from a Gaussian distribution with mean 0 and variance 0.01 [S1].
8.  **Optimization:** Minimize the total loss $\mathcal{L} = \mathcal{L}_c + \mathcal{L}_\sigma + \mathcal{L}_z + \mathcal{L}_s$ using gradient descent [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_nerf(scene_data, num_iterations):
    # Initialize networks
    Fg = GeometryMLP() # Outputs sigma, omega_m
    Dm = AppearanceMLP() # Outputs a, tau_m
    V = VisibilityMLP() # Outputs visibility SG params
    SG_Lights = InitializeSGs(dim=128) # Environment illumination
    
    # Pre-train NeRF for initial density and visibility
    NeRF = PretrainNeRF(scene_data)
    
    for iteration in range(num_iterations):
        total_loss = 0
        for ray in scene_data.rays:
            # 1. Sample points along ray
            points = SamplePoints(ray, num_samples=64)
            
            # 2. Extract features
            sigma_list = []
            omega_m_list = []
            albedo_list = []
            tau_m_list = []
            visibility_params_list = []
            
            for p in points:
                # Query geometry
                sigma, omega_m = Fg(p)
                sigma_list.append(sigma)
                omega_m_list.append(omega_m)
                
                # Query appearance
                z = EncodePosition(p) # Latent code
                a, tau_m = Dm(z)
                albedo_list.append(a)
                tau_m_list.append(tau_m)
                
                # Query visibility
                vis_params = V(p)
                visibility_params_list.append(vis_params)
            
            # 3. Compute Visibility
            # March ray from each point to light directions to get opacity eta
            # Use pre-trained NeRF for density integration
            visibilities = ComputeVisibility(points, omega_m_list, NeRF)
            
            # 4. Compute Radiance
            # Diffuse component
            # Note: Implementation of Eqn 10 requires summing over light directions
            # and convolving visibility with lighting SGs
            radiance_diffuse = ComputeDiffuseRadiance(
                points, omega_m_list, SG_Lights, visibility_params_list
            )
            
            # Specular component
            # Check visibility for each light direction
            radiance_specular = ComputeSpecularRadiance(
                points, omega_m_list, SG_Lights, visibilities
            )
            
            total_radiance = radiance_diffuse + radiance_specular
            
            # 5. Volume Rendering
            # Replace integral with summation
            rendered_color = VolumeRender(points, sigma_list, total_radiance, ray)
            
            # 6. Loss Calculation
            # Photometric loss
            gt_color = GetGroundTruthColor(ray)
            loss_c = L2Norm(rendered_color - gt_color)
            
            # Density field loss
            # Delta sigma is gradient of sigma
            delta_sigma = Gradient(sigma_list)
            loss_sigma = Sum(eta_k * (L2Norm(omega_m - delta_sigma) + Max(0, Dot(omega_m, ray.direction))))
            
            # Latent sparsity loss
            # KL divergence between rho (0.05) and average z channels
            loss_z = KL_Divergence(0.05, AverageChannels(z))
            
            # Smoothness loss
            epsilon = RandomGaussian(variance=0.01)
            loss_s = L1Norm(Normalize(Fg(p)) - Normalize(Fg(p + epsilon))) + \
                     L1Norm(Dm(z) - Dm(z + epsilon))
            
            total_loss += loss_c + loss_sigma + loss_z + loss_s
            
        # Backpropagate and update
        UpdateParameters(total_loss)
```

**Edge Cases and Pitfalls:**
*   **Normal Estimation:** Removing normal estimation ("w/o normal") bakes ambient lighting into surfaces, yielding significantly worse results (PSNR 15.4516 vs 23.6982) [S1].
*   **Visibility Modeling:** Assuming environment illumination is visible to all points ("w/o vis.") leads to incorrect predictions around occlusion boundaries [S1].
*   **Latent Space:** Mapping points directly to parameters without a latent space ("w/o latent space") results in incorrect material extraction and vaguer edges [S1].
*   **Precision:** The integration in the diffuse phase function requires careful numerical handling. The sources note that integrals are replaced with summations over discrete sampling positions [S1].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources. The method involves multiple MLP queries per sample point and visibility ray marching, which increases computational cost compared to standard NeRF.

**Measured Performance:**
On synthetic Blender scenes (balloon, mic, spot, polyhedron), NeMF achieved PSNR 23.6982, SSIM 0.9387 (Albedo), and PSNR 25.7048 (Relighting), outperforming InvRender, NeRFactor, and PhySG [S1].

**Strengths and Limitations:**
*   **Strengths:** Fully relies on volume representation without needing to recover geometry explicitly, allowing it to handle scenes with complex geometry and translucent objects (e.g., clouds) which surface-based methods cannot handle well [S1]. Enables high-quality relighting, material editing, and volume scattering simulation [S1].
*   **Limitations:** Surface-based methods (e.g., NeRD, NeRFactor) may be more efficient for simple geometry. NeMF's reliance on inverse volume rendering and visibility modeling adds complexity [S1].

**When to Use:**
Use NeMF when the scene contains complex geometry, translucent objects, or when high-quality relighting and material editing are required under unknown natural illumination. Avoid if computational resources are severely limited or if the scene is simple enough for surface-based methods [S1].

## Background
NeMF was proposed in the paper "NeMF: Inverse Volume Rendering with Neural Microflake Field" [S1]. It extends the NeRF framework by incorporating a microflake model for light transportation, addressing the limitations of surface-based inverse rendering methods [S1]. The method builds on prior work in inverse rendering, such as InvRender and NeRD, but distinguishes itself by fully utilizing a volume representation [S1].

## Sources
[S1] NeMF: Inverse Volume Rendering with Neural Microflake Field, https://doi.org/10.1109/iccv51070.2023.02095, fetched 2026-10-26
