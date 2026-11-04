# Monte Carlo Path Tracer (C++)

## Overview

This repository contains a **C++ Monte Carlo path tracer** developed as part of a **university project**.

The objective of this project was to explore **core rendering and algorithmic concepts** while working close to the hardware, with a strong focus on **performance, numerical stability, and software rigor**.

This project is academic in nature and was not intended as a production-ready renderer.

---

## Technical Focus

- C++ (modern C++)
- Monte Carlo integration
- Hemispherical sampling
- Bounding Volume Hierarchy (BVH)
- GPU experimentation (OpenGL / GLSL)
- Performance considerations and memory management

---

## Project Structure

```

DrawSampling/      Sampling experiments
MontecarloGPU/    GPU-based path tracing logic
bvh_gpu/          BVH construction and traversal
GL/               OpenGL helpers
shaders/           GLSL shaders
cmake/             CMake modules

```

---

## Build System

- CMake-based project
- Dependencies:
  - Eigen
  - GLFW
  - Assimp

Built and tested on Linux and Windows (via vcpkg).

---

## What this project demonstrates

Although academic, this project demonstrates:
- A solid understanding of C++ and low-level programming concepts
- The ability to implement and reason about complex algorithms
- Attention to performance constraints and numerical robustness
- Comfort working with large C++ codebases and build systems

---

## Context

This project was developed during my university studies as part of a computer graphics / rendering curriculum.
