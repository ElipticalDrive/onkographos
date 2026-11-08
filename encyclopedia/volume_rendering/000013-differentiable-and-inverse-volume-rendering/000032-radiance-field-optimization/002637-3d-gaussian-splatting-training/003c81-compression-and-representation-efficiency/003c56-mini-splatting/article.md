# Mini-Splatting

Also called: Constrained Gaussian Representation, Hard-constrained 3DGS, Mixed Integer 3DGS

## Overview
Mini-Splatting is a resource-efficient variant of 3D Gaussian Splatting (3DGS) that explicitly constrains the total number of Gaussians to a hard limit $N_{max}$ to bound memory footprints and accelerate training and rendering [S1, unofficial][S2]. It addresses the "overlapping" and "under-reconstruction" phenomena of vanilla 3DGS by pivoting from direct simplification to spatial reorganization, leveraging both screen-space and world-space information to densify around objects rather than merely removing primitives [S2]. Unlike voxel pruning or direct opacity/scale pruning, Mini-Splatting maintains a constant Gaussian count by iteratively pruning the least important Gaussians and splitting the most important ones, effectively reorganizing their spatial distribution [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Initial set of 3D Gaussians (typically from a standard 3DGS initialization), camera poses, and input images.
*   **Output:** A set of $N_{max}$ 3D Gaussians with optimized positions, scales, rotations, opacities, and spherical harmonic (SH) coefficients.

**Data Structures:**
*   **Gaussian Set:** A collection of $N$ Gaussians. Each Gaussian $i$ is defined by:
    *   Center position $\mathbf{c}_i \in \mathbb{R}^3$.
    *   Scale $\mathbf{s}_i \in \mathbb{R}^3$ (or covariance matrix $\Sigma_i$).
    *   Rotation $\mathbf{q}_i \in \mathbb{R}^4$ (quaternion).
    *   Opacity $\alpha_i \in [0, 1]$.
    *   Spherical Harmonic coefficients $\mathbf{c}_{i, SH}$ for view-dependent color.
*   **Hard Limit:** A scalar integer $N_{max}$ representing the maximum allowed number of Gaussians.

**Algorithm Steps:**
The optimization follows a schedule of 30,000 steps, divided into densification and simplification phases [S2].

1.  **Initialization:** Initialize the Gaussian set with $N_{init}$ Gaussians (typically from standard 3DGS initialization). Set $N_{max}$ based on target resource constraints.
2.  **Optimization Loop (Iterate $t$ from 1 to 30,000):**
    *   **Rendering:** Render the current scene using the current Gaussian set.
    *   **Loss Calculation:** Compute the loss between rendered and ground-truth images.
    *   **Gradient Update:** Update Gaussian attributes (position, scale, rotation, opacity, SH) using gradient descent.
    *   **Phase 1: Densification (Steps 1–15,000):**
        *   **Disable View-Dependent Colors:** Do not update SH coefficients during this phase to save resources and because they barely enhance densification [S2].
        *   **Blur Split Densification:** Detect blurring areas in the rendered image. Split Gaussians located in these regions to increase local density [S2].
        *   **Depth Reinitialization Densification:** Use merged depth points to reinitialize the scene geometry and Gaussian positions [S2].
        *   **Split Most Important:** Split the most important Gaussians based on an importance score [S1, unofficial].
        *   **Prune Least Important:** Iteratively prune the least important Gaussians [S1, unofficial].
        *   **Constraint:** Keep the total Gaussian count constant at $N_{max}$ by balancing splits and prunes [S1, unofficial].
    *   **Phase 2: Simplification (Steps 15,000 and 20,000):**
        *   **Intersection Preserving Simplification:** Retain only Gaussians that have the maximal impact on the rendered image, discarding redundant overlaps [S2].
        *   **Gaussian Sampling Simplification:** Sample points to preserve both the underlying geometric structure and the original rendering quality [S2].
        *   **Stack Densification and Simplification:** Apply these passes to achieve a uniform spatial distribution of Gaussians around object surfaces [S2].
    *   **Phase 3: Refinement (Steps 15,001–30,000):**
        *   **Enable SH Coefficients:** After simplification at step 15,000, enable and increase the SH level to restore view-dependent colors [S2].
        *   **Continue Optimization:** Continue optimizing all attributes including SH coefficients until step 30,000 [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mini_splatting_optimization(gaussians, images, poses, N_max, total_steps=30000):
    """
    Optimizes 3D Gaussians with a hard constraint on the number of primitives.
    
    Args:
        gaussians: Initial set of 3D Gaussians.
        images: List of input images.
        poses: List of camera poses.
        N_max: Maximum number of Gaussians allowed.
        total_steps: Total number of optimization steps (default 30,000).
        
    Returns:
        Optimized set of N_max Gaussians.
    """
    # Initialize optimization state
    current_gaussians = gaussians
    current_count = len(current_gaussians)
    
    # Ensure initial count does not exceed N_max
    if current_count > N_max:
        current_gaussians = prune_least_important(current_gaussians, N_max)
        current_count = N_max

    for step in range(1, total_steps + 1):
        # Render current scene
        rendered_images = render(current_gaussians, images, poses)
        
        # Compute loss (e.g., L1 + SSIM)
        loss = compute_loss(rendered_images, images)
        
        # Backpropagate gradients to update Gaussian attributes
        gradients = compute_gradients(loss, current_gaussians)
        update_attributes(current_gaussians, gradients)
        
        # Phase 1: Densification (Steps 1 to 15,000)
        if step <= 15000:
            # Disable SH updates to save resources
            disable_sh_updates(current_gaussians)
            
            # Detect blurring areas in rendered images
            blur_regions = detect_blur(rendered_images)
            
            # Split Gaussians in blur regions
            split_candidates = select_gaussians_in_regions(current_gaussians, blur_regions)
            split_gaussians(current_gaussians, split_candidates)
            
            # Reinitialize positions using depth points
            depth_points = compute_merged_depth(rendered_images, poses)
            reinitialize_positions(current_gaussians, depth_points)
            
            # Split most important Gaussians
            important_candidates = select_most_important(current_gaussians)
            split_gaussians(current_gaussians, important_candidates)
            
            # Prune least important Gaussians to maintain N_max
            least_important = select_least_important(current_gaussians)
            prune_gaussians(current_gaussians, least_important)
            
            # Enforce hard constraint
            current_gaussians = enforce_count(current_gaussians, N_max)
            
        # Phase 2: Simplification (Steps 15,000 and 20,000)
        elif step in [15000, 20000]:
            # Intersection preserving simplification
            current_gaussians = intersection_preserving_simplify(current_gaussians)
            
            # Gaussian sampling simplification
            current_gaussians = gaussian_sampling_simplify(current_gaussians, N_max)
            
            # Enforce hard constraint
            current_gaussians = enforce_count(current_gaussians, N_max)
            
            # Enable SH coefficients and increase SH level
            enable_sh_updates(current_gaussians, level=2) # Example: increase SH level
            
        # Phase 3: Refinement (Steps 15,001 to 30,000, excluding simplification steps)
        else:
            # Continue optimizing all attributes including SH
            pass
            
    return current_gaussians

def enforce_count(gaussians, N_max):
    """
    Ensures the number of Gaussians does not exceed N_max.
    If count > N_max, prune the least important ones.
    If count < N_max, this function does nothing (splits/prunes balance during densification).
    """
    current_count = len(gaussians)
    if current_count > N_max:
        diff = current_count - N_max
        least_important = select_least_important(gaussians, k=diff)
        gaussians = remove_gaussians(gaussians, least_important)
    return gaussians

def select_least_important(gaussians, k=None):
    """
    Selects the k least important Gaussians based on an importance score.
    Importance score is typically derived from opacity and gradient magnitude.
    """
    scores = compute_importance_scores(gaussians)
    sorted_indices = argsort(scores) # Ascending order
    if k is None:
        return [gaussians[i] for i in sorted_indices]
    else:
        return [gaussians[i] for i in sorted_indices[:k]]

def select_most_important(gaussians, k=None):
    """
    Selects the k most important Gaussians for splitting.
    """
    scores = compute_importance_scores(gaussians)
    sorted_indices = argsort(scores, reverse=True) # Descending order
    if k is None:
        return [gaussians[i] for i in sorted_indices]
    else:
        return [gaussians[i] for i in sorted_indices[:k]]

def split_gaussians(gaussians, candidates):
    """
    Splits each candidate Gaussian into two new Gaussians.
    New positions are perturbed from the original center.
    Scales are reduced. Opacity is distributed.
    """
    new_gaussians = []
    for g in candidates:
        # Create two new Gaussians from the original
        g1 = copy(g)
        g2 = copy(g)
        
        # Perturb positions
        g1.center += perturbation()
        g2.center -= perturbation()
        
        # Reduce scales
        g1.scale *= 0.5
        g2.scale *= 0.5
        
        # Distribute opacity
        g1.opacity = g.opacity / 2
        g2.opacity = g.opacity / 2
        
        new_gaussians.append(g1)
        new_gaussians.append(g2)
        
    # Remove candidates from original list and add new ones
    for g in candidates:
        gaussians.remove(g)
    gaussians.extend(new_gaussians)

def prune_gaussians(gaussians, to_prune):
    """
    Removes the specified Gaussians from the set.
    """
    for g in to_prune:
        gaussians.remove(g)

def detect_blur(rendered_images):
    """
    Detects regions in the rendered images that are blurry.
    Blurriness is often associated with high gradient magnitude in the loss or low frequency content.
    """
    # Placeholder for blur detection logic
    # Typically involves analyzing the gradient of the rendered image or the error map
    blur_regions = []
    for img in rendered_images:
        # Compute gradient magnitude
        grad_mag = compute_gradient_magnitude(img)
        # Identify regions with high gradient magnitude (edges) or specific blur patterns
        # For simplicity, assume we identify regions where the error is high and structure is low
        blur_regions.append(find_blur_regions(grad_mag))
    return blur_regions

def compute_merged_depth(rendered_images, poses):
    """
    Computes merged depth points from the rendered images and camera poses.
    """
    depth_points = []
    for img, pose in zip(rendered_images, poses):
        # Render depth buffer
        depth_buffer = render_depth(img, pose)
        # Convert depth buffer to 3D points
        points = depth_buffer_to_3d(depth_buffer, pose)
        depth_points.extend(points)
    return depth_points

def reinitialize_positions(gaussians, depth_points):
    """
    Reinitializes the positions of some Gaussians to match the depth points.
    """
    # Match Gaussians to nearest depth points and update positions
    for g in gaussians:
        nearest_point = find_nearest_point(g.center, depth_points)
        if distance(g.center, nearest_point) > threshold:
            g.center = nearest_point

def intersection_preserving_simplify(gaussians):
    """
    Retains only Gaussians that have the maximal impact on the rendered image.
    Discards redundant overlaps.
    """
    # Compute impact score for each Gaussian
    impact_scores = compute_impact_scores(gaussians)
    # Sort by impact score descending
    sorted_gaussians = sort_by_score(gaussians, impact_scores, reverse=True)
    # Keep top K Gaussians (K determined by N_max or a threshold)
    simplified_gaussians = sorted_gaussians[:N_max]
    return simplified_gaussians

def gaussian_sampling_simplify(gaussians, N_max):
    """
    Samples points to preserve geometric structure and rendering quality.
    """
    # Sample N_max Gaussians from the current set based on importance
    sampled_gaussians = sample_gaussians(gaussians, N_max)
    return sampled_gaussians

def sample_gaussians(gaussians, N_max):
    """
    Samples N_max Gaussians from the input set.
    """
    scores = compute_importance_scores(gaussians)
    # Use weighted sampling based on scores
    sampled_indices = weighted_sample(scores, N_max)
    return [gaussians[i] for i in sampled_indices]

def compute_importance_scores(gaussians):
    """
    Computes an importance score for each Gaussian.
    Score is typically a function of opacity and gradient magnitude.
    """
    scores = []
    for g in gaussians:
        # Example: score = opacity * gradient_magnitude
        score = g.opacity * compute_gradient_magnitude(g)
        scores.append(score)
    return scores

def compute_gradient_magnitude(g):
    """
    Computes the gradient magnitude for a Gaussian.
    """
    # Placeholder for gradient computation
    return 1.0 # Placeholder value

def find_blur_regions(grad_mag):
    """
    Finds regions in the gradient magnitude map that are blurry.
    """
    # Placeholder for blur region detection
    return []

def find_nearest_point(center, points):
    """
    Finds the nearest point to the center in the list of points.
    """
    min_dist = infinity
    nearest = None
    for p in points:
        dist = distance(center, p)
        if dist < min_dist:
            min_dist = dist
            nearest = p
    return nearest

def distance(p1, p2):
    """
    Computes the Euclidean distance between two points.
    """
    return sqrt(sum((p1[i] - p2[i])**2 for i in range(3)))

def perturbation():
    """
    Generates a small random perturbation vector.
    """
    return [random.gauss(0, 0.01) for _ in range(3)]

def copy(g):
    """
    Creates a deep copy of a Gaussian.
    """
    return Gaussian(center=g.center.copy(), scale=g.scale.copy(), rotation=g.rotation.copy(), 
                    opacity=g.opacity, sh_coeffs=g.sh_coeffs.copy())

def remove_gaussians(gaussians, to_remove):
    """
    Removes the specified Gaussians from the list.
    """
    return [g for g in gaussians if g not in to_remove]

def sort_by_score(gaussians, scores, reverse=False):
    """
    Sorts Gaussians based on their scores.
    """
    paired = list(zip(gaussians, scores))
    paired.sort(key=lambda x: x[1], reverse=reverse)
    return [g for g, s in paired]

def weighted_sample(scores, N_max):
    """
    Samples N_max indices from the list based on the scores.
    """
    # Normalize scores to probabilities
    total = sum(scores)
    probs = [s / total for s in scores]
    # Use numpy or similar for weighted sampling
    indices = np.random.choice(len(scores), size=N_max, replace=False, p=probs)
    return indices

def enable_sh_updates(gaussians, level=2):
    """
    Enables SH coefficient updates and sets the SH level.
    """
    for g in gaussians:
        g.sh_level = level
        g.sh_coeffs = initialize_sh_coeffs(level)

def initialize_sh_coeffs(level):
    """
    Initializes SH coefficients for a given level.
    """
    # Number of coefficients for level L is (L+1)^2
    num_coeffs = (level + 1)**2
    return np.zeros(num_coeffs * 3) # 3 for RGB

def disable_sh_updates(gaussians):
    """
    Disables SH coefficient updates.
    """
    for g in gaussians:
        g.sh_level = 0
        g.sh_coeffs = np.zeros(3) # Only base color

def update_attributes(gaussians, gradients):
    """
    Updates Gaussian attributes using gradients.
    """
    for g, grad in zip(gaussians, gradients):
        g.center -= learning_rate * grad.center
        g.scale -= learning_rate * grad.scale
        g.rotation -= learning_rate * grad.rotation
        g.opacity -= learning_rate * grad.opacity
        if g.sh_level > 0:
            g.sh_coeffs -= learning_rate * grad.sh_coeffs

def render(gaussians, images, poses):
    """
    Renders the scene using the given Gaussians and camera poses.
    """
    rendered_images = []
    for pose in poses:
        # Project Gaussians to 2D
        projected_gaussians = project_gaussians(gaussians, pose)
        # Rasterize projected Gaussians
        rendered_image = rasterize(projected_gaussians)
        rendered_images.append(rendered_image)
    return rendered_images

def project_gaussians(gaussians, pose):
    """
    Projects 3D Gaussians to 2D screen space.
    """
    projected = []
    for g in gaussians:
        # Transform center to camera space
        center_cam = pose.inv() * g.center
        # Project to 2D
        center_2d = project_point(center_cam, pose.focal_length, pose.principal_point)
        # Compute 2D covariance
        cov_2d = compute_2d_covariance(g.covariance, pose)
        projected.append(Gaussian2D(center=center_2d, covariance=cov_2d, opacity=g.opacity, 
                                    scale=g.scale, rotation=g.rotation, sh_coeffs=g.sh_coeffs))
    return projected

def project_point(point, focal_length, principal_point):
    """
    Projects a 3D point to 2D screen coordinates.
    """
    x = point.x * focal_length / point.z + principal_point.x
    y = point.y * focal_length / point.z + principal_point.y
    return Point2D(x, y)

def compute_2d_covariance(cov_3d, pose):
    """
    Computes the 2D covariance matrix from the 3D covariance.
    """
    # Jacobian of the projection
    J = compute_projection_jacobian(pose)
    cov_2d = J * cov_3d * J.T
    return cov_2d

def compute_projection_jacobian(pose):
    """
    Computes the Jacobian of the projection function.
    """
    # Placeholder for Jacobian computation
    return np.eye(2)

def rasterize(projected_gaussians):
    """
    Rasterizes the projected Gaussians to an image.
    """
    # Sort Gaussians by depth
    projected_gaussians.sort(key=lambda g: g.center.z)
    # Initialize image buffer
    image = np.zeros((height, width, 3))
    alpha_buffer = np.zeros((height, width))
    
    for g in projected_gaussians:
        # Render Gaussian to image
        render_gaussian(image, alpha_buffer, g)
        
    return image

def render_gaussian(image, alpha_buffer, g):
    """
    Renders a single Gaussian to the image.
    """
    # Compute bounding box
    bbox = compute_bounding_box(g.center_2d, g.covariance)
    # Iterate over pixels in bounding box
    for y in range(bbox.y_min, bbox.y_max):
        for x in range(bbox.x_min, bbox.x_max):
            # Compute distance from pixel center to Gaussian center
            dist = distance(Point2D(x, y), g.center_2d)
            # Compute Gaussian value
            value = compute_gaussian_value(dist, g.covariance)
            # Update image and alpha buffer
            alpha = value * g.opacity
            new_alpha = alpha_buffer[y, x] + alpha * (1 - alpha_buffer[y, x])
            if new_alpha > alpha_buffer[y, x]:
                image[y, x] = image[y, x] * (alpha_buffer[y, x] / new_alpha) + g.color * (alpha * (1 - alpha_buffer[y, x]) / new_alpha)
                alpha_buffer[y, x] = new_alpha

def compute_gaussian_value(dist, covariance):
    """
    Computes the value of a 2D Gaussian at a given distance.
    """
    # Inverse covariance
    inv_cov = np.linalg.inv(covariance)
    # Mahalanobis distance
    mah_dist = dist.T @ inv_cov @ dist
    # Gaussian value
    value = exp(-0.5 * mah_dist)
    return value

def compute_bounding_box(center, covariance):
    """
    Computes the bounding box of a 2D Gaussian.
    """
    # Eigenvalues and eigenvectors of covariance
    eigenvalues, eigenvectors = np.linalg.eig(covariance)
    # Compute bounding box size based on eigenvalues
    width = 2 * sqrt(eigenvalues[0]) * 3 # 3 sigma
    height = 2 * sqrt(eigenvalues[1]) * 3 # 3 sigma
    # Compute bounding box coordinates
    x_min = center.x - width / 2
    x_max = center.x + width / 2
    y_min = center.y - height / 2
    y_max = center.y + height / 2
    return BoundingBox(x_min, x_max, y_min, y_max)

def compute_impact_scores(gaussians):
    """
    Computes the impact score for each Gaussian.
    Impact score is typically the contribution to the rendered image.
    """
    scores = []
    for g in gaussians:
        # Render the scene with and without the Gaussian
        # The difference in the rendered image is the impact
        rendered_with = render([g], images, poses)
        rendered_without = render([], images, poses)
        impact = compute_difference(rendered_with, rendered_without)
        scores.append(impact)
    return scores

def compute_difference(img1, img2):
    """
    Computes the difference between two images.
    """
    return np.sum(np.abs(img1 - img2))

def compute_loss(rendered_images, images):
    """
    Computes the loss between rendered and ground-truth images.
    """
    loss = 0
    for rendered, ground_truth in zip(rendered_images, images):
        # L1 loss
        l1_loss = np.mean(np.abs(rendered - ground_truth))
        # SSIM loss
        ssim_loss = 1 - compute_ssim(rendered, ground_truth)
        loss += l1_loss + ssim_loss
    return loss

def compute_ssim(img1, img2):
    """
    Computes the Structural Similarity Index (SSIM) between two images.
    """
    # Placeholder for SSIM computation
    return 1.0

def compute_gradient_magnitude(g):
    """
    Computes the gradient magnitude for a Gaussian.
    """
    # Placeholder for gradient computation
    return 1.0

def find_blur_regions(grad_mag):
    """
    Finds regions in the gradient magnitude map that are blurry.
    """
    # Placeholder for blur region detection
    return []

def find_nearest_point(center, points):
    """
    Finds the nearest point to the center in the list of points.
    """
    min_dist = infinity
    nearest = None
    for p in points:
        dist = distance(center, p)
        if dist < min_dist:
            min_dist = dist
            nearest = p
    return nearest

def distance(p1, p2):
    """
    Computes the Euclidean distance between two points.
    """
    return sqrt(sum((p1[i] - p2[i])**2 for i in range(3)))

def perturbation():
    """
    Generates a small random perturbation vector.
    """
    return [random.gauss(0, 0.01) for _ in range(3)]

def copy(g):
    """
    Creates a deep copy of a Gaussian.
    """
    return Gaussian(center=g.center.copy(), scale=g.scale.copy(), rotation=g.rotation.copy(), 
                    opacity=g.opacity, sh_coeffs=g.sh_coeffs.copy())

def remove_gaussians(gaussians, to_remove):
    """
    Removes the specified Gaussians from the list.
    """
    return [g for g in gaussians if g not in to_remove]

def sort_by_score(gaussians, scores, reverse=False):
    """
    Sorts Gaussians based on their scores.
    """
    paired = list(zip(gaussians, scores))
    paired.sort(key=lambda x: x[1], reverse=reverse)
    return [g for g, s in paired]

def weighted_sample(scores, N_max):
    """
    Samples N_max indices from the list based on the scores.
    """
    # Normalize scores to probabilities
    total = sum(scores)
    probs = [s / total for s in scores]
    # Use numpy or similar for weighted sampling
    indices = np.random.choice(len(scores), size=N_max, replace=False, p=probs)
    return indices

def enable_sh_updates(gaussians, level=2):
    """
    Enables SH coefficient updates and sets the SH level.
    """
    for g in gaussians:
        g.sh_level = level
        g.sh_coeffs = initialize_sh_coeffs(level)

def initialize_sh_coeffs(level):
    """
    Initializes SH coefficients for a given level.
    """
    # Number of coefficients for level L is (L+1)^2
    num_coeffs = (level + 1)**2
    return np.zeros(num_coeffs * 3) # 3 for RGB

def disable_sh_updates(gaussians):
    """
    Disables SH coefficient updates.
    """
    for g in gaussians:
        g.sh_level = 0
        g.sh_coeffs = np.zeros(3) # Only base color

def update_attributes(gaussians, gradients):
    """
    Updates Gaussian attributes using gradients.
    """
    for g, grad in zip(gaussians, gradients):
        g.center -= learning_rate * grad.center
        g.scale -= learning_rate * grad.scale
        g.rotation -= learning_rate * grad.rotation
        g.opacity -= learning_rate * grad.opacity
        if g.sh_level > 0:
            g.sh_coeffs -= learning_rate * grad.sh_coeffs

def render(gaussians, images, poses):
    """
    Renders the scene using the given Gaussians and camera poses.
    """
    rendered_images = []
    for pose in poses:
        # Project Gaussians to 2D
        projected_gaussians = project_gaussians(gaussians, pose)
        # Rasterize projected Gaussians
        rendered_image = rasterize(projected_gaussians)
        rendered_images.append(rendered_image)
    return rendered_images

def project_gaussians(gaussians, pose):
    """
    Projects 3D Gaussians to 2D screen space.
    """
    projected = []
    for g in gaussians:
        # Transform center to camera space
        center_cam = pose.inv() * g.center
        # Project to 2D
        center_2d = project_point(center_cam, pose.focal_length, pose.principal_point)
        # Compute 2D covariance
        cov_2d = compute_2d_covariance(g.covariance, pose)
        projected.append(Gaussian2D(center=center_2d, covariance=cov_2d, opacity=g.opacity, 
                                    scale=g.scale, rotation=g.rotation, sh_coeffs=g.sh_coeffs))
    return projected

def project_point(point, focal_length, principal_point):
    """
    Projects a 3D point to 2D screen coordinates.
    """
    x = point.x * focal_length / point.z + principal_point.x
    y = point.y * focal_length / point.z + principal_point.y
    return Point2D(x, y)

def compute_2d_covariance(cov_3d, pose):
    """
    Computes the 2D covariance matrix from the 3D covariance.
    """
    # Jacobian of the projection
    J = compute_projection_jacobian(pose)
    cov_2d = J * cov_3d * J.T
    return cov_2d

def compute_projection_jacobian(pose):
    """
    Computes the Jacobian of the projection function.
    """
    # Placeholder for Jacobian computation
    return np.eye(2)

def rasterize(projected_gaussians):
    """
    Rasterizes the projected Gaussians to an image.
    """
    # Sort Gaussians by depth
    projected_gaussians.sort(key=lambda g: g.center.z)
    # Initialize image buffer
    image = np.zeros((height, width, 3))
    alpha_buffer = np.zeros((height, width))
    
    for g in projected_gaussians:
        # Render Gaussian to image
        render_gaussian(image, alpha_buffer, g)
        
    return image

def render_gaussian(image, alpha_buffer, g):
    """
    Renders a single Gaussian to the image.
    """
    # Compute bounding box
    bbox = compute_bounding_box(g.center_2d, g.covariance)
    # Iterate over pixels in bounding box
    for y in range(bbox.y_min, bbox.y_max):
        for x in range(bbox.x_min, bbox.x_max):
            # Compute distance from pixel center to Gaussian center
            dist = distance(Point2D(x, y), g.center_2d)
            # Compute Gaussian value
            value = compute_gaussian_value(dist, g.covariance)
            # Update image and alpha buffer
            alpha = value * g.opacity
            new_alpha = alpha_buffer[y, x] + alpha * (1 - alpha_buffer[y, x])
            if new_alpha > alpha_buffer[y, x]:
                image[y, x] = image[y, x] * (alpha_buffer[y, x] / new_alpha) + g.color * (alpha * (1 - alpha_buffer[y, x]) / new_alpha)
                alpha_buffer[y, x] = new_alpha

def compute_gaussian_value(dist, covariance):
    """
    Computes the value of a 2D Gaussian at a given distance.
    """
    # Inverse covariance
    inv_cov = np.linalg.inv(covariance)
    # Mahalanobis distance
    mah_dist = dist.T @ inv_cov @ dist
    # Gaussian value
    value = exp(-0.5 * mah_dist)
    return value

def compute_bounding_box(center, covariance):
    """
    Computes the bounding box of a 2D Gaussian.
    """
    # Eigenvalues and eigenvectors of covariance
    eigenvalues, eigenvectors = np.linalg.eig(covariance)
    # Compute bounding box size based on eigenvalues
    width = 2 * sqrt(eigenvalues[0]) * 3 # 3 sigma
    height = 2 * sqrt(eigenvalues[1]) * 3 # 3 sigma
    # Compute bounding box coordinates
    x_min = center.x - width / 2
    x_max = center.x + width / 2
    y_min = center.y - height / 2
    y_max = center.y + height / 2
    return BoundingBox(x_min, x_max, y_min, y_max)

def compute_impact_scores(gaussians):
    """
    Computes the impact score for each Gaussian.
    Impact score is typically the contribution to the rendered image.
    """
    scores = []
    for g in gaussians:
        # Render the scene with and without the Gaussian
        # The difference in the rendered image is the impact
        rendered_with = render([g], images, poses)
        rendered_without = render([], images, poses)
        impact = compute_difference(rendered_with, rendered_without)
        scores.append(impact)
    return scores

def compute_difference(img1, img2):
    """
    Computes the difference between two images.
    """
    return np.sum(np.abs(img1 - img2))

def compute_loss(rendered_images, images):
    """
    Computes the loss between rendered and ground-truth images.
    """
    loss = 0
    for rendered, ground_truth in zip(rendered_images, images):
        # L1 loss
        l1_loss = np.mean(np.abs(rendered - ground_truth))
        # SSIM loss
        ssim_loss = 1 - compute_ssim(rendered, ground_truth)
        loss += l1_loss + ssim_loss
    return loss

def compute_ssim(img1, img2):
    """
    Computes the Structural Similarity Index (SSIM) between two images.
    """
    # Placeholder for SSIM computation
    return 1.0
```

**Parameters:**
*   **Total Optimization Steps:** 30,000 [S2].
*   **Densification End Step:** 15,000 [S2].
*   **Simplification Steps:** 15,000 and 20,000 [S2].
*   **SH Level:** Disabled during densification (steps 1–15,000), enabled and increased after simplification (step 15,000) [S2].
*   **Hard Limit ($N_{max}$):** Not given in the sources. Determined by target resource constraints.

**Edge Cases and Pitfalls:**
*   **View-Dependent Colors:** View-dependent colors (SH coefficients) barely enhance densification and should be disabled during the densification phase to save resources [S2].
*   **Precision:** Not given in the sources.
*   **Boundaries:** The hard limit $N_{max}$ must be enforced strictly to bound memory footprints [S1, unofficial].
*   **Degenerate Input:** Not given in the sources.

## Subtopics

### [Model Compression Pipeline for 3D Gaussian Splatting](9987-model-compression-pipeline-for-3d-gaussian-splatting.md)
A compression pipeline combining Gaussian pruning, fine-tuning, SH distillation, Vector Quantization (VQ) for SH coefficients and colors, and FP16 quantization, designed to reduce computational complexity and memory usage for hardware-oriented 3DGS deployment [Parent].

### Mini-Splatting-D
A quality-prioritized variant of Mini-Splatting that omits the simplification process to maximize rendering quality [S2]. On Mip-NeRF360 (Outdoor), Mini-Splatting-D uses 4.69M Gaussians, achieving SSIM 0.831, PSNR 27.51, LPIPS 0.176. It surpasses Zip-NeRF in SSIM (0.828) and LPIPS (0.189) on this dataset [S2]. Training time is 31m48s, Peak GPU Memory is 7.45GB, Rendering FPS is 83, Peak Rendering Memory is 3.12GB [S2].

### Mini-Splatting-C
A storage-compression variant designed for scenarios requiring storage compression. It is based on Mini-Splatting and stores Gaussian centers as float32, while other attributes (scales, SH coefficients) are processed using transform coding and standard zip compression [S2].

## Operating characteristics
**Time and Memory Complexity:**
*   **Training Time:** Mini-Splatting reduces training time compared to vanilla 3DGS. On Mip-NeRF360 (Outdoor), Mini-Splatting training time is 17m56s vs 30m8s for 3DGS [S2]. Mini-Splatting-D training time is 31m48s [S2].
*   **Peak GPU Memory:** Mini-Splatting significantly reduces peak GPU memory. On Mip-NeRF360 (Outdoor), Mini-Splatting peak GPU memory is 2.61GB vs 7.45GB for 3DGS [S2]. Mini-Splatting-D peak GPU memory is 7.45GB [S2].
*   **Rendering FPS:** Mini-Splatting increases rendering FPS. On Mip-NeRF360 (Outdoor), Mini-Splatting rendering FPS is 410 vs 98 for 3DGS [S2]. Mini-Splatting-D rendering FPS is 83 [S2].
*   **Peak Rendering Memory:** Mini-Splatting reduces peak rendering memory. On Mip-NeRF360 (Outdoor), Mini-Splatting peak rendering memory is 0.40GB vs 2.79GB for 3DGS [S2]. Mini-Splatting-D peak rendering memory is 3.12GB [S2].

**Measured Performance:**
*   **Mip-NeRF360 (Outdoor):** Mini-Splatting uses 0.49M Gaussians vs 3.36M for 3DGS (7x fewer), achieving SSIM 0.822 vs 0.815, PSNR 27.34 vs 27.21 [S2].
*   **MipNeRF360 (General):** Mini-Splatting achieves SSIM 0.763, PSNR 25.83, LPIPS 0.273, and a size of 18.2 MB [S3].

**Scaling:**
*   Mini-Splatting scales well to larger scenes by maintaining a constant number of Gaussians, reducing memory usage and increasing rendering speed [S2].

**Strengths and Limitations:**
*   **Strengths:** Significant reduction in memory usage and training time, increased rendering speed, comparable or better rendering quality than vanilla 3DGS [S2].
*   **Limitations:** Requires careful tuning of the hard limit $N_{max}$ and optimization schedule [S2]. Simplification process may introduce artifacts if not properly configured [S2].

**When to Use:**
*   Use Mini-Splatting when resource constraints (memory, training time) are a priority and a slight trade-off in rendering quality is acceptable [S2].
*   Use Mini-Splatting-D when rendering quality is the primary concern and resources are less constrained [S2].
*   Use Mini-Splatting-C when storage compression is the primary goal [S2].

**When Not to Use:**
*   Do not use Mini-Splatting if maximum possible rendering quality is required and resources are abundant [S2].
*   Do not use Mini-Splatting if the scene requires a very large number of Gaussians to represent fine details, as the hard limit may restrict this [S2].

## Background
Mini-Splatting was introduced in the paper "Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians" [S2]. It addresses the inefficiency of vanilla 3DGS by reorganizing the spatial distribution of Gaussians, moving away from direct simplification to spatial reorganization [S2]. The method formulates the optimization as a mixed integer problem to bound memory footprints [S1, unofficial]. It has been cited in surveys on 3D Gaussian Splatting rasterization [S1, unofficial] and recent advances in 3D Gaussian splatting [S3].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
[S3] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
