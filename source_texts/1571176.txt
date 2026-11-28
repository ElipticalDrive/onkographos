# GPU_Radix_Sort

#### Introduction

In this project, I implemented **Radix Sort** using **Vulkan** and **CUDA** respectively and conducted a performance comparison between the two. However, I found that my Vulkan implementation performs significantly slower than the CUDA version, and I have yet to identify the root cause.

**Reference Blog (CSDN):** [https://blog.csdn.net/qq_46348003/article/details/156650297](https://www.google.com/search?q=https://blog.csdn.net/qq_46348003/article/details/156650297)

**Open-source Reference Projects:**

- https://github.com/jaesung-cs/vulkan_radix_sort
- https://github.com/b0nes164/GPUSorting

**Inspirational Video (Bilibili):** https://www.bilibili.com/video/BV1PEW4zbEKR/

- The original intent of this project was to study parallel algorithms and master the practical techniques of Vulkan and CUDA. Consequently, I chose to reimplement the algorithm from scratch. 
- This small project will also be integrated into a subsequent **3D Gaussian Splatting (3DGS)** implementation.

![p](doc/p.png)

#### Build

- The project uses C++17 and only depends on Vulkan and CUDA.
- Use **CMake** to configure the project, then run it directly. The project consists of a single target where the `main` file invokes the necessary interfaces to perform the comparison and display the results.

#### Demonstration

- Temporal performance is dependent on the **GPU model**, the **API** used, the **array length**, and the **range of random values**.
- Sorting $2^{30}$ numbers requires significant VRAM; specifically, 16GB of VRAM is insufficient for a dataset of this size.

| RTX5080       | 2^27  | 2^28  | 2^29  | RTX3060 Laptop | 2^27  | 2^28  | 2^29  |
|:-------------:|:-----:|:-----:|:-----:|:--------------:|:-----:|:-----:|:-----:|
| cuda Uint32   | 8.478 | 16.65 | 33.29 | cuda Uint32    | 22.27 | 42.94 | 94.16 |
| cuda Float    | 11.01 | 22.16 | 43.74 | cuda Float     | 30.29 | 56.97 | 128.2 |
| Vulkan Uint32 | 420.9 | 896.1 | 1796  | Vulkan Uint32  | 591.3 | 997.1 | 1851  |
| Vulkan Float  | 456.6 | 993.2 | 1863  | Vulkan Float   | 733.1 | 1225  | 2294  |

- IN RTX5080 by cuda

![test1](doc/test1.png)

- IN RTX3060 Laptop by vulkan

![test2](doc/test2.png)
