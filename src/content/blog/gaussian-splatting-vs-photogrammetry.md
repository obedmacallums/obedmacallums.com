---
author: Obed Macallums
pubDatetime: 2026-02-18T10:00:00Z
title: "Gaussian Splatting vs. Traditional Photogrammetry"
slug: gaussian-splatting-vs-traditional-photogrammetry
featured: false
draft: false
tags:
  - 3d
  - gaussian-splatting
  - photogrammetry
  - point-cloud
  - computer-vision
description:
  A technical comparison between 3D Gaussian Splatting and traditional photogrammetry. Both techniques reconstruct 3D scenes from photographs, but differ fundamentally in their approach, strengths, and ideal use cases.
---

Both techniques share the same goal: reconstructing a 3D representation of a scene from photographs. The difference lies in how they achieve it.

## Table of Contents

## Traditional Photogrammetry

Traditional photogrammetry is built on a well-defined multi-stage pipeline. The process begins with **Structure from Motion (SfM)**, which analyzes a set of overlapping photographs to identify common feature points across images. By tracking how these features shift between viewpoints, the algorithm simultaneously estimates the 3D position of each feature and the position and orientation of each camera. The output is a **sparse point cloud** — a rough skeleton of the scene geometry along with calibrated camera parameters.

From there, **Multi-View Stereo (MVS)** takes over. Using the calibrated cameras, it performs dense matching across all image pairs to produce a much denser point cloud that captures fine surface detail. This dense cloud is then converted into a **polygonal mesh** through surface reconstruction algorithms like Poisson reconstruction or Delaunay triangulation. Finally, the original photographs are projected onto the mesh to generate a photorealistic **texture**, completing the reconstruction.

This pipeline is robust and reproducible, and its outputs are directly usable in industry-standard tools. However, it comes with notable limitations:

- **Processing time**: densification and mesh generation are computationally expensive, especially for large outdoor scenes or high-resolution captures. Hours-long processing runs are common.
- **Reflective surfaces**: specular materials break the photometric consistency assumptions that MVS relies on, producing holes or noise in those areas.
- **Fine or translucent elements**: thin structures like vegetation, power lines, or glass lack sufficient texture for reliable feature matching, making them difficult or impossible to reconstruct faithfully.

Despite these challenges, photogrammetry is a mature technology backed by decades of development, with broad software support (Agisoft Metashape, RealityCapture, OpenMVG/OpenMVS, COLMAP) and a large user base across surveying, architecture, archaeology, and engineering.

## 3D Gaussian Splatting

**3D Gaussian Splatting (3DGS)**, introduced by Kerbl et al. in 2023, takes a fundamentally different approach. Instead of building an explicit mesh, it represents the scene as a collection of millions of semi-transparent 3D Gaussians — volumetric primitives, each defined by a small set of learnable parameters:

- **Position** (mean): where the Gaussian is located in 3D space.
- **Covariance** (scale + rotation): the shape and orientation of the ellipsoid, allowing Gaussians to stretch along surfaces or fill volumetric regions.
- **Opacity**: how transparent or opaque the Gaussian is.
- **Color**: encoded as **spherical harmonics** coefficients, enabling view-dependent appearance — the same Gaussian can look different depending on the viewing angle, naturally capturing specular highlights and iridescence.

The method typically initializes from a point cloud, which is most commonly a sparse SfM output but can also come from a **LiDAR sensor**. Using LiDAR as initialization provides a denser and more geometrically accurate starting point, which can accelerate convergence and improve reconstruction quality in areas that are difficult to reconstruct from images alone. After initialization, the method runs an iterative **gradient-based optimization** loop. At each step, all Gaussians are rasterized into a 2D image from a known training viewpoint using a differentiable tile-based renderer. The pixel-level difference between this rendered image and the original photograph is used to compute gradients, which update every Gaussian parameter simultaneously via backpropagation.

During training, an **adaptive density control** mechanism periodically splits Gaussians that cover large regions (to add detail) and prunes Gaussians with near-zero opacity (to reduce redundancy). This keeps the representation both accurate and efficient.

The key consequence of this design is **real-time novel view synthesis**: once trained, the scene can be rendered at dozens of frames per second by rasterizing the sorted Gaussians with alpha compositing — no ray marching, no neural network inference at render time. This makes 3DGS particularly attractive for interactive visualization, virtual production, and immersive experiences. It also handles complex visual phenomena like reflections and semi-transparency more gracefully than mesh-based texturing, since opacity and view-dependent color are first-class properties of the representation.

