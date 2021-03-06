## Overview

In this section, we will introduce the main units of training a detector:
data loading, model and iteration pipeline.

## Data loading

Following typical conventions, we use `Dataset` and `DataLoader` for data loading
with multiple workers. `Dataset` returns a dict of data items corresponding
the arguments of models' forward method.
Since the data in object detection may not be the same size (image size, gt bbox size, etc.),
we introduce a new `DataContainer` type in `mmcv` to help collect and distribute
data of different size.
See [here](https://github.com/open-mmlab/mmcv/blob/master/mmcv/parallel/data_container.py) for more details.

## Model

In mmdetection, model components are basically categorized as 4 types.

- backbone: usually a FCN network to extract feature maps, e.g., ResNet.
- neck: the part between backbones and heads, e.g., FPN, ASPP.
- head: the part for specific tasks, e.g., bbox prediction and mask prediction.
- roi extractor: the part for extracting features from feature maps, e.g., RoI Align.

We also write implement some general detection pipelines with the above components,
such as `SingleStageDetector` and `TwoStageDetector`.

### Build a model with basic components

Following some basic pipelines (e.g., two-stage detectors), the model structure
can be customized through config files with no pains.

### Write a new model

To write a new detection pipeline, you need to inherit from `BaseDetector`,
which defines the following abstract methods.

- `extract_feat()`: given an image batch of shape (n, c, h, w), extract the feature map(s).
- `forward_train()`: forward method of the training mode
- `simple_test()`: single scale testing without augmentation
- `aug_test()`: testing without augmentation (multi-scale, flip, etc.)

[TwoStageDetector](https://github.com/hellock/mmdetection/blob/master/mmdet/models/detectors/two_stage.py)
is a good example which shows how to do that.

## Iteration pipeline

We adopt distributed training for both single machine and multiple machines.
Supposing that the server has 8 GPUs, 8 processes will be started and each process runs on a single GPU.

Each process keeps an isolated model, data loader, and optimizer.
Model parameters are only synchronized once at the begining.
After a forward and backward pass, gradients will be allreduced among all GPUs,
and the optimizer will update model parameters.
Since the gradients are allreduced, the model parameter stays the same for all processes after the iteration.