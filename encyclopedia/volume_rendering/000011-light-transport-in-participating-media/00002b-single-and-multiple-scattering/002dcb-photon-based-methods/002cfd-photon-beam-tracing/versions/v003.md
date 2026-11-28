# Photon Beam Tracing

Also called: Photon beams

## Overview
Photon Beam Tracing is a volumetric radiance estimation method where photon paths are stored as beam segments rather than points, allowing for density estimation via beam intersection [S1]. It differs from standard volumetric photon tracing by storing photon energy up to the point of intersection with solid geometry, rather than discarding the photon upon collision [S1]. The photon beams map contains a superset of the information in a standard photon map [S1].

## How it works
**Inputs and Outputs:** Not given in the sources.

**Data Structures:** Beam segments are stored in a volume photon map, replacing or augmenting standard point-based photon storage [S1].

**Steps:**
1. Shoot photon paths from light sources [S1].
2. Store beam segments wherever standard volume photons are stored, with initial beam start points originating at the light source [S1].
3. When a photon path intersects a solid object, store a beam with an end point at the object intersection, rather than discarding the photon as in standard volumetric photon tracing [S1].

**Formulas:** The method computes a Beam × Point 3D radiance estimate adapted from the Beam Radiance Estimate (BRE) [S1]. Specific formulas are not given in the sources.

**Parameters:** Beam radius $r$. A fixed-width blur is implemented by interpreting each photon beam as a cylinder of a specified radius [S1]. Choosing a single blurring width that works well for the whole domain is often impossible [S1]. Details on adaptive radius adjustment are not given in the sources.

**Edge Cases and Pitfalls:** Not given in the sources.

**Pseudocode:** Not given in the sources.

## Subtopics

### Volume Photon Map Lookup
This is the process of estimating inscattering contribution during eye ray traversal using the pre-computed volume photon map [S2, unofficial]. For each step during eye ray traversal, a lookup is performed in the volume photon map [S2, unofficial]. Unlike surface photon map lookups which always use final gathering, volume photon map lookups are performed without a final gather step [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Photon Beam Tracing was introduced as an extension of volumetric photon mapping [S1]. It builds on the theory of beam radiance estimates (BRE) [S1]. The method allows for the integration of photon beams into existing volumetric photon mapping implementations with minor modifications to the photon shooting preprocess [S1].

## Sources
[S1] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
[S2] CS348b Final Project - Rendering Underwater Scenes, http://www.graphics.stanford.edu/courses/cs348b-competition/cs348b-04/underwater/index.html, fetched 2026-10-30 (unofficial)
