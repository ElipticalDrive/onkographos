# On-the-Fly GS Progressive Training

Also called: On-the-Fly Gaussian Splatting

## Overview
On-the-Fly GS Progressive Training is a method for optimizing 3D Gaussian Splatting (3DGS) fields in a streaming or sequential input context, enabling near real-time reconstruction as new images arrive [S1]. It solves the problem of incremental scene expansion without retraining the entire model from scratch, which is computationally prohibitive for large-scale or dynamic capture scenarios [S1]. Unlike offline training methods that process all images simultaneously, this approach partitions the optimization trajectory into sequential phases, using hierarchical image weighting and adaptive learning rates to prioritize new data while maintaining consistency with previously registered views [S1]. It distinguishes itself from siblings like [Two-stage Optimization](14857-two-stage-optimization.md) by integrating densification and pose refinement directly into the per-image incremental update loop, rather than separating geometry initialization from appearance optimization [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   A sequence of incoming RGB images.
*   An initial sparse point cloud and camera poses estimated by an On-the-Fly SfM module [S1].
*   The current 3DGS field (set of Gaussians with position, covariance, opacity, and spherical harmonics coefficients) [S1].

**Outputs:**
*   An updated 3DGS field optimized for the newly acquired image and its neighbors [S1].
*   Updated camera poses and sparse point cloud via local Bundle Adjustment (BA) [S1].

### Data Structures
*   **3DGS Field:** Standard 3DGS representation. Each Gaussian $g$ has position $\mathbf{p}$, rotation $\mathbf{q}$, scale $\mathbf{s}$, opacity $\alpha$, and SH coefficients $\mathbf{c}$ [S1].
*   **Sparse Point Cloud:** A set of 3D points $\mathbf{X}$ with associated depths and colors, used for initializing new Gaussians [S1].
*   **Search Tree:** A spatial index (e.g., KD-tree or Octree) built over the existing sparse point cloud to efficiently query distances for new points [S1].
*   **Image Overlap Matrix $M$:** A matrix where $M_{i,j}$ stores the number of matched feature points between image $i$ and image $j$ [S1].
*   **Training State:** Per-image counters for iterations trained ($T_j$) and current learning rates ($\eta_j$) [S1].

### Numbered Steps

#### Phase 1: Initial 3DGS Training
1.  **Initialization:** Wait until $N_0$ initial images are acquired.
2.  **SfM:** Run On-the-Fly SfM to estimate poses and generate a sparse point cloud for these $N_0$ images [S1].
3.  **Training:** Train the initial 3DGS field using the standard 3DGS rendering and optimization pipeline for a fixed number of iterations (e.g., 2,000) [S1]. This prevents overfitting to the small initial subset [S1].

#### Phase 2: Progressive 3DGS Training (Per New Image)
For each newly acquired image $n$:
1.  **Pose and Cloud Update:** On-the-Fly SfM estimates the pose of image $n$ and performs local BA to optimize poses of neighboring images, refining the sparse point cloud [S1].
2.  **Search Tree Construction:** Build a search tree over the current sparse point cloud [S1].
3.  **New Point Identification:** Query the sparse points from image $n$ against the search tree. Points with a shortest distance to the existing cloud exceeding a predefined threshold are classified as "newly added" [S1].
4.  **Gaussian Initialization:** Initialize new Gaussians at the locations of these newly added points. Append them to the current 3DGS field [S1].
5.  **Hierarchical Image Weighting:** Calculate weights for training images based on their overlap with the new image $n$:
    *   **Layer 1 (New Image):** Weight $\omega_{11} = 1$ [S1].
    *   **Layer 2 (Direct Overlap):** For images $i$ with direct overlap ($M_{i,n} \neq 0$), calculate weight:
        $$ \omega_{2i} = M_{i,n} $$
        where $M_{i,n}$ is the number of matched feature points between image $i$ and new image $n$ [S1].
    *   **Layer $k$ (Indirect Overlap, $k \ge 3$):** For images in layer $k$, calculate weight based on weights of images in layer $k-1$:
        $$ \omega_{ki} = \frac{\sum_{j=1}^{N_{k-1}} \omega_{j}^{(k-1)} \times M_{j,ki}^{(k-1)}}{N_{k-1}} $$
        where $\omega_{j}^{(k-1)}$ is the weight of image $j$ in layer $k-1$, $M_{j,ki}^{(k-1)}$ is the match count between image $j$ and image $i$, and $N_{k-1}$ is the number of images in layer $k-1$ [S1].
6.  **Training Iteration Allocation:** Perform $T_I$ total training iterations on the current 3DGS field. Split these into Local and Semi-Global optimization:
    *   **Local Optimization:** Select the top $N_m$ images with the highest weights. For each key image $i$ (where $i \in \{1, \dots, N_m\}$), calculate its specific iteration count $T_{im}$:
        $$ T_{im} = \frac{1 - e^{-\omega_i}}{\sum_{j=1}^{N_m} (1 - e^{-\omega_j})} \times T_I $$
        Train these images for $T_{im}$ iterations [S1].
    *   **Semi-Global Optimization:** Distribute the remaining $T_I / 2$ iterations evenly among all other registered images (those not in the top $N_m$) [S1].
7.  **Adaptive Learning Rate Update:** For each image $j$ trained for $T_j$ iterations, update its learning rate $\eta_j$:
    *   Calculate progress $t_i$:
        $$ t_i = \begin{cases} 1 & \text{if } T_j > T_a \\ T_j / T_a & \text{if } T_j \le T_a \end{cases} $$
        where $T_a$ is the predefined iterations required for satisfactory quality [S1].
    *   Calculate base learning rate $\eta_j^c$:
        $$ \eta_j^c = (\eta_j^0)^{(1-t_i)} \times (\eta_j^f)^{t_i} $$
        where $\eta_j^0$ is the initial learning rate and $\eta_j^f$ is the final learning rate [S1].
    *   Adjust for overlapping images. Let $I$ be the set of images overlapping with image $j$ ($M_{i,j} > 0$) and $N_I$ be the count of such images. The final learning rate $\eta_j$ is:
        $$ \eta_j = \frac{\eta_j^c + \sum_{i \in I} M_{i,j} \times \eta_i^c}{N_I + 1} $$
        [S1].
8.  **Loss Calculation and Optimization:** Compute the total loss for the current batch of images and perform gradient descent. The loss function includes a load-balancing term:
    $$ L_{total} = \lambda_1 L_1 + \lambda_{ssim} L_{ssim} + \lambda_{load} L_{load} $$
    where $L_{load} = \text{std}(G_p)$ is the standard deviation of the number of Gaussians $G_p$ associated with each pixel $p$ in the image [S1].

#### Phase 3: Final Optimization
1.  **Global BA:** Once all images are acquired, perform global Bundle Adjustment on the estimated poses [S1].
2.  **Rapid Optimization:** Apply additional optimization iterations to the current 3DGS field to improve overall rendering quality [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

class OnTheFlyGS:
    def __init__(self, N0=30, TI=200, Nm=10, Ta=200, eta0=1.6e-4, etaf=1.6e-6, 
                 lambda1=0.47, lambda_load=0.41, lambda_ssim=0.12):
        self.N0 = N0
        self.TI = TI
        self.Nm = Nm
        self.Ta = Ta
        self.eta0 = eta0
        self.etaf = etaf
        self.lambda1 = lambda1
        self.lambda_load = lambda_load
        self.lambda_ssim = lambda_ssim
        
        self.gs_field = None  # 3D Gaussian Splatting field
        self.sparse_cloud = None  # Sparse point cloud
        self.search_tree = None  # Spatial index for sparse cloud
        self.image_overlaps = {}  # Dict: (img_i, img_j) -> match_count
        self.image_iterations = {}  # Dict: img_id -> total_iterations_trained
        self.image_learning_rates = {}  # Dict: img_id -> current_learning_rate
        
    def phase1_initial_training(self, initial_images, sfm_module):
        """
        Phase 1: Train initial 3DGS field with N0 images.
        """
        # 1. SfM for initial images
        poses, sparse_cloud = sfm_module.estimate_poses_and_cloud(initial_images[:self.N0])
        self.sparse_cloud = sparse_cloud
        
        # 2. Initialize 3DGS field
        self.gs_field = initialize_3dgs_from_cloud(sparse_cloud)
        
        # 3. Train for fixed iterations (e.g., 2000)
        # Note: Standard 3DGS training loop is assumed to be available
        train_3dgs(self.gs_field, initial_images[:self.N0], iterations=2000)
        
        # Initialize tracking structures
        for img in initial_images[:self.N0]:
            self.image_iterations[img.id] = 2000
            self.image_learning_rates[img.id] = self.etaf  # Assume converged to final rate

    def phase2_progressive_training(self, new_image, sfm_module):
        """
        Phase 2: Process a single new fly-in image.
        """
        # 1. Update Poses and Cloud
        # On-the-Fly SfM updates pose of new_image and neighbors, refines cloud
        self.sparse_cloud = sfm_module.update_pose_and_cloud(new_image, self.sparse_cloud)
        
        # 2. Construct Search Tree
        self.search_tree = build_search_tree(self.sparse_cloud)
        
        # 3. Identify New Points
        new_points = []
        for point in new_image.sparse_points:
            dist = self.search_tree.query_min_distance(point)
            if dist > distance_threshold: # Threshold defined by implementation
                new_points.append(point)
                
        # 4. Initialize New Gaussians
        new_gaussians = initialize_gaussians_from_points(new_points)
        self.gs_field.append(new_gaussians)
        
        # 5. Hierarchical Image Weighting
        # Calculate weights relative to new_image
        weights = calculate_hierarchical_weights(new_image, self.image_overlaps, self.Nm)
        
        # 6. Training Iteration Allocation
        # Select top Nm images
        top_images = get_top_n_images_by_weight(weights, self.Nm)
        
        # Calculate iterations for top images
        top_iterations = {}
        sum_denom = sum(1 - math.exp(-weights[img.id]) for img in top_images)
        for img in top_images:
            w = weights[img.id]
            t_im = ((1 - math.exp(-w)) / sum_denom) * self.TI
            top_iterations[img.id] = t_im
            
        # Semi-global: remaining TI/2 iterations distributed evenly among others
        other_images = [img for img in all_registered_images if img not in top_images]
        semi_global_iters = (self.TI / 2) / len(other_images) if other_images else 0
        
        # 7. Adaptive Learning Rate Update & Optimization
        # We perform TI iterations total. 
        # Local optimization: train top_images for their allocated t_im
        # Semi-global: train other_images for semi_global_iters each
        
        # Update learning rates before optimization
        for img in top_images + other_images:
            self._update_learning_rate(img)
            
        # Perform optimization steps
        # Note: In practice, this involves rendering and backprop for the specified images
        # The exact scheduling of TI iterations across images is implementation-specific
        # but must respect the counts t_im and semi_global_iters.
        
        for img in top_images:
            train_3dgs_step(self.gs_field, [img], iterations=top_iterations[img.id], 
                            learning_rate=self.image_learning_rates[img.id],
                            loss_fn=self._compute_loss)
            
        for img in other_images:
            train_3dgs_step(self.gs_field, [img], iterations=semi_global_iters,
                            learning_rate=self.image_learning_rates[img.id],
                            loss_fn=self._compute_loss)
                            
        # Update iteration counters
        for img in top_images:
            self.image_iterations[img.id] += top_iterations[img.id]
        for img in other_images:
            self.image_iterations[img.id] += semi_global_iters

    def _update_learning_rate(self, image):
        """
        Update learning rate for a specific image based on its training progress.
        """
        Tj = self.image_iterations[image.id]
        
        # Calculate progress ti
        if Tj > self.Ta:
            ti = 1.0
        else:
            ti = Tj / self.Ta
            
        # Calculate base learning rate eta_j^c
        eta_j_c = (self.eta0 ** (1 - ti)) * (self.etaf ** ti)
        
        # Find overlapping images
        overlapping_images = []
        for other_img_id, match_count in self.image_overlaps.items():
            # Assuming image_overlaps is symmetric or stored appropriately
            # Check if other_img overlaps with current image
            if match_count > 0:
                overlapping_images.append((other_img_id, match_count))
                
        # Calculate adjusted learning rate
        numerator = eta_j_c
        denominator = 1
        sum_overlap_eta = 0
        
        for other_img_id, M_ij in overlapping_images:
            # Get learning rate of overlapping image
            eta_i_c = self.image_learning_rates.get(other_img_id, self.eta0)
            sum_overlap_eta += M_ij * eta_i_c
            denominator += 1 # N_I + 1
            
        final_eta = (eta_j_c + sum_overlap_eta) / denominator
        self.image_learning_rates[image.id] = final_eta

    def _compute_loss(self, rendered_image, ground_truth_image, gs_field):
        """
        Compute total loss including Load-Balancing-Loss.
        """
        # L1 Loss
        l1 = compute_l1_loss(rendered_image, ground_truth_image)
        
        # SSIM Loss
        ssim = compute_ssim_loss(rendered_image, ground_truth_image)
        
        # Load-Balancing-Loss
        # Count Gaussians per pixel
        gaussian_counts_per_pixel = count_gaussians_per_pixel(rendered_image, gs_field)
        l_load = np.std(gaussian_counts_per_pixel)
        
        total_loss = (self.lambda1 * l1) + (self.lambda_ssim * ssim) + (self.lambda_load * l_load)
        return total_loss

    def phase3_final_optimization(self, all_images, sfm_module):
        """
        Phase 3: Final global optimization.
        """
        # 1. Global BA
        sfm_module.global_bundle_adjustment(self.sparse_cloud, all_images)
        
        # 2. Rapid additional optimization
        train_3dgs(self.gs_field, all_images, iterations=1000) # Iterations not specified, assume reasonable number

def calculate_hierarchical_weights(new_image, overlaps, Nm):
    """
    Calculate hierarchical weights for images based on overlap with new_image.
    """
    weights = {new_image.id: 1.0} # Layer 1
    
    # Layer 2: Direct overlap
    layer2_images = []
    for (img_i, img_j), match_count in overlaps.items():
        if img_i == new_image.id:
            weights[img_j] = match_count
            layer2_images.append(img_j)
        elif img_j == new_image.id:
            weights[img_i] = match_count
            layer2_images.append(img_i)
            
    # Layer 3+: Indirect overlap
    # This is a simplified representation. In practice, BFS/DFS is used to build layers.
    # For the purpose of the algorithm description, we assume a function that
    # recursively calculates weights based on previous layer weights and match counts.
    # The formula is: omega_ki = sum(omega_j^(k-1) * M_ji) / N_(k-1)
    
    # Note: Full implementation requires maintaining layer structure.
    # Here we just return the weights calculated so far for direct neighbors.
    # Indirect weights would be computed iteratively until convergence or max depth.
    
    return weights

def get_top_n_images_by_weight(weights, n):
    """
    Get top n images by weight.
    """
    sorted_images = sorted(weights.items(), key=lambda x: x[1], reverse=True)
    return [img_id for img_id, _ in sorted_images[:n]]
```

### Parameters
*   $N_0$: Number of initial images for Phase 1. Typical value: 30 [S1].
*   Initial training iterations for Phase 1. Typical value: 2,000 iterations [S1].
*   $T_I$: Total training iterations performed for each new fly-in image in Phase 2. Typical value: 200 [S1].
*   $N_m$: Number of top-weighted images selected as key training targets for local optimization. Typical value: 10 [S1].
*   $T_a$: Predefined parameter representing iterations required for satisfactory rendering quality. Typical value: 200 [S1].
*   $\eta_0$: Initial learning rate. Typical value: $1.6 \times 10^{-4}$ [S1].
*   $\eta_f$: Final learning rate. Typical value: $1.6 \times 10^{-6}$ [S1].
*   $\lambda_1$: Weight for $L_1$-Loss. Typical value: 0.47 [S1].
*   $\lambda_{load}$: Weight for Load-Balancing-Loss. Typical value: 0.41 [S1].
*   $\lambda_{ssim}$: Weight for SSIM-Loss. Typical value: 0.12 [S1].

### Edge Cases and Pitfalls
*   **Overfitting in Phase 1:** Training too many iterations on the initial $N_0$ images can cause the model to overfit to the sparse initial view, leading to poor generalization when new images arrive [S1]. The strict iteration limit (2,000) mitigates this [S1].
*   **Threshold Selection:** The distance threshold for identifying "new" points in the sparse cloud must be carefully chosen. Too low, and existing points are re-initialized, causing instability. Too high, and new geometry is missed [S1].
*   **Learning Rate Inconsistency:** If the learning rate is not adapted per-image, regions with fewer training iterations may have lower quality than well-trained regions [S1]. The adaptive learning rate formula addresses this by considering overlapping images [S1].
*   **Gaussian Accumulation:** Without the Load-Balancing-Loss, Gaussians may accumulate abnormally in specific regions, increasing memory usage and slowing down training [S1].
*   **Thread Contention:** Using per-pixel gradient accumulation can lead to serialized atomic operations on the GPU. The method uses splatting-based parallelism (per-Gaussian gradient accumulation) to avoid this [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** The method is designed for near real-time optimization. For each new image, it performs $T_I$ iterations (typically 200) on a subset of images [S1]. The complexity per image is roughly $O(T_I \cdot N_{active})$, where $N_{active}$ is the number of images involved in the local/semi-global optimization. The use of splatting-based parallelism reduces training time by eliminating thread contention [S1].
*   **Memory Complexity:** Memory usage scales with the number of Gaussians in the 3DGS field. The Load-Balancing-Loss helps control the total number of Gaussians by preventing abnormal accumulation [S1].

### Measured Performance
*   Evaluated on RGB datasets including 3DGS, Mip-NeRF 360, On-the-Fly SfM, and Replica [S1].
*   Compared against offline training solutions (Adr-Gaussian, Taming-3DGS) and SLAM-based methods (GS-SLAM, RTG-SLAM) [S1].
*   Hardware: Tested on RTX 4090 GPU [S1].
*   The method achieves faster training times compared to retraining from scratch while maintaining competitive PSNR and SSIM metrics [S1].

### Scaling
*   The average number of training iterations per image approaches $T_I$ (200) as the total number of images increases, making the per-image cost constant [S1].
*   The hierarchical weighting ensures that only a subset of images (top $N_m$ and their neighbors) are heavily optimized, keeping computational cost manageable for large scenes [S1].

### Strengths and Limitations
*   **Strengths:**
    *   Enables incremental, near real-time 3DGS optimization [S1].
    *   Hierarchical weighting and adaptive learning rates improve local quality consistency [S1].
    *   Load-balancing loss reduces Gaussian count and accelerates training [S1].
    *   Splatting-based parallelism improves GPU utilization [S1].
*   **Limitations:**
    *   Requires a robust On-the-Fly SfM module for pose estimation and sparse cloud generation [S1].
    *   Performance depends on the quality of the initial $N_0$ images and the SfM accuracy [S1].
    *   The method is specific to RGB input in the cited experiments (RTG-SLAM comparison was limited to RGB-D) [S1].

### When to Use
*   Use when processing a stream of images (e.g., from a moving camera or UAV) and real-time or near real-time reconstruction is required [S1].
*   Use when computational resources are limited for full retraining [S1].

### When Not to Use
*   Use offline training methods if all images are available and maximum possible quality is required without time constraints [S1].
*   Use if the input is not sequential or if SfM is unreliable [S1].

## Background

### Origin
The method was proposed in the paper "Gaussian On-the-Fly Splatting: A Progressive Framework for Robust Near Real-Time 3DGS Optimization" [S1].

### History and Development
The method builds upon the original 3D Gaussian Splatting (3DGS) rendering technique [S1]. It addresses the limitations of offline 3DGS training by introducing a progressive framework that integrates with On-the-Fly SfM pipelines [S1]. It incorporates strategies from Taming 3DGS for parallel gradient accumulation [S1].

### Where it is Used
*   Simulated sequential inputs from 3DGS, Mip-NeRF 360, On-the-Fly SfM, and Replica datasets [S1].
*   Applications requiring dynamic scene reconstruction, such as UAV mapping and VSLAM (Visual Simultaneous Localization and Mapping) [S1].

## Sources

[S1] Gaussian On-the-Fly Splatting: A Progressive Framework for Robust Near Real-Time 3DGS Optimization, https://doi.org/10.1109/lra.2025.3632729, fetched 2026-10-30
