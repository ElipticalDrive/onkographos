# Feature-based TF Design via ViT Features

Also called: Feature-based TF Design, Ours, Transfer Function Design via ViT Features

## Overview
This method designs transfer functions (TFs) for volume rendering by leveraging semantic features extracted from a pre-trained Vision Transformer (ViT), specifically DINO, to guide segmentation and opacity mapping. It solves the problem of ambiguous intensity-based segmentation in medical imaging (e.g., CT scans) where different tissues share similar Hounsfield units. Unlike traditional intensity-thresholding or supervised learning approaches that require extensive annotation or training, this method performs one-time feature extraction and uses a post-processing step (Block Label Smoothing) to generate similarity maps for TF design [S1]. It sets itself apart from its parent [DINO ViT Feature Extraction](6425-dino-vit-feature-extraction.md) by providing the specific annotation and post-processing pipeline that converts raw feature volumes into usable transfer functions, and it outperforms Support Vector Machines (SVM) and Random Forests (RF) in segmentation quality with significantly fewer annotations [S1].

## How it works
**Inputs and Outputs:**
- Input: Volumetric data $V$ (e.g., CT scan).
- Output: A Transfer Function (TF) defined by similarity maps or opacity values for volume rendering.

**Data Structures:**
- Feature Volume $F$: A 3D grid of semantic feature vectors.
  - Layout: $F \in \mathbb{R}^{H \times W \times D \times C}$, where $H, W, D$ are spatial dimensions and $C$ is the feature dimension (e.g., 384 for DINO).
  - Resolution: Typically $80^3$ for consumer GPUs [S1].
- Similarity Map: A scalar field derived from comparing query annotations against $F$.

**Steps:**
1. **Feature Extraction (Pre-processing):**
   - The input volume $V$ is processed by a pre-trained DINO ViT.
   - To achieve a target feature resolution, the input to the ViT is scaled by the patch size [S1].
   - Feature maps are extracted for all three slicing directions (axial, coronal, sagittal) [S1].
   - These maps are pooled to the desired feature size to form the Feature Volume $F$ [S1].
   - This step is performed once at the beginning; no training phase is required [S1].

2. **Annotation and Inference:**
   - The user provides sparse annotations (points or regions) for structures of interest within the volume [S1].
   - For each annotated point, the corresponding feature vector is retrieved from $F$.
   - During inference, the method computes a similarity measure between the feature vector of every voxel in $F$ and the annotated feature vectors. This inference time is constant regardless of the original volume resolution [S1].

3. **Post-processing (Block Label Smoothing - BLS):**
   - The raw similarity maps undergo a post-processing step called Block Label Smoothing (BLS) [S1].
   - This step refines the similarity maps to reduce noise and blockiness, adapting to the feature resolution [S1].
   - The output is a refined similarity map used to define the transfer function (e.g., opacity as a function of similarity) [S1].

**Parameters:**
- Feature Volume Resolution ($F_{res}$): $80^3$ is sufficient for consumer GPUs with 8GB VRAM [S1]. Higher resolutions are possible but memory-intensive [S1].
- Input Scaling Factor: Determined by the ViT patch size to match the desired $F_{res}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def design_tf_via_vit(volume, annotations, feature_resolution=80):
    """
    volume: 3D numpy array of shape (H, W, D) representing CT data.
    annotations: List of tuples (x, y, z, class_id) for user annotations.
    feature_resolution: Target resolution for the feature volume F.
    """
    # Step 1: Feature Extraction
    # Scale input to match ViT patch size requirements for target resolution
    # Note: Exact scaling formula depends on ViT patch size P.
    # scaled_volume = resize(volume, target_shape_for_vit)
    
    # Extract features using pre-trained DINO ViT
    # This produces feature maps for 3 slicing directions
    feature_maps_axial = extract_vit_features(scaled_volume, direction='axial')
    feature_maps_coronal = extract_vit_features(scaled_volume, direction='coronal')
    feature_maps_sagittal = extract_vit_features(scaled_volume, direction='sagittal')
    
    # Pool feature maps to desired resolution F_res
    # F is the aggregated feature volume
    F = pool_features([feature_maps_axial, feature_maps_coronal, feature_maps_sagittal], 
                      target_shape=(feature_resolution, feature_resolution, feature_resolution))
    
    # Step 2: Annotation-based Similarity Computation
    # Retrieve feature vectors for each annotation
    annotation_features = []
    for (x, y, z, class_id) in annotations:
        # Get feature vector at voxel (x,y,z) from F
        f_voxel = F[x, y, z]
        annotation_features.append((f_voxel, class_id))
        
    # Compute similarity map for each class
    similarity_maps = {}
    for class_id, f_annot in annotation_features:
        # Compute cosine similarity or Euclidean distance between f_annot and all voxels in F
        # Sim(x,y,z) = similarity(F[x,y,z], f_annot)
        sim_map = compute_similarity_map(F, f_annot)
        similarity_maps[class_id] = sim_map
        
    # Step 3: Post-processing (Block Label Smoothing)
    refined_tf_maps = {}
    for class_id, sim_map in similarity_maps.items():
        # Apply BLS to refine the similarity map
        refined_sim = block_label_smoothing(sim_map)
        refined_tf_maps[class_id] = refined_sim
        
    return refined_tf_maps
