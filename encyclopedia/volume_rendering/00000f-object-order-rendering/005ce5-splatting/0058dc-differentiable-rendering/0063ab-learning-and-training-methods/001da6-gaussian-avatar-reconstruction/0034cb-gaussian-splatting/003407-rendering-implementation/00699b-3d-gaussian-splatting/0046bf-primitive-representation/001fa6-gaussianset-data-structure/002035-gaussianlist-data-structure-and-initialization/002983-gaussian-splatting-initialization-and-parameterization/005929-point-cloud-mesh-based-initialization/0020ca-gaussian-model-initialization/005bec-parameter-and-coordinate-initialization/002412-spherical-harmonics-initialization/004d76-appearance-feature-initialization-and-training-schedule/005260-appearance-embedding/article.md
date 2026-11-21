# Appearance Embedding
mean_embedding = [sum(v) / len(training_embeddings) for v in zip(*training_embeddings)]
return training_embeddings, mean_embedding
```

## Subtopics

### Feature Embedding for Illumination
This variant replaces spherical harmonics with a feature embedding $f$ and a trainable per-image embedding $e$ to handle intense illumination changes, such as multiple light sources in indoor scenes [S4]. It retains the opacity parameter $\alpha$ consistent with standard 2D Gaussian Splatting [S4]. This approach is chosen when spherical harmonics are insufficient for modeling complex lighting [S4].

## Operating characteristics
*   **Performance:** Improves robustness to lighting variations and can mitigate floater artifacts in geometry-critical tasks [S3, unofficial].
*   **Strengths:** Handles dynamic lighting, shadows, and appearance changes effectively [S1, unofficial][S2].
*   **Limitations:** Requires per-image optimization; validation requires a heuristic (mean embedding) [S1, unofficial].

## Background
Appearance embeddings were originally introduced in "NeRF in the Wild" to address lighting variations in uncontrolled environments [S3, unofficial]. The concept has been extended to various rendering frameworks, including K-Planes (WildGaussians) and Convex Splatting (WildConvexes) [S3, unofficial]. It is commonly used in large-scale scene reconstruction where lighting consistency is challenging [S1, unofficial].

## Sources
[S1] SCALAR-NeRF: SCAlable LARge-scale Neural Radiance Fields for Scene Reconstruction, https://arxiv.org/abs/2311.16657v1, fetched 2026-10-26 (unofficial)
[S2] Neural Radiance Fields-Based 3D Reconstruction of Power Transmission Lines Using Progressive Motion Sequence Images., https://doi.org/10.3390/s23239537, fetched 2026-10-26
[S3] On the Geometric Accuracy of Implicit and Primitive-based Representations Derived from View Rendering Constraints, https://doi.org/10.48550/arXiv.2509.10241, fetched 2026-10-30 (unofficial)
[S4] OMeGa: Joint Optimization of Explicit Meshes and Gaussian Splats for Robust Scene-Level Surface Reconstruction, https://doi.org/10.1109/WACV61042.2026.00427, fetched 2026-10-30
