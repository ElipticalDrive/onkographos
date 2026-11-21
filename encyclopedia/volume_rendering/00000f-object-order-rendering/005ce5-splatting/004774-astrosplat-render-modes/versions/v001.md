# AstroSplat Render Modes
Also called: Basic and Enhanced modes

## Overview
AstroSplat Render Modes provide two distinct visualization strategies for rendering splat fields: **Basic** and **Enhanced** [S1, unofficial]. Basic mode prioritizes clarity and rendering speed through a simpler splat presentation, while Enhanced mode offers a perceptual experience with stronger depth cues and a richer volumetric feel, though it does not simulate physical radiative transfer [S1, unofficial].

## Shared principles
The AstroSplat family operates within the broader Splatting framework, treating volumetric data as a set of transformed primitives rasterized onto the screen [S1, unofficial][S2, S3, S4, unofficial]. Both modes share the underlying mechanism of decomposing the volume into primitives (such as Gaussian splats) defined by position, density, and reconstruction kernels, which are then transformed via Model-View-Projection matrices and composited using hardware-accelerated alpha blending [S1, unofficial][S2, S3, S4, unofficial]. The distinction lies solely in the visual parameterization and compositing weights applied during the rasterization and blending stages to achieve the desired perceptual outcome.

## Subtopics

### Basic Mode
A simplified rendering strategy designed for computational efficiency and visual clarity. It reduces the complexity of the splat presentation to maximize frame rates, making it suitable for large datasets or interactive exploration where speed is paramount [S1, unofficial].

### Enhanced Mode
A perceptual rendering strategy that emphasizes depth perception and volumetric richness. It applies stronger depth cues and adjusted blending weights to create a more immersive "volumetric feel," distinguishing it from physically accurate models by prioritizing visual interpretability over radiative correctness [S1, unofficial].

## Background
The AstroSplat Render Modes originate from the AstroSplat project, specifically the repository `VitorFigueiredoResearch/AstroSplat-Browser-native-volumetric-visualization-for-astrophysical-vector-fields` [S1, unofficial]. The project focuses on native volumetric visualization for astrophysical vector fields, leveraging browser-based or native GPU acceleration to handle complex magnetic field data. The introduction of dual modes allows users to toggle between performance-critical analysis (Basic) and presentation-oriented visualization (Enhanced) [S1, unofficial].

## Sources
[S1] VitorFigueiredoResearch/AstroSplat-Browser-native-volumetric-visualization-for-astrophysical-vector-fields, https://github.com/VitorFigueiredoResearch/AstroSplat-Browser-native-volumetric-visualization-for-astrophysical-vector-fields, fetched 2026-10-26 (unofficial)
