# Vectorial Volume Rendering

Also called: Direct Volume Rendering with Ray Casting, Vectorial Volume Ray Casting, Volume Rendering for Vectorial Volumes

## Overview
Vectorial Volume Rendering extends standard scalar direct volume ray casting to handle input data where each voxel contains a distribution of material probabilities rather than a single scalar intensity [S1, S2]. It solves the problem of visualizing classified multi-material volumes by computing gradients and optical properties from weighted material contributions, allowing for the simultaneous rendering of multiple tissue or material types with distinct visual attributes [S1, S2]. Unlike scalar methods that map a single value to color/opacity, this method determines sample properties based on the probabilistic mixture of materials, distinguishing it from siblings like [Standard Ray Casting](12643-standard-ray-casting.md) which assumes a single scalar field per voxel.

## How it works
**Inputs and Outputs**
*   **Input**: A set of $K$ material probability volumes $P_k$, where $P_k(x,y,z)$ represents the probability of the $k$-th material at each voxel location. A set of user-defined material properties for each material $k$, consisting of a color vector $\mathbf{C}_k$ (RGB) and an opacity scalar $\alpha_k$ [S1, S2].
*   **Output**: A 2D image where each pixel contains the composited color $\mathbf{C}_{out}$ and accumulated opacity $\alpha_{out}$ [S1, S2].

**Data Structures**
*   **Material Probability Volumes**: $K$ 3D arrays of floating-point values, normalized such that $\sum_{k=1}^{K} P_k(x,y,z) = 1$ for all voxels.
*   **Material Properties**: A list or array of size $K$, where each entry contains $\mathbf{C}_k \in \mathbb{R}^3$ and $\alpha_k \in [0, 1]$.
*   **Density Volume**: An intermediate 3D array $D$ of the same dimensions as the input volumes, used for gradient calculation.

**Numbered Steps**

1.  **Ray Definition**: For each pixel, define a ray $r(t) = \mathbf{o} + t\mathbf{d}$ intersecting the volume bounding box at $t_{near}$ and $t_{far}$ [G]. Discretize the segment into $N$ samples with step $\Delta t$ [G].
2.  **Sampling Material Probabilities**: For each sample position $\mathbf{p}_i$ along the ray, retrieve the probability $p_{k,i}$ for each material $k \in \{1, \dots, K\}$ using tri-linear interpolation from the $K$ probability volumes [S2].
3.  **Density Volume Construction**: To enable gradient calculation, construct a scalar density volume $D$ where the value at each voxel is the weighted sum of material opacities:
    $$
    D(x,y,z) = \sum_{k=1}^{K} p_k(x,y,z) \alpha_k
    $$
    Here, $p_k$ is the material probability and $\alpha_k$ is the user-assigned opacity for material $k$ [S1]. This replaces the traditional density $\rho_k$ to simplify user input by avoiding confusion between density and opacity [S1].
4.  **Gradient Calculation**: Compute the gradient $\nabla D(\mathbf{p}_i)$ at the current sample position $\mathbf{p}_i$ using the constructed density volume $D$. Any standard scalar volume gradient calculation method (e.g., central differences) may be applied [S1]. This gradient serves as the surface normal for shading [S1, S2].
5.  **Transfer Function and Shading**: Determine the sample color $\mathbf{c}_i$ and opacity $\alpha_i$ using one of two methods:
    *   **Class Decision Method**: Calculate the first derivative of each material probability $p_k$ along the ray direction. Select the material $k^*$ that yields the maximum derivative. Assign the sample color $\mathbf{c}_i = \mathbf{C}_{k^*}$ and opacity $\alpha_i = \alpha_{k^*}$ [S1].
    *   **Composed Color and Opacity Method**: Compute the sample color and opacity directly from the interpolated probabilities and material properties. The sample color is the weighted sum of material colors: $\mathbf{c}_i = \sum_{k=1}^{K} p_{k,i} \mathbf{C}_k$. The sample opacity is derived from the total density or a specific transfer function applied to the probabilities, often enhanced by boundary detection using the gradient $\nabla D$ to increase opacity in heterogeneous regions and decrease it in homogeneous ones [S1, S2].
    *   **Shading**: Apply Phong’s shading model using the gradient $\nabla D(\mathbf{p}_i)$ as the normal vector to modulate the base color $\mathbf{c}_i$ into a shaded color $\mathbf{c}'_i$ [S2].
