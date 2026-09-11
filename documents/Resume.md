# Xiaojun Wang

San Diego, CA | 517-899-1570 | [wxjmsu@gmail.com](mailto:wxjmsu@gmail.com) | [linkedin.com/in/xiaojun-wang](https://www.linkedin.com/in/xiaojun-wang) | [github.com/gpuwangge](https://github.com/gpuwangge)

## Professional Summary
Staff GPU Engineer with 8+ years of industry experience spanning GPU architecture, functional modeling, fixed-function graphics, validation, and GPU IP integration. Deep background in GPU simulation and graphics pipelines, with hands-on experience in architectural feature development, performance analysis, hardware/software co-design, and graphics software.

## Professional Experience

### Staff GPU Engineer / Technical Lead
**MediaTek** — San Diego / San Jose, CA  
**May 2023 – Present**
- Led **Arm Mali GPU IP integration** for SoC platforms, including IP-level validation, **graphics-driver** enablement, system bring-up, and cross-functional issue triage.
- Served as **Geometry High Level Model Lead** for an in-house GPU, driving geometry-pipeline modeling(Triangle assemble/setup, Clipping, Tiling, TBDR, DVS, Compression...) and integrating a texture-compression block.
- Worked as a **GPU Architect** on a confidential next-generation GPU program, defining and developing new architectural features.
- Performed **GPU benchmark**(Aztec, Manhattan...) validation and performance profiling using PVRTune to identify bottlenecks and support optimization. Designed and Implemented Top-level Vulkan tests.  

### Senior GPU Engineer
**Qualcomm** — San Diego / Santa Clara, CA  
**January 2018 – May 2023**
- Conducted accurate high-level modeling and analysis for a large-scale C++ GPU simulator across Linux and Windows environments.
- Owned and trained engineers on two major fixed-function graphics blocks: **Tessellation** and **Primitive Control**.
- Developed features for the **CCU (Cache and Compression Unit)** and maintained source code across five fixed-function blocks in the graphics pipeline.
- Delivered two major features and more than 10 minor features spanning the 6th, 7th, and 8th generations of Adreno GPUs.
- Resolved more than 1,000 modeling bugs across multiple Qualcomm SoCs, improving simulation correctness and platform readiness.
- Created and delivered more than 200 bit-accurate unit tests using DirectX 11, DirectX 12, and Vulkan, strengthening functional coverage for next-generation Qualcomm GPUs.
- Maintained a legacy sanity-regression test list containing more than 4,000 tests across multiple GPU performance tiers and release milestones.
- Developed and maintained an image-comparison tool used by more than 50% of relevant test cases, reducing test-development effort by approximately 15% and lowering manual-analysis errors.
- Built more than 10 Python tools for image analysis, log analysis, regression triage, and workflow automation, improving engineering productivity and reducing human error.

## Other Professional & Open-Source Projects
**LuminError | Open-Source Vulkan Rendering & GPU Experimentation Framework** 
[https://github.com/gpuwangge/LuminError](https://github.com/gpuwangge/LuminError) | C++, Vulkan, GLSL (Oct 2025 – Present)  
- Engine Architecture: Engineered a modular C++ Vulkan rendering engine with 9 decoupled subsystems, directly managing GPU resources, synchronization, descriptor sets, and command submission for hybrid graphics/compute workloads.
- Rendering Pipeline: Implemented 35 example programs across graphics, compute, and ray-tracing pipelines, covering rasterization, shadow mapping, MSAA, PBR materials, glTF/GLB asset loading, and GEMM compute.
- Ray Tracing & Path Tracing: Built Vulkan KHR ray-tracing pipelines with BLAS/TLAS acceleration structures, Whitted-style ray tracing (Stanford Dragon, 870K triangles), and Monte Carlo path tracing with Next Event Estimation (Sponza scene, multi-material glTF validation).  

**Microsoft / Microsoft Research Asia** — Software Engineering & Research Intern  
Sunnyvale, CA | Beijing, China | 2008, 2009, 2013  
- Built automated data-quality pipelines (SQL Server, C) for the Windows 8 News app and developed multithreaded HCI prototypes for Microsoft Surface.

## Education
**Ph.D., Computer Science (Computer Graphics)**  Michigan State University  
East Lansing, MI, September 2009 – December 2017  
Dissertation: *Fluid Animation on Deforming Surface Meshes*. Research in geometric processing, fluid simulation, deformable materials, and 3D mesh representation.   
**B.S., Automation**  Beihang University, Beijing, China, September 2004 – July 2008  

## Publications
**Xiaojun Wang, Shiguang Liu, Yiying Tong.**  “Stain Formation on Deforming Inelastic Cloth.”  
*IEEE Transactions on Visualization and Computer Graphics.*  
DOI: 10.1109/TVCG.2017.2789203. Corpus ID: 51612072.  
**Ze Zhang, Xiaojun Wang, Yiying Tong.**  “Angle-Based Representation of Triangulated Surfaces.”  
*International Conference on Computer Graphics and Image Processing (CGIP), 2024.*

## Technical Skills
- Languages: C++/C, Python, SystemC, TLM, GLSL/HLSL
- Graphics & Compute: Vulkan, DirectX 11/12, OpenGL/ES, CUDA, OpenCL, Ray Tracing
- GPU Domains: Modeling, Driver Development, HW Verification, Parallel Computing, Performance Profiling (RenderDoc, PVRTune)
- AI & Infrastructure: AI Agents, AI Infrastructure, NPU Architecture Concepts
