---
author: Obed Macallums
pubDatetime: 2026-06-01T10:00:00Z
title: "Processing Point Clouds with PDAL"
slug: pdal-point-clouds
featured: false
draft: false
tags:
  - pdal
  - python
  - point-cloud
  - lidar
  - geospatial
  - las
description:
    A practical introduction to PDAL (Point Data Abstraction Library) for processing LiDAR point clouds. Learn the pipeline model, run filters from the command line and JSON, and integrate PDAL with Python and NumPy.
---

**PDAL** (Point Data Abstraction Library) is an open-source library for translating and processing point cloud data. If you already work with formats like **.LAS** and **.LAZ**, PDAL is the natural next step when you need to build repeatable processing workflows: reprojecting coordinates, removing outliers, classifying ground, or converting between dozens of formats. In this article we explore PDAL's pipeline model, its command-line tools, and how to use it from Python.

## Table of Contents

## What Is PDAL?

PDAL is a C++ library (with a command-line interface and Python bindings) designed around a simple but powerful idea: point cloud processing as a **pipeline** of stages. It supports reading and writing more than 30 formats — including LAS, LAZ, PLY, E57, COPC, and text — and provides a large catalog of filters for geometric and statistical operations.

It is often compared to two other tools you may already know:

- **laspy**: excellent for low-level reading and writing of LAS/LAZ files in Python, but not a processing framework.
- **Open3D**: focused on 3D geometry, registration, and visualization.

PDAL sits in a different spot: it specializes in **format translation** and **batch processing workflows**, making it the GDAL of the point cloud world.

## Installation

The recommended way to install PDAL together with its Python bindings is through Conda, which handles the native dependencies for you:

```bash
# Install PDAL and the Python bindings via conda-forge
conda install -c conda-forge pdal python-pdal
```

The Python bindings are also available on PyPI, but note that they require an existing PDAL installation on your system:

```bash
# Requires the PDAL native library to be installed first
pip install pdal
```

You can verify the installation and check the available stages with:

```bash
pdal --version
pdal --drivers   # List available readers, filters, and writers
```

## Core Concept: The Pipeline

Everything in PDAL revolves around the **pipeline**, an ordered sequence of stages through which points flow:

```text
readers  →  filters  →  writers
```

- **Readers** load points from a source (e.g. `readers.las`).
- **Filters** transform, classify, or remove points (e.g. `filters.reprojection`).
- **Writers** save the processed points to a destination (e.g. `writers.las`).

A pipeline is described declaratively in **JSON**. This makes workflows reproducible, version-controllable, and easy to share.

## Using the CLI

Before writing full pipelines, the command-line interface lets you inspect and convert files quickly.

Inspect metadata and the header of a file:

```bash
pdal info input.laz
```

Get summary statistics for each dimension:

```bash
pdal info --stats input.laz
```

Convert between formats — the `translate` command builds a simple pipeline for you:

```bash
# Convert LAZ to LAS
pdal translate input.laz output.las
```

Run a full pipeline defined in a JSON file:

```bash
pdal pipeline pipeline.json
```

## Pipelines in JSON

A pipeline can be as simple as a list of stage strings. The following pipeline reads a LAS file and writes it back out as compressed LAZ:

```json
{
  "pipeline": [
    "input.las",
    {
      "type": "writers.las",
      "filename": "output.laz",
      "compression": "laszip"
    }
  ]
}
```

When you need more control, each stage becomes an object with a `type` and its options. Here is a pipeline that reads a file, applies a filter, and writes the result:

```json
{
  "pipeline": [
    {
      "type": "readers.las",
      "filename": "input.laz"
    },
    {
      "type": "filters.range",
      "limits": "Classification[2:2]"
    },
    {
      "type": "writers.las",
      "filename": "ground_only.laz",
      "compression": "laszip"
    }
  ]
}
```

In this example, `filters.range` keeps only the points whose `Classification` value equals `2` (the ASPRS code for ground), producing a file that contains just the ground points.

## Common Filters

PDAL ships with a rich set of filters. These are some of the most useful in day-to-day point cloud work.

