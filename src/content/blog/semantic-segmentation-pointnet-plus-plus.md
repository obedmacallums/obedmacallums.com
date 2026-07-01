---
author: Obed Macallums
pubDatetime: 2026-06-28T12:30:00Z
title: "Semantic Segmentation of Point Clouds with PointNet++"
slug: semantic-segmentation-pointnet-plus-plus
featured: false
draft: false
tags:
  - python
  - pytorch
  - deep-learning
  - point-cloud
  - pointnet
  - semantic-segmentation
  - lidar
  - 3d
description:
  A practical guide to semantic segmentation of 3D point clouds with PointNet++. Learn why point clouds are hard for neural networks, how PointNet++ builds a hierarchical feature representation with sampling and grouping, and how to implement the key building blocks in PyTorch, with a worked example on the S3DIS indoor dataset.
---

Point clouds are everywhere in 3D computer vision, from LiDAR scans of cities to indoor reconstructions and robotic perception. But unlike images, point clouds are **unordered, irregular, and sparse**, which makes them a poor fit for standard convolutional networks. **PointNet++** is a landmark architecture that learns directly on raw points and remains a strong, intuitive baseline for **semantic segmentation**, the task of assigning a class label to every single point. In this guide we explore why point clouds are challenging, how PointNet++ solves the problem hierarchically, and how to implement its core building blocks in PyTorch.

## Table of Contents

## Why Point Clouds Are Hard

A point cloud is simply a set of points, each with coordinates `(x, y, z)` and optionally extra features such as color or intensity. Three properties make them difficult for neural networks:

- **Unordered**: a point cloud with `N` points has `N!` valid orderings. The network output must be **permutation invariant**, the same result regardless of how the points are listed.
- **Irregular and non-uniform density**: points are not on a grid. Some regions are dense, others sparse, so there is no fixed neighborhood like a pixel's 3x3 window.
- **Continuous geometry**: meaningful structure lives in the spatial relationships between nearby points, not in any index ordering.

Voxel-based methods solve this by quantizing space into a 3D grid so 3D convolutions apply, but they waste memory on empty space and lose fine detail. Projection-based methods render the cloud to 2D images, but discard true 3D structure. **PointNet** and **PointNet++** instead operate directly on the raw points.

## From PointNet to PointNet++

### The PointNet Idea

PointNet achieves permutation invariance with an elegant trick. Each point is passed independently through a shared **multi-layer perceptron (MLP)** to produce a per-point feature vector. Then a **symmetric function**, a global max-pooling, aggregates all point features into a single global descriptor:

```text
global_feature = MAX( MLP(p_1), MLP(p_2), ..., MLP(p_N) )
```

Because `max` is symmetric, reordering the points does not change the result. This works well for classifying whole objects, but it has a key weakness: it captures only a **global** representation and **local** geometric detail. There is no notion of local neighborhoods, so fine-grained structure is lost.

### The PointNet++ Improvement

PointNet++ fixes this by applying PointNet **hierarchically and locally**, much like a CNN builds features over growing receptive fields. The core idea is the **Set Abstraction (SA)** module, which progressively groups points into local regions, summarizes each region with a mini-PointNet, and produces a smaller set of points with richer features.

A Set Abstraction layer has three steps:

1. **Sampling**: select a subset of points as region centroids using **Farthest Point Sampling (FPS)**, which spreads centroids evenly across the cloud.
2. **Grouping**: for each centroid, gather its neighbors with a **ball query** (all points within a radius) to form local patches.
3. **PointNet encoding**: apply a shared MLP plus max-pooling to each local patch, producing one feature vector per centroid.

Stacking several SA layers builds a multi-scale hierarchy: early layers capture fine local detail, deeper layers capture broader context.

## Architecture for Semantic Segmentation

Classification only needs the encoder (the SA layers) followed by a global descriptor. **Semantic segmentation** is different: we need a label for **every original point**, so we must propagate the learned features back to full resolution.

