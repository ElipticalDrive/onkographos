# 3D Instance Segmentation with Contrastive Learning

Also called: Contrastive 3DGS Segmentation

## Overview
This method segments 3D Gaussian primitives into distinct object instances by training per-Gaussian semantic features using contrastive learning on 2D masks derived from text queries [S1]. It solves the problem of assigning semantic identity to Gaussian primitives, which do not strictly model object geometry and often include low-opacity primitives or large primitives shared among objects [S1]. It differs from sibling methods by relying on interactive 2D mask generation via GroundingDINO and SAM-HQ, followed by contrastive feature optimization and space-carving post-processing, rather than language-guided training or generalizable encoders [S1, 12438-langsplat-training-procedure.md, 16774-generalizable-semantic-gaussian-splatting.md].

## How it works
**Inputs:**
- A set of training images and camera poses.
- A text query for each instance to be segmented (e.g., "pillar") [S1].
- Pre-trained 3D Gaussian Splatting parameters $\Theta_i = \{x_i, q_i, s_i, \alpha_i, c_i\}$ [S1, 15902-feature-3d-gaussian-splatting.md].

**Outputs:**
- A set of segmented instances $I_1, \dots, I_M$, where each instance is a subset of Gaussian primitives [S1].
- 2D masks $\{M_{q,j}\}$ for each instance $q$ in each training view $j$ [S1].

**Data Structures:**
- Per-Gaussian trainable feature vector $f_i \in \mathbb{R}^d$ (dimension $d$ not specified in sources) [S1].
- 2D masks $M_{q,j}$: Binary masks for instance $q$ in view $j$ [S1].

**Parameters:**
- Push loss margin: 0.3 [S1].
- Hard case threshold: 0.5 [S1].
- Feature similarity threshold $\tau$: 0.1 [S1].
- Pixel sampling counts: $M_u = 4096$ (pixels per image), $M_s = 4096$ (pixels per mask) [S1].
- Regularization weights: $\lambda_{\text{opacity}}$, $\lambda_{\text{scale}}$ (values not given in sources) [S1].

**Steps:**

1. **Generate 2D Masks:**
   For each training view and text query, use GroundingDINO to output bounding boxes [S1]. Feed these boxes to SAM-HQ to output 2D masks per instance [S1]. Define the "background" mask as the complement of the union of all instance masks [S1].

2. **Regularize 3DGS Training:**
   During the initial 3DGS optimization, add L1 regularization to opacity $\mathbf{o}_i$ and scaling $\mathbf{s}_i$ to discourage low-opacity primitives under surfaces and large primitives shared among objects [S1]:
   $$ \mathcal{L}_{\text{reg}} = \lambda_{\text{opacity}} \frac{1}{N} \sum_{i=1}^N \mathbf{o}_i + \lambda_{\text{scale}} \frac{1}{N} \sum_{i=1}^N \mathbf{s}_i $$
   where $N$ is the number of Gaussian primitives [S1].

3. **Train Per-Gaussian Features:**
   After 3DGS training, initialize per-Gaussian features $f_i$ [S1]. Rasterize these features into per-pixel features $\hat{f}_p$ using standard alpha blending [S1]. Sample pixel pairs to compute contrastive loss [S1]:
   - Sample $M_u$ pixels uniformly in the image and $M_s$ pixels per mask [S1].
   - If two pixels belong to the same 2D mask, compute the pull loss (cosine distance) to pull their features together [S1].
   - If two pixels belong to different masks, compute the push loss (cosine distance) with a margin of 0.3 to push their features apart [S1].
   - Add extra loss terms for "hard cases" where the distance between features of different masks is larger than 0.5 [S1].
   - Weight the pull loss by the ratio of negative-to-positive pairs to counter imbalance [S1].
   - Include a regularization term to encourage rendered features $\hat{f}$ to be unit norm [S1].

4. **Interactive Segmentation:**
   - User clicks inside the mask of an instance in any training view [S1].
   - Select the rendered feature $f_q$ corresponding to the clicked pixel as the query feature [S1].
   - Compute 2D masks $\{M_{q,j}\}$ for instance $q$ in each training view $j$ by comparing rendered features with $f_q$ [S1].
   - Segment instance $I_q$ by selecting all Gaussians with features $f_i$ such that $d(f_i, f_q) \leq \tau$, where $d$ is cosine distance and $\tau = 0.1$ [S1].

