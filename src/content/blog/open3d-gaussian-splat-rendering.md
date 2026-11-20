---
author: Obed Macallums
pubDatetime: 2026-11-20T02:21:53Z
title: "Rendering 3D Gaussian Splats in Open3D 0.20"
slug: open3d-gaussian-splat-rendering
featured: false
draft: false
tags:
  - open3d
  - gaussian-splatting
  - 3d
  - point-cloud
  - python
description: A hands-on guide to the Gaussian splat support added in Open3D 0.20. Load Gaussian PLY, SPLAT and SPZ files as tensor point clouds, inspect their attributes, transform and crop them safely, render them interactively and offscreen, and composite them with meshes and point clouds from Python.
---

Until recently, viewing a **3D Gaussian Splatting** scene from Python meant using a separate viewer, a web player or a custom CUDA rasterizer. **Open3D 0.20** changed that. Its renderer can now draw Gaussian splats in interactive windows and offscreen, in the same scene as regular PBR meshes and point clouds and with correct depth occlusion between them. This tutorial covers that workflow from start to finish: loading `.ply`, `.splat` and `.spz` files, inspecting what is inside them, transforming and cropping them without breaking their appearance, and rendering color and depth images for your own pipelines.

## Table of Contents

## What's New in Open3D 0.20

If you read [Gaussian Splatting vs. Traditional Photogrammetry](/posts/gaussian-splatting-vs-traditional-photogrammetry/), you know a trained 3DGS scene is just a big set of anisotropic Gaussians. Each one has a position, a covariance (scale and rotation), an opacity and spherical harmonics (SH) color coefficients. Open3D 0.20 treats that set as a first-class geometry:

- **Formats**: Gaussian splats are read into a tensor `PointCloud` from three formats: the **Gaussian PLY** produced by most training codebases, the binary **SPLAT** format popular on the web, and Niantic's compressed **SPZ** format. All three end up with the same in-memory layout.
- **Renderer**: a GPU compute pipeline projects, sorts and composites splats into the color image. Triangle meshes are still drawn by Filament, and splats are depth-occluded per splat against them.
- **Output**: anti-aliasing, depth output and offscreen rendering through `OffscreenRenderer` are supported.
- **Geometry operations**: `scale`, `rotate`, `translate` and `crop` are splat-aware, so they keep covariances and view-dependent color consistent.

This release adds **rendering only**, not training. You still train scenes with your usual tool (the reference 3DGS implementation, gsplat, Nerfstudio, Postshot and so on) and bring the result into Open3D.

## Requirements and Installation

Install Open3D 0.20 or newer from PyPI:

```bash
python -m venv .venv
source .venv/bin/activate
pip install "open3d>=0.20" numpy matplotlib requests
```

A few platform details matter for splats:

- Wheels are published for **Python 3.10 to 3.14**. macOS x86_64 builds were discontinued in this release, so on a Mac you need Apple Silicon.
- Splat rendering uses **Vulkan on Linux and Windows** and **Metal on macOS**. The OpenGL renderer can't draw splats, so you need an up-to-date GPU driver with Vulkan support.
- On a multi-GPU Linux or Windows machine, you can choose which Vulkan device Open3D uses with the standard Vulkan loader variables, such as `VK_LOADER_DRIVERS_SELECT` or `VK_LOADER_DEVICE_SELECT`. Set them before starting Python.

Check the installed version:

```python
import open3d as o3d

print(o3d.__version__)  # should print 0.20.0 or newer
```

## Getting Sample Data