PointNet++ uses an **encoder-decoder** design, similar in spirit to U-Net:

- **Encoder**: a stack of Set Abstraction layers that progressively downsample the cloud while enriching features.
- **Decoder**: a stack of **Feature Propagation (FP)** layers that upsample features back to the original points using distance-weighted interpolation, with **skip connections** from the matching encoder level.

The final per-point features pass through a small MLP and a softmax to produce class probabilities for each point.

## Implementing the Core Blocks in PyTorch

Below we implement the essential operations. These are simplified reference versions written for clarity rather than maximum performance; production code typically uses optimized CUDA kernels.

### Farthest Point Sampling

FPS iteratively picks the point farthest from the already-selected set, ensuring even coverage of the cloud.

```python
import torch

def farthest_point_sample(xyz, n_samples):
    """
    xyz: (B, N, 3) input coordinates
    returns: (B, n_samples) indices of sampled points
    """
    B, N, _ = xyz.shape
    device = xyz.device

    centroids = torch.zeros(B, n_samples, dtype=torch.long, device=device)
    distance = torch.full((B, N), 1e10, device=device)
    farthest = torch.randint(0, N, (B,), dtype=torch.long, device=device)
    batch_idx = torch.arange(B, dtype=torch.long, device=device)

    for i in range(n_samples):
        centroids[:, i] = farthest
        centroid = xyz[batch_idx, farthest, :].view(B, 1, 3)
        dist = torch.sum((xyz - centroid) ** 2, dim=-1)
        distance = torch.minimum(distance, dist)
        farthest = torch.max(distance, dim=-1)[1]

    return centroids
```

### Ball Query Grouping

For each centroid, gather up to `n_sample` neighbors within `radius`.

```python
def ball_query(radius, n_sample, xyz, new_xyz):
    """
    xyz:     (B, N, 3) all points
    new_xyz: (B, S, 3) centroids
    returns: (B, S, n_sample) neighbor indices
    """
    B, N, _ = xyz.shape
    S = new_xyz.shape[1]
    device = xyz.device

    # Squared distance between every centroid and every point
    dist = torch.cdist(new_xyz, xyz) ** 2  # (B, S, N)

    group_idx = torch.arange(N, device=device).view(1, 1, N).repeat(B, S, 1)
    group_idx[dist > radius ** 2] = N  # mark out-of-range points

    group_idx = group_idx.sort(dim=-1)[0][:, :, :n_sample]

    # Replace invalid entries (== N) with the nearest valid neighbor
    first = group_idx[:, :, 0:1].repeat(1, 1, n_sample)
    mask = group_idx == N
    group_idx[mask] = first[mask]

    return group_idx
```

### The Set Abstraction Module

This ties sampling, grouping, and the shared MLP together.

```python
import torch.nn as nn
import torch.nn.functional as F

def index_points(points, idx):
    """Gather points by index. points: (B, N, C), idx: (B, S, K) or (B, S)."""
    B = points.shape[0]
    view_shape = [B] + [1] * (idx.dim() - 1)
    batch_idx = torch.arange(B, device=points.device).view(view_shape)
    batch_idx = batch_idx.expand(idx.shape)
    return points[batch_idx, idx, :]

class SetAbstraction(nn.Module):
    def __init__(self, n_points, radius, n_sample, in_channel, mlp):
        super().__init__()
        self.n_points = n_points
        self.radius = radius
        self.n_sample = n_sample

        layers = []
        last_channel = in_channel + 3  # +3 for relative xyz
        for out_channel in mlp:
            layers.append(nn.Conv2d(last_channel, out_channel, 1))
            layers.append(nn.BatchNorm2d(out_channel))
            layers.append(nn.ReLU())
            last_channel = out_channel
        self.mlp = nn.Sequential(*layers)

    def forward(self, xyz, features):
        """
        xyz:      (B, N, 3)
        features: (B, N, C) or None
        returns:  new_xyz (B, S, 3), new_features (B, S, mlp[-1])
        """
        idx = farthest_point_sample(xyz, self.n_points)
        new_xyz = index_points(xyz, idx)                     # (B, S, 3)

        group_idx = ball_query(self.radius, self.n_sample, xyz, new_xyz)
        grouped_xyz = index_points(xyz, group_idx)           # (B, S, K, 3)
        grouped_xyz -= new_xyz.unsqueeze(2)                  # relative coords

        if features is not None:
            grouped_feat = index_points(features, group_idx)
            grouped = torch.cat([grouped_xyz, grouped_feat], dim=-1)
        else:
            grouped = grouped_xyz

        # (B, C, K, S) for Conv2d, then max over the neighbor dimension
        grouped = grouped.permute(0, 3, 2, 1)
        new_features = self.mlp(grouped)
        new_features = torch.max(new_features, dim=2)[0]     # (B, mlp[-1], S)
        new_features = new_features.permute(0, 2, 1)         # (B, S, mlp[-1])

        return new_xyz, new_features
```