5. **Post-processing (Space Carving):**
   Remove residual Gaussians from the background reconstruction [S1]. For each Gaussian, check if at least one of the following conditions is met in *every* training image:
   1. Center projects inside instance 2D masks $\{M_{q,j}\}$.
   2. Center projects outside image boundary.
   3. Center is not visible in current camera (e.g., behind camera) [S1].
   If met for all views, remove the Gaussian from the background [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_2d_masks(training_views, text_query):
    masks = {}
    for view in training_views:
        boxes = GroundingDINO(view.image, text_query)
        instance_masks = SAM_HQ(view.image, boxes)
        background_mask = ~union(instance_masks)
        masks[view.id] = {'instances': instance_masks, 'background': background_mask}
    return masks

def train_contrastive_features(gaussians, training_views, masks, M_u=4096, M_s=4096):
    # Initialize features
    for g in gaussians:
        g.feature = random_unit_vector()
    
    for epoch in range(num_epochs):
        for view in training_views:
            # Sample pixels
            sampled_pixels = sample_pixels(view.image, M_u, M_s, masks[view.id])
            
            # Rasterize features for sampled pixels
            pixel_features = {}
            for p in sampled_pixels:
                pixel_features[p] = rasterize_feature(gaussians, p, view.camera)
            
            # Compute contrastive loss
            loss = 0
            positive_pairs = []
            negative_pairs = []
            
            for p1, p2 in pair_pixels(sampled_pixels, masks[view.id]):
                f1 = pixel_features[p1]
                f2 = pixel_features[p2]
                dist = cosine_distance(f1, f2)
                
                if same_mask(p1, p2, masks[view.id]):
                    positive_pairs.append(dist)
                else:
                    negative_pairs.append(dist)
                    if dist > 0.5: # Hard case
                        loss += hard_case_loss(dist)
            
            # Pull loss (weighted by negative/positive ratio)
            if positive_pairs:
                pull_loss = mean(positive_pairs)
                weight = len(negative_pairs) / len(positive_pairs)
                loss += weight * pull_loss
            
            # Push loss with margin
            if negative_pairs:
                push_loss = margin_loss(negative_pairs, margin=0.3)
                loss += push_loss
            
            # Feature normalization regularization
            norm_loss = mean([norm(f) - 1.0 for f in pixel_features.values()])
            loss += norm_loss
            
            # Update features
            update_gaussian_features(gaussians, loss)

def segment_instance(query_pixel, gaussians, training_views, masks, tau=0.1):
    # Get query feature from clicked pixel
    query_feature = rasterize_feature(gaussians, query_pixel, query_pixel.camera)
    
    # Segment Gaussians
    instance_gaussians = []
    for g in gaussians:
        dist = cosine_distance(g.feature, query_feature)
        if dist <= tau:
            instance_gaussians.append(g)
    
    # Generate 2D masks for this instance in all views
    instance_masks = {}
    for view in training_views:
        view_masks = []
        for p in view.image_pixels:
            f = rasterize_feature(gaussians, p, view.camera)
            if cosine_distance(f, query_feature) <= tau:
                view_masks.append(p)
        instance_masks[view.id] = view_masks
    
    return instance_gaussians, instance_masks

def space_carving(gaussians, instance_masks, training_views):
    gaussians_to_remove = []
    for g in gaussians:
        remove = True
        for view in training_views:
            projected = project(g.center, view.camera)
            inside_instance = False
            for inst_masks in instance_masks.values():
                if projected in inst_masks:
                    inside_instance = True
                    break
            
            outside_boundary = not is_inside_image(projected)
            not_visible = is_behind_camera(g.center, view.camera)
            
            if not (inside_instance or outside_boundary or not_visible):
                remove = False
                break
        
        if remove:
            gaussians_to_remove.append(g)
    
    for g in gaussians_to_remove:
        remove_from_scene(g)
```

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Interactive segmentation requires only one click per instance [S1]. 2D masks aggregate information from all views, resulting in better masks than SAM-HQ alone [S1]. Regularization and post-processing improve visual quality [S1].
- **Limitations:** Low-opacity Gaussians under surfaces may not receive enough gradients [S1]. Large Gaussians shared among objects may get conflicting gradients [S1]. Residual Gaussians may remain, requiring space carving [S1].
- **When to use:** When interactive segmentation of specific instances in a 3DGS scene is required, and text queries can be provided [S1].
- **When not to use:** When automatic, fully unsupervised segmentation is needed without user interaction [S1].

## Background
The method is described in "Splat and Replace: 3D Reconstruction with Repetitive Elements" [S1]. It builds on previous work [Bhalgat et al. 2023; Fan et al. 2023] and uses contrastive learning [Radford et al. 2021]. It addresses challenges in applying contrastive learning to 3DGS, such as low-opacity primitives and shared large primitives [S1].

## Sources
[S1] Splat and Replace: 3D Reconstruction with Repetitive Elements, https://doi.org/10.1145/3721238.3730727, fetched 2026-10-26
