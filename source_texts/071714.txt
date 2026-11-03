# Plenoxels alpha

Volumetric rendering using ReLU fields.  

![Rendered on CPU](https://github.com/RevelcoS/Plenoxels/raw/main/out_cpu.png)

## Features

* Approximate colored rendering

## Setup

The project setup is only available on Windows under [MSYS2](https://www.msys2.org/) UCRT64 environment.  

Download dependencies:

```sh
./libs.sh
```

Download dependencies from GitHub:

```sh
git submodule init && git submodule update
```

Setup libraries (LiteMath, stb):

```sh
make libs
```

Build the project:

```sh
make all
```

## Render

To run the project execute:

```sh
make run
```

## Scene

The sparse grid model must be on path `assets/model.dat`.  
  
Camera description from scene/camera.txt:  

```txt
Position <float3>position
Direction <float3>direction
Up <float3>up
FOV <float>FOV
```

## Runtime

CPU: Intel Core i7-7700HQ 2.80GHz  
Memory: SODIMM 16GB 2.40GHz  
GPU: NVIDIA GeForce GTX 1060

```sh
Render with CPU (1 thread):     7.83239s
Render with OpenMP (4 threads): 1.4008s
```