### Feature Propagation (Decoder)

The decoder upsamples features back to denser point sets using inverse-distance-weighted interpolation from the three nearest neighbors, then concatenates skip features.

```python
class FeaturePropagation(nn.Module):
    def __init__(self, in_channel, mlp):
        super().__init__()
        layers = []
        last_channel = in_channel
        for out_channel in mlp:
            layers.append(nn.Conv1d(last_channel, out_channel, 1))
            layers.append(nn.BatchNorm1d(out_channel))
            layers.append(nn.ReLU())
            last_channel = out_channel
        self.mlp = nn.Sequential(*layers)

    def forward(self, xyz1, xyz2, feat1, feat2):
        """
        xyz1: (B, N, 3) dense points to upsample to
        xyz2: (B, S, 3) sparse points with features
        feat1: (B, N, C1) skip features from encoder (or None)
        feat2: (B, S, C2) features to propagate
        """
        dist = torch.cdist(xyz1, xyz2)                       # (B, N, S)
        dist, idx = dist.sort(dim=-1)
        dist, idx = dist[:, :, :3], idx[:, :, :3]            # 3 nearest

        weight = 1.0 / (dist + 1e-8)
        weight = weight / weight.sum(dim=-1, keepdim=True)   # (B, N, 3)

        interp = torch.sum(
            index_points(feat2, idx) * weight.unsqueeze(-1), dim=2
        )                                                    # (B, N, C2)

        if feat1 is not None:
            new_feat = torch.cat([feat1, interp], dim=-1)
        else:
            new_feat = interp

        new_feat = new_feat.permute(0, 2, 1)                 # (B, C, N)
        new_feat = self.mlp(new_feat)
        return new_feat.permute(0, 2, 1)                     # (B, N, mlp[-1])
```

## A Segmentation Model and Training Loop

We assemble a compact encoder-decoder for the **S3DIS** dataset (Stanford 3D Indoor Spaces), which labels indoor points into 13 classes such as floor, wall, chair, table, and door.

```python
class PointNet2Seg(nn.Module):
    def __init__(self, num_classes=13):
        super().__init__()
        # Encoder
        self.sa1 = SetAbstraction(1024, 0.1, 32, in_channel=6, mlp=[32, 32, 64])
        self.sa2 = SetAbstraction(256, 0.2, 32, in_channel=64, mlp=[64, 64, 128])
        self.sa3 = SetAbstraction(64, 0.4, 32, in_channel=128, mlp=[128, 128, 256])
        # Decoder
        self.fp3 = FeaturePropagation(256 + 128, [256, 128])
        self.fp2 = FeaturePropagation(128 + 64, [128, 64])
        self.fp1 = FeaturePropagation(64, [64, 64])
        # Head
        self.head = nn.Sequential(
            nn.Conv1d(64, 64, 1), nn.BatchNorm1d(64), nn.ReLU(),
            nn.Dropout(0.5),
            nn.Conv1d(64, num_classes, 1),
        )

    def forward(self, xyz, features):
        l1_xyz, l1_feat = self.sa1(xyz, features)
        l2_xyz, l2_feat = self.sa2(l1_xyz, l1_feat)
        l3_xyz, l3_feat = self.sa3(l2_xyz, l2_feat)

        l2_feat = self.fp3(l2_xyz, l3_xyz, l2_feat, l3_feat)
        l1_feat = self.fp2(l1_xyz, l2_xyz, l1_feat, l2_feat)
        l0_feat = self.fp1(xyz, l1_xyz, None, l1_feat)

        logits = self.head(l0_feat.permute(0, 2, 1))         # (B, C, N)
        return logits.permute(0, 2, 1)                       # (B, N, C)
```

