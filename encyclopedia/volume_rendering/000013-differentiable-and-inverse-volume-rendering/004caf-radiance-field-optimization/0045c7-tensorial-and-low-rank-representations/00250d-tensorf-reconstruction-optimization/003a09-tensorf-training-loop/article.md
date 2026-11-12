# TensoRF Training Loop

Also called: TensoRF Optimization

## Overview
The TensoRF Training Loop is the iterative optimization procedure used to fit the tensorial radiance field parameters to a set of multi-view images. It solves the problem of minimizing the difference between rendered pixel colors and ground truth observations by performing gradient descent on the decomposed tensor components. It sets itself apart from standard NeRF optimization by operating on low-rank tensor factors (vector and matrix components) rather than a dense MLP, requiring specific regularization terms (orthogonal, L1, and Total Variation) to maintain the structural integrity and smoothness of the tensor decomposition [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   Training dataset: A collection of images, camera poses, and intrinsic parameters.
*   Scene bounding box (`aabb`): Defines the spatial extent of the scene.
*   Near/Far clipping planes (`near_far`): Depth range for ray sampling.
*   Background assumption (`white_bg`): Boolean indicating if the background is white.
*   NDC ray flag (`ndc_ray`): Boolean indicating if rays are normalized to [-1, 1].
*   Hyperparameters: `density_n_comp`, `appearance_n_comp`, `app_dim`, `shadingMode`, `alphaMask_thres`, `density_shift`, `distance_scale`, `pos_pe`, `view_pe`, `fea_pe`, `featureC`, `step_ratio`, `fea2denseAct`, `Ortho_reg_weight`, `L1_reg_weight`, `TV_weight_density`, `TV_weight_app`, `lr_decay_target_ratio`, `lr_decay_iters`, `batch_size`, `nSamples` [S1, unofficial].

**Outputs:**
*   Optimized tensor factors (density vectors/matrices, appearance vectors/matrices).
*   Rendered images, depth maps, and uncertainty maps for evaluation.

### Data Structures
The model `tensorf` stores the radiance field as decomposed tensors. Key fields include:
*   `density_n_comp`: Number of density components.
*   `appearance_n_comp`: Number of appearance components.
*   `app_dim`: Dimensionality of the appearance vector.
*   `featureC`: Feature channel dimension.
*   Internal tensor factors: Low-rank matrices and vectors representing the 3D grid.

### Numbered Steps

1.  **Initialization**:
    *   Load the training and test datasets.
    *   Extract scene parameters: `aabb`, `near_far`, `white_bg`, `ndc_ray` [S1, unofficial].
    *   Initialize the `tensorf` model with the specified hyperparameters (e.g., `density_n_comp`, `appearance_n_comp`, `app_dim`, etc.) [S1, unofficial].
    *   Calculate the learning rate decay factor:
        $$ lr\_factor = args.lr\_decay\_target\_ratio^{(1/args.lr\_decay\_its)} $$
        [S1, unofficial]
    *   If not using NDC rays, filter rays to the bounding box:
        `allrays, allrgbs = tensorf.filtering_rays(allrays, allrgbs, bbox_only=True)` [S1, unofficial].
    *   Initialize a `SimpleSampler` for batching rays [S1, unofficial].
    *   Initialize the optimizer (e.g., Adam) with the model parameters [S1, unofficial].

2.  **Iteration Loop** (for each iteration):
    *   **Ray Sampling**:
        *   Get ray indices: `ray_idx = trainingSampler.nextids()` [S1, unofficial].
        *   Fetch rays and ground truth RGBs:
            `rays_train, rgb_train = allrays[ray_idx], allrgbs[ray_idx].to(device)` [S1, unofficial].
    *   **Rendering**:
        *   Render the batch using the `renderer`. The number of samples per ray is calculated as:
            $$ nSamples = \min(args.nSamples, cal\_n\_samples(reso\_cur, args.step\_ratio)) $$
            [S1, unofficial]
        *   Call renderer:
            `rgb_map, alphas_map, depth_map, weights, uncertainty = renderer(rays_train, tensorf, chunk=args.batch_size, N_samples=nSamples, white_bg=white_bg, ndc_ray=ndc_ray, device=device, is_train=True)` [S1, unofficial].
    *   **Loss Computation**:
        *   Compute MSE loss:
            `loss = torch.mean((rgb_map - rgb_train) ** 2)` [S1, unofficial].
        *   **Orthogonal Regularization** (if `Ortho_reg_weight > 0`):
            *   Compute orthogonal difference: `loss_reg = tensorf.vector_comp_diffs()` [S1, unofficial].
            *   Add to total loss: `total_loss += Ortho_reg_weight * loss_reg` [S1, unofficial].
        *   **L1 Density Regularization** (if `L1_reg_weight > 0`):
            *   Compute L1 density: `loss_reg_L1 = tensorf.density_L1()` [S1, unofficial].
            *   Add to total loss: `total_loss += L1_reg_weight * loss_reg_L1` [S1, unofficial].
        *   **Total Variation (TV) Density Regularization** (if `TV_weight_density > 0`):
            *   Decay weight: `TV_weight_density *= lr_factor` [S1, unofficial].
            *   Compute TV loss: `loss_tv = tensorf.TV_loss_density(tvreg) * TV_weight_density` [S1, unofficial].
            *   Add to total loss: `total_loss += loss_tv` [S1, unofficial].
        *   **Total Variation (TV) Appearance Regularization** (if `TV_weight_app > 0`):
            *   Decay weight: `TV_weight_app *= lr_factor` [S1, unofficial].
            *   Compute TV loss: `loss_tv = tensorf.TV_loss_app(tvreg) * TV_weight_app` [S1, unofficial].
            *   Add to total loss: `total_loss += loss_tv` [S1, unofficial].
    *   **Optimization Step**:
        *   Zero gradients: `optimizer.zero_grad()` [S1, unofficial].
        *   Backpropagate: `total_loss.backward()` [S1, unofficial].
        *   Step optimizer: `optimizer.step()` [S1, unofficial].
        *   Update learning rate for all parameter groups:
            `param_group['lr'] = param_group['lr'] * lr_factor` [S1, unofficial].
    *   **Evaluation and Updates**:
        *   Periodically evaluate test PSNR and log metrics to TensorBoard [S1, unofficial].
        *   If current iteration is in `update_AlphaMask_list` and current voxel count < $256^3$, update the volume resolution (alpha mask) [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class TensoRFOptimizer:
    def __init__(self, dataset, model, args):
        self.dataset = dataset
        self.model = model
        self.args = args
        self.device = args.device
        
        # 1. Initialization
        self.aabb, self.near_far, self.white_bg, self.ndc_ray = self.dataset.get_scene_params()
        
        # Calculate LR decay factor
        self.lr_factor = args.lr_decay_target_ratio ** (1.0 / args.lr_decay_iters)
        
        # Filter rays if not NDC
        if not self.ndc_ray:
            self.allrays, self.allrgbs = self.model.filtering_rays(
                self.dataset.allrays, self.dataset.allrgbs, bbox_only=True
            )
        else:
            self.allrays = self.dataset.allrays
            self.allrgbs = self.dataset.allrgbs
            
        self.sampler = SimpleSampler(len(self.allrays))
        self.optimizer = torch.optim.Adam(self.model.parameters(), lr=args.lr)
        
        # Pre-calculate constants for TV loss
        self.tvreg = None # Assuming TVLoss module is initialized within model or separately

    def compute_tv_loss(self, tensor, weight, is_density=True):
        """
        Computes Total Variation loss for density or appearance components.
        Based on TVLoss implementation in [S1, unofficial].
        """
        # Note: The exact tensor shape passed here depends on the internal structure of tensorf.
        # Assuming tensor is of shape (Batch, C, H, W) or similar spatial grid.
        # The source code in [S1, unofficial] shows TVLoss operates on 4D tensors.
        
        batch_size = tensor.size()[0]
        h_x = tensor.size()[2]
        w_x = tensor.size()[3]
        
        count_h = self._tensor_size(tensor[:,:,1:,:])
        count_w = max(self._tensor_size(tensor[:,:,:,1:]), 1)
        
        h_tv = torch.pow((tensor[:,:,1:,:] - tensor[:,:,:h_x-1,:]), 2).sum()
        w_tv = torch.pow((tensor[:,:,:,1:] - tensor[:,:,:,:w_x-1]), 2).sum()
        
        loss = 2 * (h_tv / count_h + w_tv / count_w) / batch_size
        return weight * loss

    def _tensor_size(self, t):
        return t.size()[1] * t.size()[2] * t.size()[3]

    def train_step(self, iteration):
        # 2. Ray Sampling
        ray_idx = self.sampler.nextids()
        rays_train = self.allrays[ray_idx].to(self.device)
        rgb_train = self.allrgbs[ray_idx].to(self.device)
        
        # Calculate nSamples
        reso_cur = self.model.get_current_resolution()
        nSamples = min(self.args.nSamples, cal_n_samples(reso_cur, self.args.step_ratio))
        
        # 3. Rendering
        rgb_map, alphas_map, depth_map, weights, uncertainty = self.model.renderer(
            rays_train, 
            self.model, 
            chunk=self.args.batch_size, 
            N_samples=nSamples, 
            white_bg=self.white_bg, 
            ndc_ray=self.ndc_ray, 
            device=self.device, 
            is_train=True
        )
        
        # 4. Loss Computation
        loss = torch.mean((rgb_map - rgb_train) ** 2)
        total_loss = loss
        
        # Orthogonal Regularization
        if self.args.Ortho_reg_weight > 0:
            loss_reg = self.model.vector_comp_diffs()
            total_loss += self.args.Ortho_reg_weight * loss_reg
            
        # L1 Density Regularization
        if self.args.L1_reg_weight > 0:
            loss_reg_L1 = self.model.density_L1()
            total_loss += self.args.L1_reg_weight * loss_reg_L1
            
        # TV Density Regularization
        if self.args.TV_weight_density > 0:
            self.args.TV_weight_density *= self.lr_factor
            # Assuming model provides access to density tensor for TV loss
            density_tensor = self.model.get_density_tensor() 
            loss_tv = self.compute_tv_loss(density_tensor, self.args.TV_weight_density, is_density=True)
            total_loss += loss_tv
            
        # TV Appearance Regularization
        if self.args.TV_weight_app > 0:
            self.args.TV_weight_app *= self.lr_factor
            # Assuming model provides access to appearance tensor for TV loss
            app_tensor = self.model.get_appearance_tensor()
            loss_tv = self.compute_tv_loss(app_tensor, self.args.TV_weight_app, is_density=False)
            total_loss += loss_tv
            
        # 5. Optimization Step
        self.optimizer.zero_grad()
        total_loss.backward()
        self.optimizer.step()
        
        # Update LR
        for param_group in self.optimizer.param_groups:
            param_group['lr'] = param_group['lr'] * self.lr_factor
            
        # 6. Evaluation and Updates
        if iteration % self.args.eval_interval == 0:
            self.evaluate_test_set()
            
        if iteration in self.args.update_AlphaMask_list:
            if self.model.get_voxel_count() < 256**3:
                self.model.update_volume_resolution()

    def evaluate_test_set(self):
        # Placeholder for test evaluation logic
        pass
```

### Parameters
*   `N_voxel_init`: Initial number of voxels [S1, unofficial].
*   `nSamples`: Number of samples per ray, calculated as `min(args.nSamples, cal_n_samples(reso_cur, args.step_ratio))` [S1, unofficial].
*   `lr_decay_target_ratio`: Target learning rate ratio for decay [S1, unofficial].
*   `Ortho_reg_weight`: Weight for orthogonal regularization [S1, unofficial].
*   `L1_reg_weight_inital`: Initial weight for L1 density regularization [S1, unofficial].
*   `TV_weight_density`: Initial weight for TV density regularization [S1, unofficial].
*   `TV_weight_app`: Initial weight for appearance regularization [S1, unofficial].

### Edge Cases and Pitfalls
*   **Voxel Count Limit**: The volume resolution update is only triggered if the current voxel count is less than $256^3$ [S1, unofficial].
*   **TV Loss Decay**: The TV regularization weights decay by `lr_factor` at every step, which may reduce their influence over time [S1, unofficial].
*   **NDC Filtering**: If NDC rays are not used, rays must be filtered to the bounding box to ensure valid sampling [S1, unofficial].

## Subtopics

### [NeuManifold](9771-neumanifold.md)
NeuManifold is a three-stage pipeline for reconstructing neural watertight manifolds with efficient, high-quality rendering support. It solves the problem of generating high-fidelity 3D meshes from 2D images by combining the robust initialization of volume rendering with the geometric precision of differentiable rasterization. Unlike pure volumetric methods, it produces explicit triangle meshes; unlike pure mesh-based methods, it avoids the artifacts caused by linear interpolation in non-linear density fields by using a differentiable marching cubes algorithm.

### [Separable Component-Wise Convolution for Joint Pose Optimization](10576-separable-component-wise-convolution-for-joint-pose-optimiza.md)
This method enables robust joint optimization of camera poses and decomposed low-rank tensorial radiance fields by controlling the frequency spectrum of the scene representation. It prevents premature high-frequency signals in the voxel volume from causing camera poses to get stuck in local minima, a failure mode common in naive joint optimization. It sets itself apart from hash-encoding-based methods by specifically leveraging the separability of the input low-rank tensor signal, making it more efficient than traditional separable convolution kernels.

### StyleTRF
StyleTRF is a method for stylized view generation that decouples style adaptation from view capture by fine-tuning the appearance component of a pre-optimized Tensorial Radiance Field (TensoRF) using sparse stylized priors [S1, unofficial]. It sets itself apart from joint optimization methods like SNeRF by freezing the geometric density parameters of the scene representation, thereby preserving crisp geometry while adapting appearance, and by using a deterministic CNN-based stylization module (Johnson et al.) to ensure 3D consistency across viewpoints [S1, unofficial].

### [TensoRF Decomposition Models](13296-tensorf-decomposition-models.md)
This family defines the factorization structures for the radiance field tensor in TensoRF, balancing storage efficiency against rendering quality by decomposing the 3D field into lower-rank components. Unlike the standard Vector-Matrix (VM) decomposition which enforces a fixed 1:1 ratio of vector to matrix ranks, these models allow arbitrary adjustment of this ratio, providing greater flexibility in model size control.

### TensoRF Downsampling
Downsamples both train and test data by a factor of 4 [S2, unofficial]. Training iterations are reduced to 2,500 (vs. 30,000 in original TensoRF paper) due to memory and time constraints [S2, unofficial]. Tested on the synthetic Lego scene from the NeRF dataset [4] with 100 training views and 200 test views [S2, unofficial].

### Depth Supervised TensoRF
Applies depth supervision loss during training [S2, unofficial]. A key pitfall is low PSNR with low data. There is minimal improvement (0.01 PSNR) over the non-depth supervised approach because COLMAP key points are orders of magnitude fewer than color pixel comparisons, causing depth loss gradients to affect only a small portion of voxel space [S2, unofficial].

## Operating characteristics

*   **Time Complexity**: The training loop iterates for a fixed number of steps (e.g., 30,000 for full data, 2,500 for downsampled) [S1, S2, unofficial]. Each step involves ray sampling, rendering (which scales with $N_{samples}$ and tensor factor dimensions), and backpropagation through the tensor operations.
*   **Memory Complexity**: Depends on the tensor decomposition ranks (`density_n_comp`, `appearance_n_comp`) and the voxel resolution. The low-rank structure significantly reduces memory compared to dense voxel grids or large MLPs [S1, unofficial].
*   **Measured Performance**: On the synthetic Lego scene with full data, TensoRF achieves a test PSNR of 29.92 [S2, unofficial]. With downsampling and low data (100 views), PSNR drops to 15.23 [S2, unofficial].
*   **Strengths**: Efficient memory usage due to tensor decomposition; fast rendering via low-rank factors; effective regularization (Ortho, L1, TV) for smoothness [S1, unofficial].
*   **Limitations**: Sensitive to initialization; depth supervision shows minimal benefit in low-data regimes due to sparse keypoint coverage [S2, unofficial].
*   **When to Use**: Suitable for scenes where memory is constrained or fast training is desired, and high-quality multi-view images are available.
*   **When Not to Use**: Not ideal for scenarios requiring explicit depth accuracy with sparse input views, as depth supervision provides negligible gains [S2, unofficial].

## Background

TensoRF was introduced in the paper "Tensorial Radiance Fields" (ECCV 2022) [S1, unofficial]. It represents the radiance field as a tensor decomposition, enabling efficient storage and rendering. The training loop incorporates specific regularization techniques to handle the low-rank constraints, distinguishing it from standard NeRF optimization which uses an MLP [S1, unofficial].

## Sources

[S1] GitHub - apchenstu/TensoRF: [ECCV 2022] Tensorial Radiance Fields, a ..., https://github.com/apchenstu/TensoRF, fetched 2026-10-29 (unofficial)
[S2] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26 (unofficial)
