# Stable Head Pose Estimation via 3D Dense Face Reconstruction

![FaceReconstructionDemo](https://s3.ax1x.com/2021/01/06/sZVyhq.gif)

[![Language grade: Python](https://img.shields.io/lgtm/grade/python/g/1996scarlet/Dense-Head-Pose-Estimation.svg?logo=lgtm&logoWidth=18)](https://lgtm.com/projects/g/1996scarlet/Dense-Head-Pose-Estimation/context:python)
[![License](https://badgen.net/github/license/1996scarlet/Dense-Head-Pose-Estimation)](LICENSE)
[![ECCV](https://badgen.net/badge/ECCV/2020/red)](https://www.ecva.net/papers/eccv_2020/papers_ECCV/html/3162_ECCV_2020_paper.php)

Reimplementation of [(ECCV 2020) Towards Fast, Accurate and Stable 3D Dense Face Alignment](https://github.com/cleardusk/3DDFA_V2) via Tensorflow Lite framework, face mesh, head pose, landmarks, and more.

* CPU real-time face deteciton, alignment, and reconstruction pipeline.
* Lightweight render library, 5x faster (3ms vs 15ms) than the [Sim3DR](https://github.com/cleardusk/3DDFA_V2/tree/master/Sim3DR) tools.
* Camera matrix and dense/sparse landmarks prediction via a single network.
* Generate facial parameters for robust head pose and expression estimation.

## Setup

### Basic Requirements

* Python 3.6+
* `pip3 install -r requirements.txt`

### Render for Dense Face

* GCC 6.0+
* `bash build_render.sh`
* **(Cautious)** For Windows user, please refer to [this tutorial](https://stackoverflow.com/questions/1130479/how-to-build-a-dll-from-the-command-line-in-windows-using-msvc) for more details.

## 3D Facial Landmarks

In this project, we perform dense face reconstruction by 3DMM parameters regression.
The regression target is simplified as camera matrix (**C**, shape of 3x4), appearance parameters (**S**, shape of 1x40), and expression variables (**E**, shape of 1x10), with 62 dimensions in total.

The sparse or dense facial landmarks can be estimated by applying these parameters to a predefined 3D model, such as [BFM](https://faces.dmi.unibas.ch/bfm/main.php?nav=1-1-0&id=details).
More specifically, the following formula describes how to generate a face through parameters:

<p align="center">
  <img alt="Generate Face" src="https://latex.codecogs.com/svg.latex?F=U_{base}+S\cdot%20W_{shp}+E\cdot%20W_{exp}">
</p>

where **U** and **W** are from pre-defined face model.
Combine them linearly with parameters to generate sparse or dense faces.
Finally, we need to integrate the posture information into the result:

<p align="center">
  <img alt="With matrix" src="https://latex.codecogs.com/svg.latex?F=R\cdot%20F+T">
</p>

where **R** (shape of 3x3) and **T** (shape of 3x1) denote rotation and translation matrices, respectively, which are fractured from the camera matrix **C**.

### Sparse

<p align="center">
  <img alt="sparse demo" src="https://s3.ax1x.com/2021/01/10/slO9je.gif">
</p>

Since we have reconstructed the entire face, the 3D face alignment can be achieved by selecting the landmarks at the corresponding positions. See [[TPAMI 2017] Face alignment in full pose range: A 3d total solution](https://arxiv.org/abs/1804.01005) for more details.

Comparing with the method of first detecting 2D landmarks and then performing depth estimation, directly fitting 3DMM to solve 3D face alignment can not only obtain more accurate results in larger pose scenes, but also has obvious advantages in speed.

We provide a demonstration script that can generate 68 landmarks based on the reconstructed face. Run the following command to view the real-time 3D face alignment results:

``` bash
python3 demo_video.py -m sparse -f <your-video-path>
```

### Dense

<p align="center">
  <img alt="dense demo" src="https://s3.ax1x.com/2021/01/09/sQ01VP.gif">
</p>

Currently, our method supports up to 38,365 landmarks.
We draw landmarks every 6 indexes for a better illustration.
Run the demonstrate script in **dense** mode for real-time dense facial landmark localization:

``` bash
python3 demo_video.py -m dense -f <your-video-path>
```

## Face Reconstruction

Our network is multi-task since it can directly regress 3DMM params from a single face image for reconstruction as well as estimating the head pose via **R** and **T** prediction.

During training, the predicted **R** can supervise the params regression branch to generate more precise pose parameters.
Meanwhile, the landmarks calculated via the params are provided as the labeled data to the pose estimation branch for training, through the SolvePnP tool.

<!-- 整个网络的头部姿态估计任务本质上是半监督的, 只需要少量的初始化带标签数据即可激活整个训练过程

我们的方法结合了

不仅利用了PNP, 并且通过动态拟合3d模型避免了先验模型精度较低所导致的姿态估计不准的问题 -->

### Head Pose

![Head Pose](https://s3.ax1x.com/2021/01/14/sdfSJI.gif)

Traditional head pose estimation approaches, such as [Appearance Template Models](https://www.researchgate.net/publication/2427763_Face_Recognition_by_Support_Vector_Machines), [Detector Arrays](https://ieeexplore.ieee.org/document/609310), and [Mainfold Embedding](https://ieeexplore.ieee.org/document/4270305) have been extensively studied.
However, methods based on deep learning improved the prediction accuracy to meet actual needs, until recent years.

Given a set of predefined 3D facial landmarks and the corresponding 2D image projections, the SolvePnP tool can be utilized to calculate the rotation matrix.
However, the adopted mean 3D human face model usually introduces intrinsic error during the fitting process.
Meanwhile, the additional landmarks extraction component is also kind of cumbersome.

<!-- 6DoF

R 提供了3个自由度pitch, raw, roll; T提供了另外3个x, y, z

与原始模型不同的是, 我们的模型能够为输入的每张人脸图像直接预测其旋转矩阵R和平移矩阵T, 以及稠密或稀疏的特征点.

区别于先预测特征点,再基于先验3D模型并使用SolvePNP估计R和T的方法, 通过神经网络直接输出相机矩阵的方法有更强的泛化性以

直接预测相机矩阵的方法能够显著降低网络训练成本,

| Method | Yaw | Pitch | Roll | MAE |
| :-: | :-: | :-: | :-: | :-: |
| 3DDFA_V1  | 0.23ms  | 7.79ms | 0.39ms | 3.92ms |
| 3DDFA_V2  | 0.23ms  | 7.79ms | 0.39ms | 3.92ms |
| FSA-Net  | 0.23ms  | 7.79ms | 0.39ms | 3.92ms |
| Ours  | 0.23ms  | 7.79ms | 0.39ms | 3.92ms | -->

``` bash
python3 demo_video.py -m pose -f <your-video-path>
```

### Expression

![Expression](https://s3.ax1x.com/2021/01/06/sZV0BQ.jpg)

<!-- As

由于我们的方法能够预测稠密的人脸特征点以及姿态信息,

因此也可以进行粗略的表情估计, -->

Run the `demo_image.py` script for expression rederring:

``` bash
python3 demo_image.py <your-image-path>
```

### Mesh

``` bash
python3 demo_video.py -m mesh -f <your-video-path>
```

| Scheme | THREAD=1 | THREAD=2 | THREAD=4 |
| :-: | :-: | :-: | :-: |
| Inference  | 7.79ms  | 6.88ms | 5.83ms |

``` bash
python3 video_speed_benchmark.py <your-video-path>
```

| Stage | Preprocess | Inference | Postprocess | Render |
| :-: | :-: | :-: | :-: | :-: |
| Each face cost  | 0.23ms  | 7.79ms | 0.39ms | 3.92ms |

## Citation

``` bibtex
@inproceedings{guo2020towards,
    title={Towards Fast, Accurate and Stable 3D Dense Face Alignment},
    author={Guo, Jianzhu and Zhu, Xiangyu and Yang, Yang and Yang, Fan and Lei, Zhen and Li, Stan Z},
    booktitle={Proceedings of the European Conference on Computer Vision (ECCV)},
    year={2020}
}
```
