# Interest-Based NPR Toolkit

Also called: Non-Photorealistic Rendering (NPR) Toolkit, Variable-Detail NPR

## Overview
The Interest-Based NPR Toolkit is a flexible framework within the Volume Illustration pipeline that generates 2D non-photorealistic illustrations by associating user-defined interest labels with subsets of rendering techniques [S1, unofficial]. It solves the problem of programmatic bias in automated visualization by allowing variable detail levels per segment, where different labels trigger different artistic styles simultaneously [S1, unofficial]. Unlike physics-based rendering, it prioritizes communicative abstraction and artistic vision over physical accuracy [S1, unofficial].

## How it works
The method operates on an enriched model consisting of a geometric model (identified segments) and a structural model (associated interest labels) [S1, unofficial].

**Inputs:**
1. **Geometric Model:** A set of segments derived from segmentation [S1, unofficial].
2. **Structural Model:** A mapping from segment IDs to interest labels [S1, unofficial].
3. **NPR Technique Library:** A set of implemented techniques (greyscale shading, simple outlines, copper plate lines, short hatching lines) [S1, unofficial].

**Outputs:**
A single 2D illustration image where each segment is rendered according to the NPR techniques associated with its interest label [S1, unofficial].

**Data Structures:**
* **Segment Map:** A shadow volume recording the segment associated with each voxel [S1, unofficial].
* **Label Map:** A mapping `SegmentID -> InterestLabel` (integer) [S1, unofficial].
* **Technique Map:** A mapping `InterestLabel -> Subset{NPR_Technique}` representing a one-to-many association [S1, unofficial].

**Numbered Steps:**
1. **Segmentation:** Identify segments in the volume data. The system assumes all voxels in a segment are equally interesting (Single Interest Volume assumption) [S1, unofficial].
2. **Enrichment:** Assign an interest label to each segment to augment the geometric model with structural information, forming an enriched model [S1, unofficial].
3. **Association:** The user defines a one-to-many mapping where each interest label is associated with a subset of available NPR techniques to avoid programmatic biases [S1, unofficial].
4. **Rendering:** Render all NPR techniques associated with all interest labels simultaneously to generate the final 2D illustration [S1, unofficial].

**Formulas:**
Not given in the sources.

**Parameters:**
* **Number of Interest Labels:** Defaults to 5, an arbitrary decision generalizable to any number [S1, unofficial].
* **Default Label:** All segments default to label 3 if not explicitly assigned [S1, unofficial].
* **Technique Properties:** Techniques are customizable to portray silhouettes, surface geometry, value, light and shade, and a primitive form of texture [S1, unofficial].

**Edge Cases and Pitfalls:**
Segmentation avoids the algorithmic deficiencies of isosurfacing, such as poor topological properties and the need for prior dataset knowledge [S1, unofficial]. Automatic label allocation is impossible because the system cannot infer the user's artistic vision [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_blank_image(width, height):
    """Initialize a 2D buffer with zero values."""
    return [[0.0] * width for _ in range(height)]

def apply_npr_technique(voxels, technique_name, image_buffer):
    """
    Applies a specific NPR technique to the geometry defined by voxels.
    The sources do not provide the mathematical implementation for the NPR techniques.
    Standard operations would compute gradients, lighting, or line projections per voxel.
    """
    # [G] Apply technique rendering logic to voxels in image_buffer
    pass

def render_interest_based_npr(segment_map, label_map, technique_map, width, height):
    """
    segment_map: Dict mapping SegmentID to list of Voxels
    label_map: Dict mapping SegmentID to InterestLabel (int)
    technique_map: Dict mapping InterestLabel to List of Technique Strings
    """
    output_image = create_blank_image(width, height)
    
    for segment_id, voxels in segment_map.items():
        interest_label = label_map[segment_id]
        techniques = technique_map.get(interest_label, [])
        
        for technique in techniques:
            apply_npr_technique(voxels, technique, output_image)
            
    return output_image
```

## Subtopics

### Interest label assignment
A generalization of binary segmentation where the level of interest in each segment is quantified by assigning an interest label [S1, unofficial]. The relationship between segments and labels is many-to-one [S1, unofficial]. The user assigns one of five labels to each segment in a synchronized structural view [S1, unofficial]. This supports variable detail levels without enforcing a specific hierarchy [S1, unofficial].

## Operating characteristics
* **Complexity:** Rendering uses a fast, pixel-based, geometric buffer approach to maintain interactivity [S1, unofficial].
* **Memory:** Supports single-level undo; multiple-level undo is a future improvement requiring extra memory or computation [S1, unofficial].
* **Strengths:** High flexibility, avoidance of programmatic biases, and creative artistic control [S1, unofficial].
* **Limitations:** Requires manual label assignment and technique mapping, making it non-fully automatic [S1, unofficial].

## Background
Originated in the 2002 thesis "Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration" by J.L. McKinley [S1, unofficial]. It builds on the concept of enriched models combining geometric and structural information [S1, unofficial]. It follows the lead of Ebert and Rheingans in providing a flexible NPR toolkit [S1, unofficial].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
