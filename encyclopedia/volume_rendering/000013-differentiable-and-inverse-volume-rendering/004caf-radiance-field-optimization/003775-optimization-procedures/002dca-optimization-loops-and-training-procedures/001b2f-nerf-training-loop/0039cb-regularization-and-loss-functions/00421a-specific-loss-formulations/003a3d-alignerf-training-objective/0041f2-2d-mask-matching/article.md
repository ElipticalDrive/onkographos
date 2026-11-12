# 2D Mask Matching

Also called: Mask IoU Assignment, Multi-view Mask Consistency

## Overview
2D Mask Matching aligns 2D instance masks predicted from different camera views to a consistent set of 3D instances detected by a NeRF-based detector (specifically NeRF-RCNN) [S1]. It solves the problem of inconsistent instance labeling across views, where the same physical object might receive different instance IDs or be missed entirely in different images [S1]. Unlike standard per-pixel losses, this method establishes a one-to-one correspondence between predicted 2D masks and 3D instances using Intersection over Union (IoU), ensuring multi-view consistency for the instance field [S1, S2].

## How it works
**Inputs:**
- A set of $N$ training images with corresponding camera poses.
- A pre-trained panoptic segmentation model (Mask2Former [S1]) initialized on COCO [S1].
- A 3D instance detection model (NeRF-RCNN) providing coarse 3D masks for each detected instance [S1].
- A class mapping from COCO categories to the target dataset categories [S1].

**Outputs:**
- A set of consistently annotated 2D instance masks for each training image, aligned to specific 3D instance IDs.
- Masks with insufficient overlap are marked as unlabeled and excluded from training [S1].

**Data Structures:**
- **3D Instance Masks:** A set of binary volumes or point clouds representing the coarse 3D shape of each detected instance $I_k$ from NeRF-RCNN.
- **2D Predicted Masks:** A list of binary masks $M_{pred}^{(v)}$ for each view $v$, output by Mask2Former.
- **Class Map:** A dictionary mapping COCO class IDs to dataset-specific class IDs [S1].

**Steps:**
1. **Initial Segmentation:** Run the pre-trained Mask2Former model on all training images to generate initial 2D panoptic segmentation masks [S1].
2. **Background Filtering:** Filter out masks belonging to background classes using the COCO-to-dataset class mapping [S1].
3. **3D-to-2D Projection:** For each 3D instance $I_k$ detected by NeRF-RCNN, project its 3D mask onto the 2D image plane of each training view $v$ to obtain a projected mask $M_{proj}^{(v, k)}$ [S1].
4. **IoU Computation:** For each view $v$, compute the Intersection over Union (IoU) between every predicted 2D mask $m_{pred} \in M_{pred}^{(v)}$ and every projected 3D instance mask $M_{proj}^{(v, k)}$ [S1]. The IoU for a pair is defined as:
   $$ \text{IoU}(m_{pred}, M_{proj}^{(v, k)}) = \frac{|m_{pred} \cap M_{proj}^{(v, k)}|}{|m_{pred} \cup M_{proj}^{(v, k)}|} $$
   where $|\cdot|$ denotes the number of pixels in the intersection or union [S1].
