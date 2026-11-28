# Wild Scene Preprocessing & Test Set Selection

Also called: Data Cleanup, Image filtering for unconstrained datasets, Phototourism Dataset Pre-filtering

## Overview
A coarse pre-filtering pipeline to remove low-quality images dominated by transient objects from unconstrained photo collections, ensuring robust training and evaluation for view synthesis systems [S1][S2, unofficial].

## How it works
**Inputs:** Unconstrained photo collection (images and metadata) [S1][S2, unofficial].
**Outputs:** Filtered training set and curated test set [S1][S2, unofficial].

**Data Structures:**
*   Image: Standard raster format.
*   NIMA Score: Float [0, 10].
*   Transient Occupancy: Float [0, 1] (fraction of image area classified as transient) [S1][S2, unofficial].

**Steps:**
1.  **Sharpness Filtering:** Compute the variance of Laplacians for each image; discard images more than 1.5 standard deviations below the mean sharpness [S3].
2.  **Semantic & Motion Filtering:** Run Mask R-CNN to segment people, animals, and vehicles [S3]. Mask out all pixels belonging to people and animals [S3]. For vehicles, check for Structure-from-Motion (SfM) points with reprojection error < 1.5 pixels under the vehicle masks; discard vehicles with low SfM point density (indicating motion) [S3]. Remove vehicle license plates using EgoBlur, constraining masks to segmented vehicle regions [S3].
3.  **Quality & Transient Filtering:** Compute the NIMA score for each image [S1][S2, unofficial]. Compute transient occupancy by running a DeepLab v3 model (trained on ADE20k) and calculating the fraction of image area classified as transient objects [S1][S2, unofficial].
4.  **Training Set Curation:** Remove images with a NIMA score below 3 or transient occupancy exceeding 80% [S1][S2, unofficial].
5.  **Test Set Selection:** From the remaining pool, identify candidates with NIMA scores in the 70th percentile and transient occupancy in the bottom 25% [S2, unofficial].
6.  **Manual Verification:** Manually review candidates to minimize transient occluders and ensure well-focused, representative qualities [S1][S2, unofficial].

**Parameters:**
*   Sharpness discard threshold: > 1.5 standard deviations below mean [S3].
*   SfM reprojection error threshold: < 1.5 pixels [S3].
*   NIMA removal threshold: 3 [S1][S2, unofficial].
*   Transient occupancy removal threshold: 80% [S1][S2, unofficial].
*   Test set NIMA percentile: 70th [S2, unofficial].
*   Test set transient occupancy percentile: Bottom 25% [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_wild_scene(images, nima_scores, deeplab_segmentations, mask_rcnn_masks, sfm_points, sharpness_mean, sharpness_std):
    training_images = []
    test_candidates = []

    for i, img in enumerate(images):
        # Step 1: Sharpness filtering
        sharpness = compute_variance_of_laplacians(img)
        if sharpness < sharpness_mean - 1.5 * sharpness_std:
            continue

        # Step 2: Semantic & motion filtering
        people_animal_mask = mask_rcnn_masks[i]['people'] | mask_rcnn_masks[i]['animals']
        vehicle_mask = mask_rcnn_masks[i]['vehicles']
        img.apply_mask(people_animal_mask)
        if vehicle_mask.any():
            sfm_points_under_vehicle = [p for p in sfm_points if point_in_mask(p, vehicle_mask)]
            if len(sfm_points_under_vehicle) < 100: # [G]
                continue
        img = apply_egoblur(img, vehicle_mask)

        # Step 3 & 4: Quality & transient filtering
        nima = nima_scores[i]
        seg = deeplab_segmentations[i]
        transient_pixels = count_class_pixels(seg, 'transient')
        total_pixels = img.width * img.height
        transient_ratio = transient_pixels / total_pixels

        if nima >= 3.0 and transient_ratio <= 0.8:
            training_images.append(img)
            test_candidates.append({'image': img, 'nima': nima, 'ratio': transient_ratio})

    # Step 5: Test set selection
    if not test_candidates:
        return training_images, []

    nima_vals = sorted([c['nima'] for c in test_candidates])
    ratio_vals = sorted([c['ratio'] for c in test_candidates])
    nima_threshold = nima_vals[int(0.7 * len(nima_vals))]
    ratio_threshold = ratio_vals[int(0.25 * len(ratio_vals))]

    final_test_set = [
        c['image'] for c in test_candidates
        if c['nima'] >= nima_threshold and c['ratio'] <= ratio_threshold
    ]

    # Step 6: Manual verification required [S1][S2, unofficial]
    return training_images, final_test_set

def compute_variance_of_laplacians(img):
    # Standard operation: apply Laplacian kernel, compute variance of response [G]
    return standard_variance(apply_laplacian_kernel(img))

def count_class_pixels(seg, class_name):
    return sum(1 for pixel in seg if pixel.label == class_name)

def point_in_mask(point, mask):
    return mask[point.y, point.x]
```

## Operating characteristics
*   **Complexity:** Dominated by inference time of NIMA and DeepLab v3 models. Not given in the sources.
*   **Performance:** Applied to scenes like Brandenburg Gate (763 images, 564M pixels), Sacré Cœur (830 images, 605M pixels), Trevi Fountain (1,689 images, 1249M pixels), Taj Mahal (811 images, 581M pixels), Prague (2,000 images, 1417M pixels), and Hagia Sophia (606 images, 434M pixels) [S1].
*   **Strengths:** Removes low-quality and highly occluded images, ensuring robust training and evaluation [S1][S2, unofficial].
*   **Limitations:** Relies on the accuracy of pretrained models (NIMA, DeepLab) [S2, unofficial]. Manual verification for test sets is required [S1][S2, unofficial].

## Background
Originated in NeRF in the Wild (NeRF-W) [S1] to address the challenge of evaluating view synthesis on unconstrained photo collections. Extended in UP-NeRF [S2, unofficial] with specific percentile-based criteria for test set selection. Additional cleanup steps for sharpness, motion, and license plates are documented in hierarchical 3D Gaussian rendering pipelines [S3].

## Sources
[S1] NeRF in the Wild: Neural Radiance Fields for Unconstrained Photo Collections, https://doi.org/10.1109/cvpr46437.2021.00713, fetched 2026-10-26
[S2] UP-NeRF: Unconstrained Pose-Prior-Free Neural Radiance Fields, https://arxiv.org/abs/2311.03784v2, fetched 2026-10-26 (unofficial)
[S3] A Hierarchical 3D Gaussian Representation for Real-Time Rendering of Very Large Datasets, https://www.youtube.com/watch?v=p-mb2TzVJZk, fetched 2026-10-31