You can use any scene you trained yourself. To follow along with public data, Niantic publishes SPZ samples in its [spz repository](https://github.com/nianticlabs/spz), and the official Open3D tutorial uses some of them. The helper below downloads one into a local folder:

```python
from pathlib import Path

import requests

DATA_DIR = Path("splat_data")
DATA_DIR.mkdir(exist_ok=True)

SAMPLES = {
    "hornedlizard.spz": (
        "https://raw.githubusercontent.com/nianticlabs/spz/main/"
        "samples/hornedlizard.spz"
    ),
}


def download(name: str, url: str) -> Path:
    dest = DATA_DIR / name
    if dest.is_file():
        return dest
    with requests.get(url, stream=True, timeout=30) as r:
        r.raise_for_status()
        with open(dest, "wb") as fh:
            for chunk in r.iter_content(chunk_size=1 << 20):
                fh.write(chunk)
    return dest


spz_path = download("hornedlizard.spz", SAMPLES["hornedlizard.spz"])
print(spz_path, spz_path.stat().st_size / 1e6, "MB")
```

The rest of the tutorial also uses a Gaussian PLY file called `scene.ply`. This is the `point_cloud.ply` that most 3DGS trainers write at the end of optimization.

## Detecting Splat Files

The `.ply` extension is ambiguous. As explained in [Understanding the .PLY Point Cloud Format](/posts/ply-format/), a PLY file can hold a mesh, a plain point cloud or, with the right vertex properties, Gaussian splats. Open3D 0.20 can inspect a file before loading it and report which kinds of geometry it contains:

```python
import open3d as o3d


def describe(path) -> list[str]:
    gtype = o3d.io.read_file_geometry_type(str(path))
    flags = []
    if gtype & o3d.io.CONTAINS_GAUSSIAN_SPLATS:
        flags.append("GAUSSIAN_SPLATS")
    if gtype & o3d.io.CONTAINS_TRIANGLES:
        flags.append("TRIANGLES")
    if gtype & o3d.io.CONTAINS_POINTS:
        flags.append("POINTS")
    if gtype & o3d.io.CONTAINS_LINES:
        flags.append("LINES")
    return flags


for path in ["splat_data/hornedlizard.spz", "scene.ply", "bunny.ply"]:
    print(f"{path:35s} {describe(path)}")
```

These flags are the basis of a generic loader. Splat files go through the tensor point cloud reader, meshes through `read_triangle_model`, and everything else is treated as a plain point cloud. We'll build that loader later.

## Loading and Inspecting Gaussian Splats

Gaussian splats are loaded with the **tensor** IO module, `o3d.t.io`, not the legacy `o3d.io.read_point_cloud`:

```python
pcd = o3d.t.io.read_point_cloud("splat_data/hornedlizard.spz")
print(pcd)
print("Number of splats:", len(pcd.point.positions))
```

The splat parameters are stored as named point attributes. The official tutorial lists `positions`, `rot`, `scale`, `opacity`, `f_dc` and `f_rest`:

| Attribute   | Meaning                                                            |
| ----------- | ------------------------------------------------------------------ |
| `positions` | Center (mean) of each Gaussian                                     |
| `rot`       | Orientation of each Gaussian as a quaternion                       |
| `scale`     | Extent of the Gaussian along its three local axes                  |
| `opacity`   | Per-splat opacity                                                  |
| `f_dc`      | Degree-0 spherical harmonics coefficients (the base color)         |
| `f_rest`    | Higher-order SH coefficients that encode view-dependent appearance |

Don't hard-code their shapes or ranges. Print them for the file you actually loaded:

```python
for key in ["positions", "rot", "scale", "opacity", "f_dc", "f_rest"]:
    if key not in pcd.point:
        print(f"{key:10s} (missing)")
        continue
    arr = pcd.point[key].numpy()
    print(
        f"{key:10s} shape={str(arr.shape):16s} dtype={arr.dtype} "
        f"min={arr.min():.3f} max={arr.max():.3f}"
    )
```

Two things are worth knowing about the formats:

- **SPLAT files only store base color.** They have no higher-order SH coefficients, so they don't support view-dependent effects such as specular highlights. A scene exported to `.splat` looks "flatter" than the same scene in PLY or SPZ.
- **SPZ is compressed but not lossless.** It quantizes the attributes to shrink files by roughly an order of magnitude compared to PLY. That is great for distribution, but keep the original PLY as your master copy.

Because a splat scene is just a tensor point cloud, you can use NumPy for quick analysis, such as computing the scene's bounds:

```python
import numpy as np

xyz = pcd.point.positions.numpy()
print("Min bound:", xyz.min(axis=0))
print("Max bound:", xyz.max(axis=0))
print("Center:   ", xyz.mean(axis=0))
```

## Rendering Splats Interactively

To render a tensor point cloud as splats, give it a `MaterialRecord` with the `gaussianSplat` shader and pass it to `o3d.visualization.draw` in dictionary form:

```python
import open3d as o3d

pcd = o3d.t.io.read_point_cloud("splat_data/hornedlizard.spz")

mat = o3d.visualization.rendering.MaterialRecord()
mat.shader = "gaussianSplat"

o3d.visualization.draw(
    [{"name": "lizard", "geometry": pcd, "material": mat}],
    title="Gaussian splats in Open3D",
    show_ui=True,
    show_skybox=False,
    bg_color=(0.0, 0.0, 0.0, 1.0),
)
```

Without the material, Open3D would draw the same object as ordinary colored points. The material is what switches it to the splat pipeline.

Inside a Jupyter notebook running on a remote machine, import the web visualizer's `draw` instead. It takes the same arguments:

```python
from open3d.web_visualizer import draw
```

### Anti-aliasing for Mip-Splatting Scenes

MSAA is disabled for splat views. Open3D offers a **Gaussian anti-aliasing** option instead, meant for scenes that were **trained with anti-aliasing**, such as those produced with Mip-Splatting. Turn it on through the material:

```python
mat = o3d.visualization.rendering.MaterialRecord()
mat.shader = "gaussianSplat"
mat.gaussian_splat_antialias = True
```

This applies a density compensation that cancels the brightening caused by the shader's low-pass filter. Don't enable it for scenes trained without anti-aliasing, because they will render too dark.

## Transforming and Cropping Splats Correctly

This is the most important practical detail in the release. In the [Point Cloud Registration with Open3D](/posts/point-cloud-registration-open3d/) guide we used `transform(T)` with a 4×4 matrix everywhere. **Don't do that with anisotropic splats.** A general affine matrix can't update Gaussian covariances and SH coefficients consistently, so the scene ends up with distorted shapes or wrong view-dependent color.

Use the splat-aware methods instead:

- `scale(s, center)` multiplies the linear scales by `|s|`. Only **isotropic** (uniform) scaling is supported.
- `rotate(R, center)` updates the quaternions and also rotates the higher-order SH coefficients, so reflections still point the right way.
- `translate(t)` moves the positions.

A small helper that applies a similarity transform in the order **Scale → Rotate → Translate** works for splats, meshes and point clouds alike, because all of them expose these three methods:

```python
import numpy as np
import open3d as o3d


def apply_srt(geometry, scale=1.0, rot_deg=(0.0, 0.0, 0.0), t=(0.0, 0.0, 0.0)):
    """Scale -> Rotate -> Translate in place. Euler angles are XYZ, in degrees."""
    R = o3d.geometry.get_rotation_matrix_from_xyz(
        np.deg2rad(np.asarray(rot_deg, dtype=np.float64))
    )
    center = np.zeros(3, dtype=np.float64)
    geometry.scale(float(scale), center)
    geometry.rotate(np.asarray(R), center)
    geometry.translate(np.asarray(t, dtype=np.float64))
    return geometry


pcd = o3d.t.io.read_point_cloud("scene.ply")
# Many 3DGS scenes come out of COLMAP upside down: flip them 180° around X
apply_srt(pcd, scale=1.0, rot_deg=(180.0, 0.0, 0.0))
```

If you got a 4×4 matrix from registration, decompose it into rotation and translation and pass those to `rotate` and `translate`. As long as the matrix is rigid (no shear, no non-uniform scale), that is equivalent to the full transform.

### Cropping a Region of Interest

Trained scenes often include floaters and distant background you don't care about. A tensor point cloud can be cropped with an axis-aligned bounding box. Splats outside the box are removed, and the retained splats keep all of their attributes unchanged:

```python
import open3d.core as o3c

aabb = o3d.t.geometry.AxisAlignedBoundingBox(
    o3c.Tensor([-1.5, -1.0, -1.5], o3c.float32),
    o3c.Tensor([1.5, 1.0, 1.5], o3c.float32),
)
cropped = pcd.crop(aabb)
print(f"{len(pcd.point.positions)} -> {len(cropped.point.positions)} splats")
```

Save the result with the tensor writer:

```python
o3d.t.io.write_point_cloud("scene_cropped.ply", cropped)
```

Writing back to Gaussian PLY is the safest choice, because every 3DGS tool can read it.

## Compositing Splats with Meshes and Point Clouds

The main strength of Open3D's splat support is that splats live in the **same scene** as everything else. Below is a small generic loader, built on the geometry-type flags, that returns entries ready for `draw`:

```python
import sys
from pathlib import Path

import open3d as o3d


def load_entry(path, name=None, **srt):
    path = Path(path)
    name = name or path.stem
    gtype = o3d.io.read_file_geometry_type(str(path))

    if gtype & o3d.io.CONTAINS_GAUSSIAN_SPLATS:
        geom = o3d.t.io.read_point_cloud(str(path))
        apply_srt(geom, **srt)
        mat = o3d.visualization.rendering.MaterialRecord()
        mat.shader = "gaussianSplat"
        return {"name": name, "geometry": geom, "material": mat}

    if gtype & o3d.io.CONTAINS_TRIANGLES:
        # glTF/OBJ models with materials and textures
        model = o3d.io.read_triangle_model(str(path))
        for mesh_info in model.meshes:
            apply_srt(mesh_info.mesh, **srt)
        return {"name": name, "geometry": model}

    if gtype & o3d.io.CONTAINS_POINTS:
        geom = o3d.io.read_point_cloud(str(path))
        apply_srt(geom, **srt)
        return {"name": name, "geometry": geom}

    print(f"Unsupported file: {path}", file=sys.stderr)
    return None
```

Now we can mix a splat scene, a textured glTF model, a LiDAR-style point cloud and some procedural geometry:

```python
import numpy as np

entries = [
    load_entry("scene.ply", name="garden", rot_deg=(180.0, 0.0, 0.0)),
    load_entry("vase.glb", name="vase", scale=0.5, t=(0.0, 0.2, 1.0)),
    load_entry("scan.ply", name="scan", t=(2.0, 0.0, 0.0)),
]
entries = [e for e in entries if e is not None]

# A procedural box with a lit material
box = o3d.geometry.TriangleMesh.create_box(0.3, 0.3, 0.3)
box.compute_vertex_normals()
box.paint_uniform_color([0.9, 0.3, 0.2])
box.translate([-0.5, 0.0, 0.5])
box_mat = o3d.visualization.rendering.MaterialRecord()
box_mat.shader = "defaultLit"
entries.append({"name": "box", "geometry": box, "material": box_mat})

# A coordinate frame helps you check orientation
frame = o3d.geometry.TriangleMesh.create_coordinate_frame(size=0.5)
entries.append({"name": "frame", "geometry": frame})

o3d.visualization.draw(
    entries,
    title="Splats + meshes + point clouds",
    show_ui=True,
    show_skybox=False,
    bg_color=(0.0, 0.0, 0.0, 1.0),
    ibl_intensity=100000,
)
```

The box and the glTF model are rasterized by Filament. The splats are composited by the compute pipeline and occluded per splat against the mesh depth, so the box correctly hides the splats behind it. This is useful for placing CAD or BIM models inside a captured environment, showing a LiDAR scan next to a splat reconstruction of the same site, or adding measurement guides.

## Offscreen Rendering: Color and Depth

For batch jobs, datasets and CI, you usually want images rather than a window. `OffscreenRenderer` renders the same mixed scenes to CPU images:

```python
import matplotlib.pyplot as plt
import numpy as np
import open3d as o3d

WIDTH, HEIGHT = 1280, 720
renderer = o3d.visualization.rendering.OffscreenRenderer(WIDTH, HEIGHT)
renderer.scene.set_background([0.0, 0.0, 0.0, 1.0])

for entry in entries:
    geom = entry["geometry"]
    if isinstance(geom, o3d.visualization.rendering.TriangleMeshModel):
        renderer.scene.add_model(entry["name"], geom)
    else:
        mat = entry.get("material", o3d.visualization.rendering.MaterialRecord())
        renderer.scene.add_geometry(entry["name"], geom, mat)

# Frame the scene: setup_camera(fov_deg, center, eye, up)
bounds = renderer.scene.bounding_box
center = bounds.get_center()
extent = np.linalg.norm(bounds.get_max_bound() - bounds.get_min_bound())
eye = center + np.array([0.0, -extent * 0.2, extent * 0.6])
renderer.setup_camera(60.0, center.tolist(), eye.tolist(), [0.0, -1.0, 0.0])

color = np.asarray(renderer.render_to_image())
depth = np.asarray(renderer.render_to_depth_image(z_in_view_space=True))

valid = depth[np.isfinite(depth) & (depth > 0)]
d_min, d_max = (valid.min(), valid.max()) if valid.size else (0.0, 1.0)

fig, axes = plt.subplots(1, 2, figsize=(16, 5))
axes[0].imshow(color)
axes[0].set_title("Color")
axes[0].axis("off")
im = axes[1].imshow(depth, cmap="plasma", vmin=d_min, vmax=d_max)
fig.colorbar(im, ax=axes[1], label="depth")
axes[1].set_title("Eye-space depth")
axes[1].axis("off")
plt.tight_layout()
plt.show()
```

With `z_in_view_space=True`, the depth image contains **linear eye-space depth** in scene units, which are meters if your scene is metric. Pixels with nothing rendered are not finite or positive, which is why the example masks them before computing the color range.

### Rendering a Turntable Sequence

The same renderer can produce an orbit around the scene, ready to be encoded into a video with `ffmpeg`:

```python
from pathlib import Path

out_dir = Path("turntable")
out_dir.mkdir(exist_ok=True)

radius = extent * 0.6
n_frames = 120
for i in range(n_frames):
    angle = 2.0 * np.pi * i / n_frames
    eye = center + np.array(
        [radius * np.sin(angle), -extent * 0.2, radius * np.cos(angle)]
    )
    renderer.setup_camera(60.0, center.tolist(), eye.tolist(), [0.0, -1.0, 0.0])
    img = renderer.render_to_image()
    o3d.io.write_image(str(out_dir / f"frame_{i:04d}.png"), img)
```

```bash
ffmpeg -framerate 30 -i turntable/frame_%04d.png -pix_fmt yuv420p turntable.mp4
```

Because you get both color and depth from the same camera, this loop is also a simple way to create synthetic RGB-D datasets from a splat reconstruction, for example to test depth-based algorithms or the TSDF integration pipeline in Open3D.

## Known Limitations and Tips

The release notes are explicit about the current rough edges:

- **No OpenGL**: splats need Vulkan (Linux/Windows) or Metal (macOS). If nothing shows up, check your Vulkan driver first.
- **Hardware issues**: some platforms can hang or crash when displaying splats. Windows machines with an Intel Arc A770m are a known case.
- **Tile limits**: very large splats, or splats very close to the camera, can exceed tile limits and be culled. Very dense scenes can drop tile entries once they reach capacity. Material capacity parameters can tune these limits. In practice, cropping floaters and keeping the camera a reasonable distance from the scene goes a long way.
- **No MSAA**: use `gaussian_splat_antialias` for anti-aliased scenes instead.
- **Transforms**: never apply a general 4×4 `transform()` to anisotropic splats. Stick to `scale`, `rotate` and `translate`.

Some practical tips:

- Crop before you render. Removing background splats reduces sorting and compositing work and helps avoid the tile limits.
- Keep a PLY master and use SPZ for distribution, since SPZ is lossy.
- When combining a splat scene with metric data such as LiDAR, remember that 3DGS scenes trained from COLMAP poses have arbitrary scale. Estimate a similarity transform first, for example from a few control points, then apply it with `apply_srt`.

## Conclusion

Open3D 0.20 turns Gaussian splats from a separate, viewer-only format into ordinary geometry that you can script. You can detect them with `read_file_geometry_type`, load PLY, SPLAT or SPZ files into a tensor `PointCloud`, transform and crop them with splat-aware methods, and render them next to meshes and point clouds with the `gaussianSplat` shader, either interactively or offscreen with color and depth output. For anyone already using Open3D for registration, reconstruction or LiDAR processing, that means splat reconstructions can join the same Python pipelines as the rest of the 3D data, without extra viewers or custom rasterizers.

## References

- [Open3D v0.20.0 release notes](https://github.com/isl-org/Open3D/releases/tag/v0.20.0)
- [Open3D 0.20 announcement: Gaussian splatting and full Intel GPU support](https://www.open3d.org/2026/09/24/open3d-v0-20-is-here-with-gaussian-splatting-and-full-intel-gpu-support/)
- [3D Gaussian Splatting in Open3D (official tutorial)](https://www.open3d.org/docs/latest/tutorial/visualization/gaussian_splatting.html)
- [Open3D v0.20 adds Gaussian splat rendering and SPZ support (Radiance Fields)](https://radiancefields.com/open3d-v0.20-adds-gaussian-splat-rendering-and-spz-support)
- [Niantic SPZ format and samples](https://github.com/nianticlabs/spz)
