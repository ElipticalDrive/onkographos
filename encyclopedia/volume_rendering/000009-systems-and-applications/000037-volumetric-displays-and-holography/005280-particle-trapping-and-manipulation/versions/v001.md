# Particle Trapping and Manipulation
Also called: Free-space particle volumetric displays, Levitated particle displays

## Overview
Particle Trapping and Manipulation renders 3D images in free space by suspending microscopic or millimetric particles using optical or acoustic forces and illuminating them to exploit persistence of vision [S1, unofficial]. It solves the "clipping" limitation of holographic and light-field displays, where image points are physically separated from the scattering surface, thereby restricting viewing angles and preventing geometries such as long-throw projections or wrap-around displays [S1, unofficial]. By co-locating the scattering surface (the trapped particle) with the image point, these displays provide a 360° in-plane viewing angle around every voxel at any depth, enabling unconstrained viewing geometries [S1, unofficial].

## Shared principles
The family relies on the synchronization of particle position and illumination color/intensity. The core mechanism involves three stages:
1.  **Trapping:** A force field (optical or acoustic) creates a potential well that confines a single particle against gravity and drag.
2.  **Scanning:** The trap moves the particle along a 3D trajectory $(x(t), y(t), z(t))$ at a speed sufficient to blur the particle's motion into a continuous path via persistence of vision.
3.  **Illumination:** A light source (lasers or LEDs) illuminates the particle synchronously with its position to generate specific voxels with defined colors.

The choice of particle material and size is dictated by the trapping force:
*   **Optical Traps** require absorbing particles (e.g., cellulose) to generate photophoretic forces via thermal creep in gas [S1, unofficial].
*   **Acoustic Traps** require dielectric particles (e.g., expanded polystyrene) to experience radiation forces in standing waves [S1, unofficial].

## Subtopics

### [Photophoretic-Trap Volumetric Display](6753-photophoretic-trap-volumetric-display.md)
A free-space display using photophoretic forces to trap a microscopic particle [6753-photophoretic-trap-volumetric-display.md]. It is distinguished by its use of near-invisible trapping lasers and RGB illumination, enabling high-resolution voxels ($\approx 10 \mu m$) and unconstrained viewing angles [6753-photophoretic-trap-volumetric-display.md]. Choose this method for high-resolution, full-color 3D images where optical safety and hardware complexity are secondary to image quality [6753-photophoretic-trap-volumetric-display.md].

### [Acoustophoretic Volumetric Display](11093-acoustophoretic-volumetric-display.md)
A free-space display using acoustic radiation forces to trap a millimetric particle [11093-acoustophoretic-volumetric-display.md]. It is distinguished by its use of safe, low-cost ultrasonic phased arrays, allowing for a larger operational region (up to half a meter) without optical hazards [11093-acoustophoretic-volumetric-display.md]. Choose this method for larger-scale displays where safety and cost are priorities, accepting lower resolution and speed limitations compared to optical methods [11093-acoustophoretic-volumetric-display.md].

### Kumagai Volumetric Bubble Display
A method using bubbles excited by lasers in liquid as voxels [S1, unofficial]. By increasing liquid viscosity, slow-floating bubbles are generated and arranged to stack 2D cross-sections, realizing volumetric image display [S1, unofficial]. This system has a very long voxel residence time, requiring time for image refresh, but can express complex shapes [S1, unofficial].

## Background
The concept of trapping particles for volumetric display was pioneered by Smalley et al. in 2018, who used a $405 nm$ laser to trap $\approx 10 \mu m$ cellulose particles in air, illuminating them with coaxial RGB lasers to generate voxels [S1, unofficial]. This was the first method to adopt computer-hologram-based light irradiation patterns for volumetric displays, demonstrating that holographic design improves voxel count and brightness [S1, unofficial].

Acoustic variants emerged earlier; in 2014, "Pixie Dust" proposed using 285 ultrasonic transducers across four speakers to create acoustic ponderomotive fields for levitating multiple particles [S1, unofficial]. In 2019, Hirayama et al. demonstrated particle trapping using standing acoustic waves [S1, unofficial]. The Kumagai method utilizes laser-excited bubbles in viscous liquids, offering long residence times for complex shape expression [S1, unofficial].

## Sources
[S1] ボリュメトリックディスプレイ, https://www.jstage.jst.go.jp/article/jvrsj/29/3/29_27/_pdf/-char/ja, fetched 2026-10-30 (unofficial)