```

**Edge Cases and Pitfalls:**
- **Memory Limits:** High memory requirements prevent the use of larger transformer models (ViT-B, ViT-L) or models with higher patch sizes [S1]. All feature maps for three slicing directions must be saved simultaneously [S1].
- **Ambiguous Structures:** Structures with similar CT intensities (e.g., bladder vs. kidneys/tissue) may be recognized as similar by the feature extractor, requiring careful threshold selection in the TF [S1].
- **Block Artifacts:** Renderings may exhibit block artifacts if structures cannot be perfectly detected at their surfaces [S1].
- **Disconnected Components:** Connected components filtering can discard falsely detected disconnected structures (e.g., separate kidneys), but fails if structures are too close [S1].

## Subtopics

### Neural DINO Feature MLP
A 4-layer Multi-Layer Perceptron (MLP) that outputs DINO features ($f_{dino}$) for a 3D coordinate $x$. It is used in contexts where continuous feature fields are needed rather than discrete voxel grids. The hidden dimension is 256. The input 3D coordinate $x$ is encoded using 6 frequencies. The output feature dimension is 384 [S2].

### SAM Encoder Feature Extraction
Extraction of semantic features using a pre-trained Vision Transformer (ViT) from the Segment Anything Model (SAM). The SAM encoder uses an MAE pre-trained ViT-H/16 with 14x14 windowed attention and four equally-spaced global attention blocks. The input image is resized to 1024x1024 by padding the shorter side, then downsampled by a factor of 16 to obtain a 64x64 feature map. The feature dimension is 256. To handle aspect ratios, a 64x64 feature map is cropped to $64W/H \times 64$ (where $W>H$). During inference, the rendered feature is padded back to 64x64 so the SAM decoder receives equivalent semantic information. The padding region is cropped out to obtain the ground-truth feature containing only semantic information [S3].

## Operating characteristics
- **Time Complexity:** Feature extraction is a one-time pre-processing cost. Inference time for annotation-based TF generation is constant regardless of the original volume resolution [S1]. Post-processing (BLS) time varies with resolution [S1].
- **Memory Complexity:** High. Requires saving feature maps for three slicing directions. 8GB VRAM suffices for $80^3$ resolution [S1]. Higher resolutions or larger models (ViT-B/L) exceed consumer GPU budgets [S1].
- **Performance:** Outperforms SVM and Random Forest approaches in segmentation quality (mean IoU of 0.981 on CT-ORG) even when SVM/RF annotation counts are increased by three orders of magnitude [S1]. SVM and RF approaches are significantly slower [S1].
- **Strengths:** No training phase required; leverages self-supervised semantic features; robust to intensity ambiguities compared to thresholding; high segmentation quality with sparse annotations.
- **Limitations:** Memory-intensive; sensitive to similar appearances of different structures; potential for block artifacts in rendering; limited by ViT patch size and model size due to VRAM constraints [S1].
- **When to Use:** When semantic understanding of volume structures is needed, training data is scarce, and intensity-based segmentation fails due to tissue similarity. Suitable for consumer hardware if $80^3$ resolution is acceptable.
- **When Not to Use:** When high-resolution feature maps are critical and sufficient VRAM is unavailable; when structures are extremely close together and connected component filtering cannot separate them.

## Background
The method originates from research leveraging self-supervised Vision Transformers (ViT) for segmentation-based transfer function design in volume rendering. It was proposed to overcome the limitations of traditional intensity-based TF design and supervised learning methods that require extensive annotated data. The approach builds on the DINO ViT feature extraction technique [6425-dino-vit-feature-extraction.md] and was evaluated on medical imaging datasets like CT-ORG. It was published in IEEE Transactions on Visualization and Computer Graphics in 2024 [S1].

## Sources
[S1] Leveraging Self-Supervised Vision Transformers for Segmentation-Based Transfer Function Design., https://doi.org/10.1109/tvcg.2024.3401755, fetched 2026-10-26
[S2] PIR: Photometric Inverse Rendering with Shading Cues Modeling and Surface Reflectance Regularization, https://doi.org/10.1109/3dv66043.2025.00055, fetched 2026-10-26
[S3] Feature 3DGS: Supercharging 3D Gaussian Splatting to Enable Distilled Feature Fields, https://doi.org/10.1109/cvpr52733.2024.02048, fetched 2026-10-26
