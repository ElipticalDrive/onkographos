# Data Augmentation with Virtual Object Insertion

Also called: AR data augmentation, Lighting-Augmented Data Augmentation, Virtual Object Insertion for Training Data, Virtual object insertion for perception

## Overview
This method utilizes differentiable object insertion to automatically generate synthetic training data for downstream 3D object detection tasks by inserting diverse 3D assets into real-world images [S1]. It distinguishes itself from category-specific image manipulation methods by predicting physics-based scene lighting, making it agnostic to object class and enabling the insertion of rarely observed categories [S1].

## How it works
The method operates as an automatic pipeline that augments a subset of real-world images with virtual objects, leveraging the differentiable object insertion pipeline [Parent](19344-differentiable-object-insertion.md) to ensure physically consistent lighting and shadows [S1].

**Inputs:**
1.  A subset of real-world images (e.g., front-camera images from nuScenes) [S1].
2.  A set of 3D models for target categories (e.g., cars, construction vehicles) [S1].
3.  Scene depth maps and camera intrinsics (inherited from the parent differentiable insertion pipeline) [S1, Parent].

**Outputs:**
1.  Augmented images with inserted virtual objects, realistic lighting effects (cast shadows, specular highlights), and occlusion handling [S1].
2.  Free labels for the inserted objects (3D bounding box and orientation) [S1].

**Data Structures:**
*   **G-buffer:** Caches the depth of the inserted object for occlusion resolution [S1].
*   **Scene Depth Map:** The ground truth or estimated depth of the real scene [S1].

**Steps:**
1.  **Collection:** Collect a set of 3D models for target categories. Sources cite assets from artists such as Hum3D, be fast, rabser, FirelightCGStudio, amaranthus, 3DTree_LLC, 3dferomon, and Pipon3D [S1].
2.  **Insertion:** For each image in the training subset, randomly select one 3D object and insert it into the scene using the differentiable object insertion pipeline [S1]. The pipeline ensures the object is influenced by the estimated physics-based scene lighting [S1].
3.  **Occlusion Handling:** Compare the depth of the inserted object (cached in the G-buffer) with the scene depth map. Display the pixel corresponding to the closer depth value to handle occlusion correctly [S1].
4.  **Training Data Generation:** The resulting augmented images, along with the known 3D bounding box and orientation of the inserted objects, form the synthetic training dataset [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_augmented_dataset(real_images, depth_maps, camera_intrinsics, model_library):
    augmented_images = []
    augmented_labels = []
    
    # Select a subset of real data (e.g., 10% of nuScenes training set)
    subset = select_subset(real_images, depth_maps, camera_intrinsics, ratio=0.1)
    
    for image, depth_map, intrinsics in subset:
        # Randomly select a 3D model from the library
        model = random.choice(model_library)
        
        # Randomly place the object in the scene
        # The differentiable insertion pipeline handles lighting estimation
        # and compositing based on known geometry M and material Θ
        inserted_image, g_buffer = differentiable_object_insert(
            image=image,
            depth_map=depth_map,
            intrinsics=intrinsics,
            model=model
        )
        
        # Handle occlusion: compare inserted object depth with scene depth
        # Display the closer pixel
        final_image = resolve_occlusion(inserted_image, g_buffer, depth_map)
        
        # Extract free labels (3D bounding box and orientation)
        # These are known because the object's pose is chosen/generated
        labels = get_object_labels(model, g_buffer)
        
        augmented_images.append(final_image)
        augmented_labels.append(labels)
        
    return augmented_images, augmented_labels
```

**Parameters:**
*   **Augmented Dataset Size:** Approximately 15,000 new images [S1].
*   **Training Strategy:** Same strategy and hyperparameters as the baseline detector [36], but supervision for attributes and velocity is omitted as they are absent in augmented data [S1].

**Edge Cases and Pitfalls:**
*   **Night-time Scenes:** The model can predict reasonable results for night-time scenes despite lacking HDR data supervision for this domain, relying instead on limited FoV LDR data and adversarial supervision [S1].
*   **Category Agnosticism:** Unlike methods with underlying assumptions about object categories, this method's reliance on physics-based lighting estimation allows it to handle classes rarely observed in the real world (e.g., construction vehicles) without degradation [S1].

## Operating characteristics
*   **Performance:** The detector's mean Average Precision (mAP) improves by 2% overall compared to training on real data alone [S1].
*   **Lighting Impact:** An additional 1% of the mAP gain is attributed specifically to better lighting estimation, while the other 1% comes from the naive addition of objects [S1].
*   **Generalization:** Performance improves in categories not directly augmented, indicating the method enhances the detector's general robustness [S1].
*   **Strengths:** Fully automatic process; produces realistic lighting effects (cast shadows, "clear coat" reflections); agnostic to object class; enables insertion of rare classes [S1].
*   **Limitations:** Primarily focused on daytime outdoor scenes; night-time results rely on adversarial supervision due to scarcity of ground truth lighting [S1].

## Background
This method was proposed in the context of Neural Light Field Estimation for Street Scenes, aiming to solve the scarcity of diverse training data for 3D object detectors [S1]. It builds upon differentiable rendering techniques to allow end-to-end optimization of AR objectives and lighting estimation [S1]. The approach addresses the limitations of existing image manipulation methods that assume specific object categories, offering a more generalizable solution for data augmentation in autonomous driving scenarios [S1].

## Sources
[S1] Neural Light Field Estimation for Street Scenes with Differentiable Virtual Object Insertion, https://doi.org/10.1007/978-3-031-20086-1_22, fetched 2026-10-30