### Storage and Delivery Formats

One practical challenge with 3DGS is file size: a scene with millions of Gaussians stored as a standard `.ply` file can easily reach hundreds of megabytes. This has driven a wave of compressed formats designed for efficient storage and web delivery:

- **Compressed PLY**: a simple quantized variant of the standard format, offering modest size reduction.
- **SPZ** (open-sourced by Niantic): a more aggressive binary format, roughly 10× smaller than PLY.
- **SOG** (Spatially Ordered Gaussians, by PlayCanvas): encodes Gaussian parameters as WebP images inside a ZIP archive, achieving 15–20× compression. Designed specifically for web delivery.

The ecosystem is moving toward standardization: in 2025, Khronos and OGC announced a proposal to integrate Gaussian Splats into the **glTF** standard, which would make them a first-class interoperable 3D asset format across engines and tools.

### Beyond Static Scenes

The technique has also expanded beyond static reconstruction. **4D Gaussian Splatting** extends the representation to handle dynamic scenes — each Gaussian can change its parameters over time, enabling capture of moving subjects at real-time frame rates. This opens the door to applications in sports broadcast, volumetric video, and digital doubles for film production.

## Geometric Precision

This is where photogrammetry still holds a clear advantage.

Photogrammetry builds geometry through **triangulation** — a mathematically grounded process tied to the actual physical geometry of the scene. The resulting meshes and point clouds carry metric accuracy that can be directly used for measurement, CAD export, or integration into GIS/BIM workflows.

3DGS optimizes for **visual fidelity**, not geometric accuracy. The Gaussians are positioned to minimize rendering error from known viewpoints, which does not guarantee that they correspond precisely to actual surface geometry. This makes 3DGS less suitable for applications where precise measurements or clean geometry are required.

| Criterion | Photogrammetry | 3D Gaussian Splatting |
| --- | --- | --- |
| Geometric precision | High | Moderate |
| Real-time rendering | No | Yes |
| Reflective surfaces | Difficult | Better handled |
| Fine/translucent details | Difficult | Better handled |
| CAD/GIS/BIM integration | Native | Limited |
| Processing time | Long | Faster |

## Recent Advances in 3DGS

Since its introduction in 2023, 3DGS has evolved at a fast pace across several fronts.

**Geometry extraction.** One of the original method's main weaknesses was the lack of clean, usable geometry. Two notable papers address this directly. **2DGS** (SIGGRAPH 2024) replaces 3D volumetric Gaussians with flat 2D disks that align naturally with surfaces, enabling accurate mesh extraction with much less noise. **SuGaR** (CVPR 2024) keeps standard 3D Gaussians but adds a regularization term that forces them to align with the scene surface during training, then extracts a mesh via Poisson reconstruction and optionally binds new Gaussians to the triangles for high-quality rendering — making the scene editable and animatable.

**Large-scale rendering.** **FlashGS** (CVPR 2025) reworked the rasterization pipeline to sustain real-time performance on large outdoor scenes at 4K resolution. **Voyager** takes a different angle: a cloud-client architecture that streams only the Gaussians visible from the current viewpoint, achieving over 100× data reduction for city-scale mobile experiences.

**Integration with generative models.** Methods like **PSHuman** and **PARTE** combine 3DGS with diffusion models to infer occluded regions and improve geometric detail, particularly useful for human reconstruction where full coverage from photos is rarely possible.

**Standardization.** In August 2025, Khronos and OGC announced a proposal to add Gaussian Splats to the **glTF** ecosystem, signaling the transition from research artifact to production-ready interchange format.

## What Comes Next?

Will we soon reach a single method that delivers both visual quality and geometric precision simultaneously?

The trajectory suggests convergence, but we are not there yet. For now, the choice between techniques depends on the application:

- **Measurement, engineering, or GIS workflows** — photogrammetry remains the right tool.
- **Visualization, virtual production, or real-time rendering** — 3DGS offers a compelling alternative.

The most interesting developments may come from hybrid pipelines that use photogrammetry's geometric rigor as an initialization or constraint for Gaussian-based optimization.

## Conclusion

Gaussian Splatting and traditional photogrammetry are complementary rather than competing technologies. Each excels where the other struggles. Understanding their fundamental differences — triangulation-based geometry versus visually-optimized Gaussian representations — is essential for choosing the right approach and for anticipating where the field is heading.