6.  **Compositing**: Composite the shaded sample $(\mathbf{c}'_i, \alpha_i)$ into the accumulated pixel color and opacity using the front-to-back volume rendering equation [G]:
    $$
    \mathbf{C}_{acc} \leftarrow \mathbf{C}_{acc} + (1 - \alpha_{acc}) \cdot \mathbf{c}'_i \cdot \alpha_i
    $$
    $$
    \alpha_{acc} \leftarrow \alpha_{acc} + (1 - \alpha_{acc}) \cdot \alpha_i
    $$
    Initialize $\mathbf{C}_{acc} = (0,0,0)$ and $\alpha_{acc} = 0$ [G].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vectorial_volume_rendering(probability_volumes, material_properties, ray, num_samples):
    """
    probability_volumes: List of K 3D arrays (float), shape (Dx, Dy, Dz)
    material_properties: List of K tuples (color: (R,G,B), opacity: float)
    ray: Object with origin o, direction d, t_near, t_far
    num_samples: Integer N
    """
    K = len(probability_volumes)
    
    # Precompute Density Volume for Gradient Calculation
    # D[x,y,z] = sum(p_k * alpha_k)
    density_volume = zeros_like(probability_volumes[0])
    for k in range(K):
        p_k = probability_volumes[k]
        alpha_k = material_properties[k].opacity
        density_volume += p_k * alpha_k
        
    # Compute Gradient of Density Volume (e.g., using central differences)
    # grad_D is a 3D array of vectors (dx, dy, dz)
    grad_D = compute_gradient(density_volume)
    
    # Initialize Accumulators
    acc_color = (0.0, 0.0, 0.0)
    acc_alpha = 0.0
    
    # Ray Marching
    dt = (ray.t_far - ray.t_near) / num_samples
    
    for i in range(num_samples):
        t = ray.t_near + i * dt
        pos = ray.o + t * ray.d
        
        # 1. Sample Material Probabilities via Trilinear Interpolation
        probs = []
        for k in range(K):
            p_val = trilinear_interpolate(probability_volumes[k], pos)
            probs.append(p_val)
            
        # 2. Get Gradient at Position
        # Interpolate gradient from precomputed grad_D
        normal = trilinear_interpolate_vector(grad_D, pos)
        
        # 3. Determine Sample Color and Opacity
        # Using Composed Color and Opacity method as primary example
        # Note: Class Decision method would select max derivative material instead
        
        sample_color = (0.0, 0.0, 0.0)
        sample_alpha = 0.0
        
        # Compose Color
        for k in range(K):
            c_k = material_properties[k].color
            sample_color = (
                sample_color[0] + probs[k] * c_k[0],
                sample_color[1] + probs[k] * c_k[1],
                sample_color[2] + probs[k] * c_k[2]
            )
            
        # Compose Opacity (Simplified: weighted sum of opacities)
        # More complex transfer functions may use normal magnitude for boundary enhancement
        for k in range(K):
            alpha_k = material_properties[k].opacity
            sample_alpha += probs[k] * alpha_k
            
        # Clamp opacity
        sample_alpha = min(sample_alpha, 1.0)
        
        # 4. Shading (Phong Model)
        # Requires light direction L, view direction V, and normal N
        # Assuming standard Phong parameters are available
        shaded_color = apply_phong_shading(sample_color, normal, ray.d)
        
        # 5. Front-to-Back Compositing
        if sample_alpha > 0:
            # Contribution of current sample
            contribution = (1.0 - acc_alpha) * shaded_color * sample_alpha
            acc_color = (
                acc_color[0] + contribution[0],
                acc_color[1] + contribution[1],
                acc_color[2] + contribution[2]
            )
            acc_alpha += (1.0 - acc_alpha) * sample_alpha
            
    return acc_color, acc_alpha
```

**Parameters**
*   $K$: Number of materials. Typical values depend on the segmentation (e.g., 3-10 for medical tissues) [S1, S2].
*   $\alpha_k$: User-assigned opacity for material $k$. Range $[0, 1]$ [S1, S2].
*   $\mathbf{C}_k$: User-assigned color for material $k$. Range $[0, 1]$ per channel [S1, S2].

**Edge Cases and Pitfalls**
*   **Gradient Noise**: Since the density volume $D$ is a weighted sum of probabilities, noise in probability estimates can lead to noisy gradients, affecting shading quality [S1].
*   **Class Decision Ambiguity**: In the Class Decision method, if multiple materials have similar derivatives, the selection may be unstable [S1].
*   **Opacity Summation**: In the Composed method, simple summation of opacities can exceed 1.0 if not clamped or modeled physically (e.g., Beer-Lambert) [S2].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot K)$ per pixel, where $N$ is the number of samples and $K$ is the number of materials. This is linear in both $N$ and $K$, making it more expensive than scalar rendering ($O(N)$) due to the need to sample $K$ volumes and compute a weighted sum [S1, S2].
*   **Memory Complexity**: $O(V \cdot K)$ for storing $K$ probability volumes, plus $O(V)$ for the intermediate density volume and its gradient, where $V$ is the number of voxels [S1, S2].
*   **Strengths**: Allows for the simultaneous visualization of multiple classified materials with distinct colors and opacities without requiring separate rendering passes for each material [S1]. The use of a density volume for gradients provides a consistent normal field for shading across material boundaries [S1].
*   **Limitations**: Computationally heavier than scalar rendering due to multiple texture fetches and interpolation operations per sample [S2]. The quality of shading depends heavily on the accuracy of the probability volumes and the constructed density field [S1].
*   **When to Use**: When visualizing multi-material volumes where material boundaries and internal structures of different classes need to be seen simultaneously with distinct visual cues [S1, S2].
*   **When Not to Use**: For single-material scalar volumes where standard ray casting is sufficient and faster [S1].

## Background
Vectorial Volume Rendering was developed to address the limitations of scalar volume rendering when applied to classified multi-material data. Traditional scalar methods map a single value to optical properties, which is insufficient for representing the probabilistic nature of segmented medical or scientific data where voxels may contain mixtures of materials [S1, S2]. The method expands on the scalar volume rendering pipeline by introducing specific techniques for gradient calculation (using a weighted density volume) and transfer function design (Class Decision or Composed Color/Opacity) [S1]. It was proposed in the context of visualizing spatially aligned volumes after 3D/3D image registration, where multiple modalities or segmentations need to be merged and rendered together [S1, S2]. The approach builds on the foundational work of Drebin et al. regarding density-based gradient calculation for shading [S1, S2].

## Sources
[S1] A Multi-Volume Visualization Framework for Spatial Aligned Volumes after 3D/3D Image Registration, https://doi.org/10.1109/ICISE.2009.77, fetched 2026-10-26
[S2] A multivolume visualization method for spatially aligned volumes after 3D/3D image registration, https://doi.org/10.1016/j.irbm.2011.01.040, fetched 2026-10-26