S3DIS points typically carry `(x, y, z, r, g, b)`, so `features` holds the RGB plus normalized coordinates (6 channels here). A standard training loop uses per-point cross-entropy:

```python
import torch.optim as optim

model = PointNet2Seg(num_classes=13).cuda()
optimizer = optim.Adam(model.parameters(), lr=1e-3, weight_decay=1e-4)
criterion = nn.CrossEntropyLoss()

for epoch in range(num_epochs):
    model.train()
    for xyz, feats, labels in train_loader:        # labels: (B, N)
        xyz, feats, labels = xyz.cuda(), feats.cuda(), labels.cuda()

        logits = model(xyz, feats)                  # (B, N, C)
        loss = criterion(logits.reshape(-1, 13), labels.reshape(-1))

        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
```

> Tip: S3DIS scenes are large, so the standard practice is to split rooms into overlapping blocks (for example 1m x 1m columns) and randomly sample a fixed number of points (such as 4096) per block during training.

## Evaluating Segmentation Quality

The standard metric for semantic segmentation is **mean Intersection over Union (mIoU)**, averaged across classes. For each class:

```text
IoU = TP / (TP + FP + FN)
```

Where TP, FP, and FN are true positives, false positives, and false negatives counted over all points. Averaging IoU across the 13 classes gives mIoU, which handles class imbalance far better than raw point accuracy, since common classes like floor and ceiling would otherwise dominate.

```python
def compute_iou(preds, labels, num_classes):
    ious = []
    preds = preds.reshape(-1)
    labels = labels.reshape(-1)
    for c in range(num_classes):
        pred_c = preds == c
        label_c = labels == c
        intersection = (pred_c & label_c).sum().item()
        union = (pred_c | label_c).sum().item()
        if union > 0:
            ious.append(intersection / union)
    return sum(ious) / len(ious) if ious else 0.0
```

## Practical Tips and Limitations

- **Normalize coordinates** per block (center and scale) so the network sees consistent ranges.
- **Data augmentation** helps a lot: random rotation around the vertical axis, jitter, and random scaling.
- **Class imbalance**: consider weighted cross-entropy when rare classes (like clutter or board) underperform.
- **Performance**: the reference FPS and ball-query code above is `O(N^2)` and fine for teaching, but use optimized libraries for real workloads.
- **Beyond PointNet++**: newer architectures such as **KPConv**, **Point Transformer**, and sparse-convolution networks like **MinkowskiNet** push accuracy higher, but PointNet++ remains the clearest entry point for understanding point-based learning.

## Conclusion

PointNet++ elegantly extends the permutation-invariant insight of PointNet into a hierarchical, local-to-global feature learner that works directly on raw point clouds. For **semantic segmentation**, its encoder-decoder design with Set Abstraction and Feature Propagation produces a label for every point while respecting the irregular, unordered nature of 3D data. By implementing sampling, grouping, and propagation from scratch in PyTorch, you gain an intuition that transfers to virtually every modern point-based architecture. Whether you are parsing indoor scenes with S3DIS, classifying LiDAR sweeps, or building robotic perception systems, PointNet++ is an essential and approachable foundation for deep learning on 3D point clouds.