### Reprojection

Transform coordinates from one spatial reference system to another:

```json
{
  "type": "filters.reprojection",
  "in_srs": "EPSG:4326",
  "out_srs": "EPSG:32633"
}
```

### Cropping

Keep only the points inside a bounding box or polygon:

```json
{
  "type": "filters.crop",
  "bounds": "([500000, 500500], [4500000, 4500500])"
}
```

### Downsampling

Reduce point density by keeping one representative point per voxel — useful for speeding up downstream processing:

```json
{
  "type": "filters.voxelcenternearestneighbor",
  "cell": 1.0
}
```

### Outlier Removal

Detect and flag statistical outliers, then drop them:

```json
[
  {
    "type": "filters.outlier",
    "method": "statistical",
    "mean_k": 8,
    "multiplier": 2.0
  },
  {
    "type": "filters.range",
    "limits": "Classification![7:7]"
  }
]
```

The `filters.outlier` stage marks noise points with classification `7`; the following `filters.range` removes them.

### Ground Classification

Classify ground points using the Simple Morphological Filter (SMRF):

```json
{
  "type": "filters.smrf"
}
```

## Using PDAL from Python

The Python bindings let you execute pipelines programmatically and access the results as **NumPy structured arrays**, which makes PDAL a natural fit alongside the scientific Python stack.

```python
import pdal
import json

# Define the pipeline as a Python dict
pipeline_def = {
    "pipeline": [
        {
            "type": "readers.las",
            "filename": "input.laz"
        },
        {
            "type": "filters.outlier",
            "method": "statistical",
            "mean_k": 8,
            "multiplier": 2.0
        }
    ]
}

# Create and execute the pipeline
pipeline = pdal.Pipeline(json.dumps(pipeline_def))
count = pipeline.execute()
print(f"Processed {count} points")

# Access the results as NumPy arrays
arrays = pipeline.arrays
points = arrays[0]

print(f"Available dimensions: {points.dtype.names}")
print(f"First point X, Y, Z: {points['X'][0]}, {points['Y'][0]}, {points['Z'][0]}")
```

Because `points` is a structured NumPy array, you can compute statistics, filter dimensions, or feed the data into other libraries directly:

```python
import numpy as np

# Compute the elevation range
z = points["Z"]
print(f"Min Z: {z.min():.2f}, Max Z: {z.max():.2f}, Mean Z: {z.mean():.2f}")
```

## Practical Example

Let's combine several stages into a realistic end-to-end workflow. The following pipeline reads a LAZ file, reprojects it to a UTM zone, removes statistical outliers, classifies the ground, and writes the cleaned result back to LAZ:

```json
{
  "pipeline": [
    {
      "type": "readers.las",
      "filename": "raw_scan.laz"
    },
    {
      "type": "filters.reprojection",
      "in_srs": "EPSG:4326",
      "out_srs": "EPSG:32633"
    },
    {
      "type": "filters.outlier",
      "method": "statistical",
      "mean_k": 8,
      "multiplier": 2.0
    },
    {
      "type": "filters.range",
      "limits": "Classification![7:7]"
    },
    {
      "type": "filters.smrf"
    },
    {
      "type": "writers.las",
      "filename": "cleaned_scan.laz",
      "compression": "laszip"
    }
  ]
}
```

Run it with a single command:

```bash
pdal pipeline workflow.json
```

This kind of declarative workflow is exactly where PDAL shines: the same JSON file can be applied to hundreds of tiles, committed to version control, and shared with your team.

## Conclusion

PDAL provides a robust, format-agnostic framework for processing point cloud data. Its pipeline model turns complex LiDAR workflows into reproducible, declarative recipes, while the command-line tools make quick inspection and conversion effortless. When combined with the Python bindings and NumPy, PDAL integrates seamlessly into data analysis pipelines.

If you only need to read and write LAS/LAZ files, a lighter library like `laspy` may be enough; for 3D geometry and registration, `Open3D` is a better fit. But when your task is **translating formats** or **building repeatable processing workflows** at scale, PDAL is the tool of choice for working with LiDAR and point cloud data.
