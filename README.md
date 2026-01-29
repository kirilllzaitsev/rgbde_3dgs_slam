# SfM-free 3DGS-SLAM with RGBD and event sensors

## Introduction

Standard SfM-free 3DGS SLAM breaks under blur and low light conditions. To address this, recent research focuses on leveraging the event sensor that is largely blur-free and gives meaningful signal in scenes with low light. However, a downside of state-of-the-art in this line of work is implicit camera trajectory optimization via RGBD reconstruction losses. When the viewpoints become sparse (e.g., due to high resolution of RGBD frames that reduces camera's bandwidth), the mapping performance is strong, while camera tracking becomes the bottleneck. Sparse views break the common assumption of marginal difference in camera poses between subsequent steps.

This work explores possible ways for dealing with the problem of sparsity in 3DGS-SLAM from RGBD and event data. Assuming that RGBD frames are sparse while events are temporally dense by definition, we propose to extract an update the camera pose from the event stream. The main insight is that relative pose priors (for example, from an optical flow) significantly improve the tracking performance in sparse viepoint settings, with up to ~2.5x improvement compared to a SOTA baseline. EGS-SLAM [1] is taken as a SOTA RGBD+E pipeline for 3DGS-SLAM.

## System overview

The diagram below shows the overall system architecture:

![System Overview](docs/pipeline.png)

The main addition to EGS-SLAM is the event-based pose tracker that estimates relative camera poses between two timestamps. The estimated relative poses are then used as priors in the pose optimization step of EGS-SLAM.

## References

1. [EGS-SLAM: RGB-D Gaussian Splatting SLAM with Events](https://arxiv.org/abs/2508.07003)