5. **Assignment:** Assign each predicted mask $m_{pred}$ to the 3D instance $I_k$ that yields the highest IoU score [S1].
6. **Thresholding:** If the maximum IoU for a predicted mask $m_{pred}$ is less than or equal to 0.05, it is treated as an unlabeled area and excluded from the instance NeRF training [S1].
7. **Refinement (Optional):** The aligned masks can be fed into an iterative refinement model to correct errors caused by partial occlusion or misprediction, producing refined masks for final instance field training [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def align_2d_masks(training_images, camera_poses, nerf_rcnn_instances, mask2former_model, coco_to_dataset_map, iou_threshold=0.05):
    """
    Aligns 2D predicted masks to 3D NeRF-RCNN instances.
    
    Args:
        training_images: List of input images.
        camera_poses: List of camera poses corresponding to images.
        nerf_rcnn_instances: List of 3D instance masks/volumes from NeRF-RCNN.
        mask2former_model: Pre-trained Mask2Former model.
        coco_to_dataset_map: Mapping from COCO class IDs to dataset class IDs.
        iou_threshold: Minimum IoU for a mask to be considered valid (default 0.05).
        
    Returns:
        aligned_masks: Dictionary mapping view index to list of (instance_id, mask) tuples.
        unlabeled_masks: List of masks excluded from training.
    """
    aligned_masks = {}
    unlabeled_masks = []
    
    # Step 1: Get initial 2D segmentation
    predicted_masks_list = mask2former_model.predict(training_images)
    
    # Step 2: Filter background
    filtered_masks_list = []
    for masks in predicted_masks_list:
        valid_masks = []
        for mask, class_id in masks:
            if class_id in coco_to_dataset_map:
                valid_masks.append((mask, coco_to_dataset_map[class_id]))
        filtered_masks_list.append(valid_masks)
        
    # Step 3 & 4: Project 3D instances and compute IoU
    # Note: Projection logic depends on the specific 3D representation (voxels/points)
    # Here we assume a function project_3d_mask_to_2d exists
    for v_idx, (image, pose) in enumerate(zip(training_images, camera_poses)):
        view_aligned = []
        view_unlabeled = []
        predicted_masks = filtered_masks_list[v_idx]
        
        for pred_mask, pred_class in predicted_masks:
            best_iou = -1.0
            best_instance_id = None
            
            for inst_id, inst_3d_mask in nerf_rcnn_instances.items():
                # Project 3D instance mask to current view
                proj_mask = project_3d_mask_to_2d(inst_3d_mask, pose)
                
                # Compute IoU
                intersection = np.logical_and(pred_mask, proj_mask).sum()
                union = np.logical_or(pred_mask, proj_mask).sum()
                if union == 0:
                    iou = 0.0
                else:
                    iou = intersection / union
                
                if iou > best_iou:
                    best_iou = iou
                    best_instance_id = inst_id
            
            # Step 5 & 6: Assign and Threshold
            if best_iou > iou_threshold:
                view_aligned.append((best_instance_id, pred_mask))
            else:
                view_unlabeled.append(pred_mask)
                
        aligned_masks[v_idx] = view_aligned
        unlabeled_masks.extend(view_unlabeled)
        
    return aligned_masks, unlabeled_masks

def project_3d_mask_to_2d(3d_mask, pose):
    """
    Projects a 3D instance mask to a 2D image plane.
    Implementation detail: [G] This typically involves rasterizing the 3D mask 
    using the camera intrinsic and extrinsic parameters.
    """
    # Placeholder for specific projection geometry
    # In practice, this converts 3D points/voxels to 2D pixels
    pass
```

**Parameters:**
- `iou_threshold`: 0.05. Masks with IoU $\le$ 0.05 are ignored [S1].

**Edge Cases and Pitfalls:**
- **Partial Occlusion:** Uniformly sampled images often contain partially occluded objects, which can mislead the pre-trained Mask2Former model, leading to missing or erroneous masks [S1].
- **Background Noise:** Background masks must be explicitly filtered using the class mapping; otherwise, they may incorrectly match to 3D instances [S1].
- **No Match:** If a predicted mask does not exceed the IoU threshold with any 3D instance, it is discarded. This prevents noise from corrupting the instance field [S1].

## Subtopics

## Operating characteristics
- **Time Complexity:** Dominated by the initial Mask2Former inference and the projection/IoU computation. The number of comparisons is $O(N_{views} \times N_{pred\_masks} \times N_{3d\_instances})$.
- **Memory Complexity:** Requires storing 3D instance masks and 2D predicted masks for all views.
- **Strengths:** Ensures multi-view consistency for instance fields, which is critical for high-quality instance decomposition [S1].
- **Limitations:** Relies heavily on the quality of the initial 3D instance detection (NeRF-RCNN) and the 2D segmentation model (Mask2Former). Errors in these models propagate to the instance field [S1].
- **When to use:** When training instance-specific NeRF fields (Instance-NeRF) where consistent labeling across views is required [S1].
- **When not to use:** For standard view synthesis without instance-level supervision, or when 3D instance detection is unavailable [S1].

## Background
The method was introduced in "Instance Neural Radiance Field" (Instance-NeRF) [S1]. It addresses the inconsistency of 2D instance masks across different views, which makes direct use in instance field training difficult [S1]. The approach builds on NeRF-RCNN for 3D instance detection and Mask2Former for 2D segmentation [S1].

## Sources
[S1] Instance Neural Radiance Field, https://doi.org/10.1109/ICCV51070.2023.00079, fetched 2026-10-29
