# SfM-free 3DGS-SLAM with RGBD and event sensors

## Introduction

Standard SfM-free 3DGS SLAM breaks under blur and low light conditions. To address this, recent research focuses on leveraging the event sensor that is largely blur-free and gives meaningful signal in scenes with low light. However, a downside of state-of-the-art in this line of work is implicit camera trajectory optimization via RGBD reconstruction losses. When the viewpoints become sparse (e.g., due to high resolution of RGBD frames that reduces camera's bandwidth), the mapping performance is strong, while camera tracking becomes the bottleneck. Sparse views break the common assumption of marginal difference in camera poses between subsequent steps.

This work explores possible ways for dealing with the problem of sparsity in 3DGS-SLAM from RGBD and event data. Assuming that RGBD frames are sparse while events are temporally dense by definition, we propose to extract an update the camera pose from the event stream. The main insight is that relative pose priors (for example, from an optical flow) significantly improve the tracking performance in sparse viepoint settings, with up to ~2.5x improvement compared to a SOTA baseline. EGS-SLAM [1] is taken as a SOTA RGBD+E baseline for 3DGS-SLAM.

## System overview

The diagram below shows a high-level system:

![System Overview](docs/pipeline.png)

At each timestep, the system receives a blurry RGBD frame and a stream of events. These inputs are used for optimizing 3D Gaussian Splatting (3DGS) to track the camera pose (tracking stage) and reconstruct the scene (mapping stage). The output is an updated 3DGS map and the camera pose at the current timestep. Our work contributes to improving the tracking stage. Note that the mapping stage also refines the camera pose after tracking [2].

The tracking stage is structured as follows:

![Tracking stage](docs/optim_stage.png)

where camera pose at timestep $t$ is denoted as $P_t$, event interval is between timesteps $\tau_{start}$ and $\tau_{end}=t$, batch of events from the interval is $\tau_{k}$. The mapping stage has almost the same structure, working without our pose tracking module.

The pipeline processes an incoming blurry RGB frame at timestep $t$ as follows:

1. The camera pose is propagated from the previous timestep $P_{t-1}$ to the current timestep $P_t$ using an event-based pose tracker that operates on event batches to estimate relative camera poses.

2. The propagated pose $P_t$ is then refined using EGS-SLAM's pose optimization module, which utilizes RGBD and event data to optimize the camera pose. The approach uses three reconstruction losses:

    - Blurry RGB reconstruction loss: 3DGS renders several sharp latent images during the camera exposure interval when the blurry RGB frame is captured. These latent images are averaged to form the blurry image, which is then compared to the input RGB frame.
    - Depth reconstruction loss: ensures that the rendered depth map from the 3DGS map aligns with the input depth frame.
    - Brightness change loss: reconstructs the changes in brightness between intensity images obtained from events and rendered latent images.

The main addition to EGS-SLAM is the event-based pose tracker that estimates relative camera poses between two timestamps before the tracking stage. Leveraging this prior on $P_t$, the pose optimization step of EGS-SLAM refined it during the tracking and mapping stages, removing the accumulated drift from the event-based tracker.

## Experimental setup & baselines

We build our approach on top of EGS-SLAM. We implement the event-based pose tracker with several methods: contrast maximization [6], pretrained models for event-based optical flow or point tracking, or a foundational model for RGB optical flow [3] distilled to event inputs following [7]. To test the motion prior efficacy, we created an RGB point detection and matching baseline based on SuperPoint and Lightglue [8].

We introduce sparsity by uniformly subsampling frames, picking every 4-/8-/16-th frame of the dataset.

### Evaluation

In the dense setup, we evaluate results following the approach of [1], computing the tracking and reconstruction metrics on non-keyframe frames which are dynamically selected during optimization.

In the sparse setup, unseen frames for evaluation are uniformly selected in between the frames used for optimization. We follow the evaluation approach of [5], where the pose for a given eval frame is interpolated from the two adjacent frames and optimized based on the reconstruction losses.

### Metrics

Pose/Tracking: ATE on camera trajectories (without scale correction).

Reconstruction: PSNR/SSIM/LPIPS on rendered views.

Note that the two sets of metrics are not directly correlated, as good reconstruction can be achieved even with poor camera tracking, since views for reconstruction are rendered at estimated rather than groundtruth (GT) poses.

### Datasets

We evaluate our method using the datasets of [1]:

- EventReplica: synthetic data; seven indoor scenes with 400 frames on average; low/medium motion blur; events generated via ESIM [4].

- DEVD: real data; eight indoor scenes with 365 frames on average; medium/high motion blur.

## Where SOTA fails

TBD

## Adding relative pose prior

TBD

## Secondary experiments & side insights

TBD

## Limitations

TBD

## Takeaways

- 3DGS-SLAM on blurry data works well when augmented with event priors such as brightness loss
- camera tracking is negatively affected by frame sparsity, while mapping remains sufficiently high quality
- the reason for the tracking failure is the assumption on small delta poses that allows to implicitly optimize them via rendering losses
- relative pose priors improve sparse-view tracking performance by up to 2.5x while occasionally displaying instability on dense views
  - additional filtering based on correspondences improves pose reliability, but has to be configured manually and is not scalable
- dense motion cues (RGB flow, event flow) are more reliable and robust than sparse matches on textureless scenes

What didn't work:

- explicit reprojection-based pose supervision fails without sufficient high-quality correspondences
- learning-based dense flow from events is promising but currently bottlenecked by generalization and data scarcity
  - contrast maximization for optical flow lags behind deep learning approaches in both quality and efficiency
- distilling a foundational RGB flow model [3] to event inputs on EventReplica/DEVD datasets [1] lacks data to generalize well

## References

1. [EGS-SLAM: RGB-D Gaussian Splatting SLAM with Events](https://arxiv.org/abs/2508.07003)
2. [BAD-Gaussians: Bundle Adjusted Deblur Gaussian Splatting](https://arxiv.org/html/2403.11831v2)
3. [Unifying Flow, Stereo and Depth Estimation](https://github.com/autonomousvision/unimatch)
4. [ESIM: an Open Event Camera Simulator](https://rpg.ifi.uzh.ch/esim.html)
5. [InstantSplat: Sparse-view SfM-free Gaussian Splatting in Seconds](https://arxiv.org/abs/2403.20309)
6. [Secrets of Event-Based Optical Flow](https://arxiv.org/abs/2207.10022)
7. [Segment Any Events via Weighted Adaptation of Pivotal Tokens](https://arxiv.org/abs/2312.16222)
8. [LightGlue: Local Feature Matching at Light Speed](https://arxiv.org/abs/2306.13643)