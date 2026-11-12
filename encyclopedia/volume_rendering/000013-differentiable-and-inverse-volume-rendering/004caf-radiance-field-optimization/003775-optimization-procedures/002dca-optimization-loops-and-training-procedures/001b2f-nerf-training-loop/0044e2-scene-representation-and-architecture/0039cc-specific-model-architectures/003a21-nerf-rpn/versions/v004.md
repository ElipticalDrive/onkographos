# NeRF-RPN

Also called: NeRF Region Proposal Network

## Overview
NeRF-RPN is a framework for 3D object detection and instance segmentation that utilizes a multi-scale Object-NeRF architecture to predict 3D axis-aligned bounding boxes (AABB) and object classes directly from a neural radiance field [S1]. It distinguishes itself from standard NeRF variants by enabling the regression of 3D bounding boxes without requiring rendering from multiple viewpoints, and it supports the detection of objects without pre-defined class labels [S1].

## How it works
**Inputs and Outputs:**
The method takes a scene representation as input and outputs 3D bounding boxes, class labels, and discrete 3D masks for detected objects. The core input is a multi-scale Object-NeRF structure [S1].

**Data Structures:**
The architecture consists of two parallel branches:
1.  **Scene Branch:** Encodes the global geometry and appearance of the scene using a voxel representation of 3D neural volume features [S1].
2.  **Object Branch:** Encodes each independent object within the scene [S1].

**Method Steps:**
1.  **Scene Encoding:** The scene branch processes the input to generate a voxel-based representation of the 3D neural volume features, capturing the global geometry and appearance [S1].
2.  **Object Encoding:** The object branch processes the scene to encode individual objects. This branch operates independently for each detected object instance [S1].
3.  **Proposal Generation:** The network regresses 3D bounding boxes for objects present in the scene. This regression is performed directly on the neural field representation, avoiding the need to render intermediate views [S1].
4.  **Classification:** Learnable object activation codes are used to classify the target entities within the scene [S1].
5.  **Refinement (via NeRF-RCNN):** While NeRF-RPN generates proposals, the refinement into final discrete instance masks and precise bounding boxes is handled by the subsequent NeRF-RCNN stage, which appends box and mask prediction heads [S1].

**Parameters:**
*   **Multi-scale:** The architecture utilizes multiple scales for the Object-NeRF representation [S1].
*   **Activation Codes:** Learnable codes are used for object classification [S1].

**Edge Cases and Pitfalls:**
*   The method supports copying, moving, or rotating objects but is limited in its ability to edit the spatial position of objects within the scene [S1].
*   It is a general framework capable of detecting objects without class labels [S1].

## Subtopics

### [NeRF-RCNN](10610-nerf-rcnn.md)
Extends NeRF-RPN by appending box and mask prediction heads to refine proposals into final 3D instances, outputting discrete masks and bounding boxes.

## Operating characteristics
*   **Strengths:** Enables 3D object detection and bounding box regression without rendering from multiple viewpoints; supports detection of unlabelled objects; provides a general framework for instance-level understanding [S1].
*   **Limitations:** Cannot change the spatial position of objects during editing; limited to copy, move, or rotate operations [S1].
*   **Performance:** Not given in the sources.

## Background
NeRF-RPN was proposed by Hu et al. in 2023 as part of the broader application of neural radiance fields to 3D object detection and semantic segmentation tasks [S1]. It addresses the need for detecting object boundaries in 3D space using the explicit representation capabilities of NeRF.

## Sources
[S1] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
