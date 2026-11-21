# NeRF-RCNN


## Overview
NeRF-RCNN is a proposal-based 3D instance segmentation model that takes a pre-trained NeRF as input and outputs 3D Axis-Aligned Bounding Boxes (AABB), class labels, and discrete 3D masks of detected objects [S1]. It extends NeRF-RPN by appending a box prediction head and a mask prediction head to refine proposals into final instances [S1]. Unlike Semantic-NeRF, which does not produce instance-level information, NeRF-RCNN outputs discrete instance masks and bounding boxes [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A pre-trained Instance-NeRF model (providing radiance and density fields), Region of Interests (RoIs) provided by the NeRF-RPN stage, and ground truth data including 3D bounding boxes and 3D instance segmentation masks [S1].
*   **Outputs:** 3D AABBs, class probabilities, and discrete 3D masks for each detected object [S1].

**Data Structures:**
*   **Grid:** A uniform grid covering the full traceable volume of the pre-trained Instance-NeRF model, storing sampled appearance and density values [S1].
*   **RoI Features:** Extracted features for each proposal using 3D RoIAlign [S1].
*   **Box Offsets:** A vector $t_{i,k}$ of size 6 for each class $k$ and RoI $i$, representing translation and scale adjustments [S1].
*   **Masks:** Binary masks of size $m \times m \times m$ for each RoI [S1].

**Parameters:**
*   $m=5$: Resolution for the box head mask [S1].
*   $m=10$: Resolution for the mask head mask [S1].
*   $\lambda_1, \lambda_2$: Hyper-parameters weighting the regression and mask losses in the total loss function [S1].
*   NMS threshold for NeRF-RCNN: 0.15 [S1].
*   Score threshold for detection: 0.5 [S1].

**Steps:**
1.  **Sampling:** Uniformly sample appearance and density on a grid covering the full traceable volume of the pre-trained Instance-NeRF model [S1].
2.  **Proposal Generation:** Pass the sampled grid to NeRF-RPN to output proposals and a feature pyramid [S1].
3.  **Feature Extraction:** Use a backbone identical to NeRF-RPN to extract features from the sampled grid [S1].
4.  **RoIAlign:** Use 3D RoIAlign to obtain features within the RoI of each output proposal. This extends standard RoIAlign with an additional dimension to extract consistent features for each RoI on the feature map [S1].
5.  **Target Assignment:** For each RoI, set the ground truth box with the largest intersection over union (IoU) as its regression target [S1].
6.  **Box Head Processing:**
    *   Flatten the aligned RoI features.
    *   Pass through a fully connected layer for encoding [S1].
    *   Split into a box branch and a classification branch [S1].
    *   The box branch regresses a RoI to a more accurate bounding box per class [S1].
    *   The classification branch predicts classification scores [S1].
7.  **Mask Head Processing:**
    *   Feed aligned features into a 3D CNN (mask head) [S1].
    *   The mask head predicts $L$ binary masks of size $m \times m \times m$ for each RoI, where $L$ is the number of classes including background [S1].
    *   Apply a sigmoid function as the activation [S1].
8.  **Refinement (Post-Processing/Training Aid):** Apply 2D mask refinement using CascadePSP on the Instance-NeRF results to improve segmentation quality on object boundaries. Compose the refined masks into a single segmentation map to further optimize the instance field [S1].

**Formulas:**

*   **Box Offsets:** The box head output $t_{i,k}$ is defined as:
    $$ t_{i,k} = (tx_{i,k}, ty_{i,k}, tz_{i,k}, tw_{i,k}, tl_{i,k}, th_{i,k}) $$
    where:
    $$ tx_{i,k} = \frac{x_k - x_a}{w_a} $$
    $$ ty_{i,k} = \frac{y_k - y_a}{l_a} $$
    $$ tz_{i,k} = \frac{z_k - z_a}{h_a} $$
    $$ tw_{i,k} = \log\left(\frac{w_k}{w_a}\right) $$
    $$ tl_{i,k} = \log\left(\frac{l_k}{l_a}\right) $$
    $$ th_{i,k} = \log\left(\frac{h_k}{h_a}\right) $$
    Here, $x_k, y_k, z_k$ are the center coordinates of the predicted box, $w_k, l_k, h_k$ are the side lengths, and $x_a, y_a, z_a, w_a, l_a, h_a$ are the corresponding parameters of the RoI [S1].

*   **Classification Loss:** Multi-class cross-entropy loss (implemented as BCE in the source formula) for class prediction:
    $$ L_{cls} = \frac{1}{|N|} \sum_{i \in N} L_{BCE}(p_i, p^*_i) $$
    where $p_i$ is the predicted classification score vector after sigmoid, $p^*_i$ is the ground truth, and $N$ is the set of sampled RoIs [S1].

*   **Box Regression Loss:** Smooth L1 loss for bounding box regression:
    $$ L_{reg} = \frac{1}{|N_p|} \sum_{i \in N_p} \sum_{k=1}^L p_{i,k} L_{smooth}(t_{i,k}, t^*_i) $$
    where $N_p$ is the set of positive samples, $L$ is the number of classes including background, $p_{i,k}$ is the $k$-th dimension of $p_i$, $t_{i,k}$ is the box offsets of class $k$, and $t^*_i$ is the ground-truth target [S1]. Note that for an RoI associated with ground truth class $c$, only the $c$-th box regression loss contributes [S1].

*   **Mask Loss:** Binary cross-entropy loss for 3D mask prediction:
    $$ L_{mask} = \frac{\lambda}{|N_p|} \sum_{i \in N_p} \sum_{k=1}^L p_{i,k} L_p(m_{i,k}, m^*_i) $$
    where $m_{i,k}$ is the predicted mask of class $k$, $m^*_i$ is the ground truth mask, and $L_p$ is the loss function (BCE) [S1]. Only the mask BCE loss corresponding to the ground truth label contributes [S1].

*   **Total Loss:**
    $$ L = L_{cls} + \lambda_1 L_{reg} + \lambda_2 L_{mask} $$
    where $\lambda_1, \lambda_2$ are hyper-parameters [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def uniform_sample_grid(nerf, resolution):
    """Uniformly sample appearance and density on a grid covering the full traceable volume."""
    return nerf.sample_uniform(resolution)

def extract_backbone_features(grid):
    """Extract features using a backbone identical to NeRF-RPN."""
    return grid.backbone()

def roi_align_3d(feature_pyramid, roi):
    """Extend RoIAlign with an additional dimension to extract consistent features for each RoI."""
    return feature_pyramid[0].align_3d(roi)

def fully_connected(x, W, b):
    """Standard fully connected layer encoding."""
    return x @ W + b

def conv3d(x, channels, kernel_size):
    """3D CNN predicting L binary masks."""
    return x.conv3d(channels, kernel_size)

def sigmoid(x):
    """Apply sigmoid activation."""
    return 1.0 / (1.0 + math.exp(-x))

def decode_boxes(rois, offsets):
    """Decode box offsets to AABBs using RoI parameters."""
    boxes = []
    for roi, offset in zip(rois, offsets):
        xa, ya, za, wa, la, ha = roi.bbox_params
        tx, ty, tz, tw, tl, th = offset
        xk = tx * wa + xa
        yk = ty * la + ya
        zk = tz * ha + za
        wk = wa * math.exp(tw)
        lk = la * math.exp(tl)
        hk = ha * math.exp(th)
        boxes.append((xk, yk, zk, wk, lk, hk))
    return boxes

def assign_positive_targets(boxes, gt_boxes):
    """Set ground truth box with largest IoU as regression target."""
    pos_indices = []
    for i, box in enumerate(boxes):
        best_iou = 0.0
        for gt in gt_boxes:
            iou = compute_iou(box, gt)
            if iou > best_iou:
                best_iou = iou
        if best_iou > 0.5:
            pos_indices.append(i)
    return pos_indices

def binary_cross_entropy(pred, target):
    """Compute BCE loss."""
    return -mean(target * log(sigmoid(pred)) + (1 - target) * log(1 - sigmoid(pred)))

def smooth_l1_loss(pred, target):
    """Compute Smooth L1 loss."""
    diff = abs(pred - target)
    return mean(where(diff < 1, 0.5 * diff**2, diff - 0.5))

def nerf_rcnn_forward(pretrained_nerf, rpn_proposals, grid_resolution):
    # Step 1: Uniformly sample appearance and density on a grid covering the full traceable volume
    grid = uniform_sample_grid(pretrained_nerf, grid_resolution)
    
    # Step 2: Extract features using a backbone identical to NeRF-RPN
    feature_pyramid = extract_backbone_features(grid)
    
    # Step 3: Use 3D RoIAlign to obtain features within the RoI of each proposal
    roi_features = []
    for roi in rpn_proposals:
        features = roi_align_3d(feature_pyramid, roi)
        roi_features.append(features)
    roi_features = stack(roi_features)
    
    # Step 4: Box Head Processing
    flattened = flatten(roi_features)
    encoded = fully_connected(flattened, W_fc, b_fc)
    box_encoded = fully_connected(encoded, W_box, b_box)
    cls_encoded = fully_connected(encoded, W_cls, b_cls)
    
    # Box branch regresses offsets
    box_offsets = fully_connected(box_encoded, W_reg, b_reg)
    # Classification branch predicts scores
    class_logits = fully_connected(cls_encoded, W_cls_out, b_cls_out)
    
    # Step 5: Mask Head Processing
    # 3D CNN predicting L binary masks of size m x m x m
    masks = conv3d(roi_features, channels=L, kernel_size=m)
    masks = sigmoid(masks)
    
    # Step 6: Decode Box Offsets to AABBs
    boxes = decode_boxes(rpn_proposals, box_offsets)
    
    return boxes, class_logits, masks

def nerf_rcnn_loss(predictions, gt_boxes, gt_masks, gt_classes, lambda1, lambda2):
    boxes, class_logits, masks = predictions
    
    # Identify positive samples (RoIs with largest IoU with GT)
    pos_indices = assign_positive_targets(boxes, gt_boxes)
    
    # Classification Loss (BCE)
    class_loss = mean(binary_cross_entropy(class_logits, gt_classes))
    
    # Box Regression Loss (Smooth L1) for positive samples
    box_loss = mean(smooth_l1_loss(boxes[pos_indices], gt_boxes[pos_indices]))
    
    # Mask Loss (BCE) for positive samples and ground truth class
    mask_loss = mean(binary_cross_entropy(masks[pos_indices], gt_masks[pos_indices]))
    
    # Total Loss
    total_loss = class_loss + lambda1 * box_loss + lambda2 * mask_loss
    return total_loss
```

## Subtopics

### Panoptic NeRF
Panoptic NeRF specializes in urban environments for 3D-to-2D label propagation, a key task in extending urban autonomous driving datasets [S2]. The method uses two semantic fields: one learned by a semantic head and another determined by 3D bounding boxes [S2]. According to the authors, the rigid bounding box-based semantics force the model to learn the correct geometry, whereas the learned semantic head improves semantic understanding [S2].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Benchmarked on the NeRF-RPN dataset with 1015 scenes, multi-view images, camera poses, 3D bounding boxes, and 2D/3D ground truth instance segmentation [S1].
*   **Strengths:** It is one of the first methods to propose 3D instance segmentation for NeRF without utilizing ground-truth information at test time, generalizing to different kinds of scenes and requiring only a pre-trained NeRF during inference [S1].
*   **Limitations:** Instance label regularization can smooth out detailed structures like thin chair legs or lamp stands, degrading segmentation quality on object silhouettes [S1].
*   **When to use:** When 3D instance segmentation is required for a pre-trained NeRF scene [S1].
*   **When not to use:** When instance-level information is not needed (Semantic-NeRF is sufficient) or when ground-truth instance masks are not available for training [S1].

## Background
NeRF-RCNN was proposed in the paper "Instance Neural Radiance Field" [S1]. It is based on the architecture of Mask R-CNN [19] and extends NeRF-RPN [23] [S1]. The method was trained on a large NeRF segmentation dataset built on 3D-FRONT, a large-scale synthetic indoor scene dataset [S1]. The dataset includes 1015 scenes with multi-view images, camera poses, 3D bounding boxes, 2D ground truth instance segmentation, and 3D voxelized instance segmentation ground truth [S1].

## Sources
[S1] Instance Neural Radiance Field, https://doi.org/10.1109/ICCV51070.2023.00079, fetched 2026-10-29
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
