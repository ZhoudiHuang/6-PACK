# 6-PACK

<p align="center">
    <img src ="assets/pullfig.png" width="1000" />
</p>

## Table of Content
- [Overview](#overview)
- [Requirements](#requirements)
- [Dataset](#dataset)
- [Training](#training)
- [Evaluation](#evaluation)
- [Trained Checkpoints](#trained-checkpoints)
- [Inference](#inference)
- [Citations](#citations)
- [License](#license)

## Overview

This repository is the implementation code of the paper "6-PACK: Category-level 6D Pose Tracker with Anchor-Based Keypoints" ([arXiv](https://arxiv.org/abs/1910.10750), [Project](https://sites.google.com/view/6packtracking), [Video](https://youtu.be/INBjNZsnfy4)) by Wang et al. at [Stanford Vision and Learning Lab](http://svl.stanford.edu/) and [Shanghai Jiao Tong University MVIG Lab](http://mvig.sjtu.edu.cn/). The model is used for category-level 6D object pose tracking. The architecture is designed based on prior work [DenseFusion](https://github.com/j96w/DenseFusion). In this repo, we provide our full implementation code of training/evaluation/inference of the 6-PACK model.


## Requirements

* Python 2.7/3.5/3.6
* [PyTorch 0.4.1](https://pytorch.org/) (PyTroch 1.0 branch is on schedule)
* PIL
* scipy
* numpy
* logging
* CUDA 7.5/8.0/9.0


## Dataset

This work is tested on:

* [NOCS-REAL275](https://github.com/hughw19/NOCS_CVPR2019): Training and Testing sets follow [NOCS](https://arxiv.org/abs/1901.02970). The dataset contains 6 object categories: bottle, bowl, camera, can, laptop and mug. The training set includes 275K discrete frames of synthetic data generated with 1085 models of instances of the classes from [ShapeNetCore](https://www.shapenet.org/) and 7 real videos depicting in total 3 instances of objects for each category. The testing set has 6 real videos depicting in total 3 different (unseen) instances for each object category with 3,200 frames in total.

Please follow these steps to get the same dataset as we do:

1. Download the `CAMERA Dataset Training`, `Real Dataset Training`, `Real Dataset Test` and `Ground truth pose annotation Val&Real_test` from the original [NOCS dataset](https://github.com/hughw19/NOCS_CVPR2019). 

2. Create a folder named `My_NOCS`. Inside this folder, create another folder named `data/` and unzip all the downloaded zip files into this folder. After this step, you should have these directories: `My_NOCS/data/real_train`, `My_NOCS/data/real_val`, `My_NOCS/data/train` and `My_NOCS/data/gts`.

3. You will then find that the training set of NOCS dataset doesn't contain 6D pose labels but with ground truth NOCS-space. Therefore, we have to generate the 6D pose labels for training data from the NOCS-space. The pose generation provided in the NOCS code is calculated by randomly selecting several pixels from the segment of the object in the NOCS-space. We found that the ground truth of real training data generated by this method is not good enough. So we change the random selection to selecting pixels close to the center of the segment. The results are still not perfect but are better. You can download our generated labels and scale information for each object from [NOCS-REAL275-additional.zip](https://drive.google.com/drive/folders/1FWNYJ1E7nMJe7FSK_jVD8YsuMrAVhmvJ?usp=sharing).

4. Unzip all the components of [NOCS-REAL275-additional.zip](https://drive.google.com/drive/folders/1FWNYJ1E7nMJe7FSK_jVD8YsuMrAVhmvJ?usp=sharing) into your `My_NOCS/` folder. After this step, you should have these directories: `My_NOCS/data_list`, `My_NOCS/data_pose`, `My_NOCS/model_pts` and `My_NOCS/model_scales`. Here the `model_pts` are the object pointclouds copied from [ShapeNetCore](https://www.shapenet.org/). The `model_scales` is calculated by us through finding the smallest fit 3D bbox on the object pointclouds.

5. Then, please also run `python data_preprocess.py` in the `6-PACK/dataset/` folder to generate the `_bbox.txt` files we use for the training.

6. We also follow NOCS to use [COCO Dataset](http://cocodataset.org) for data augmentation. Download [COCO Train Images 2017](http://cocodataset.org/#download) and move the `train2017/` folder into `My_NOCS/` folder.

Now you are having the exact same dataset as we do. All of our models for benchmarking and real robot experiments are trained on this dataset.


## Training

Please run:
```bash
python train.py --category NUM
```
where `NUM` is the category number you want to train.

**Checkpoints and Resuming**: After the training of each 100 instances, a `model_current_(category_name).pth` checkpoint will be saved. You can use it to resume the training. After each testing epoch, if the average distance result is the best so far, a `model_(epoch)_(best_score)_(category_name).pth` checkpoint will be saved. You can use it for the evaluation.

**Data Augmentation**: Since the training dataset is mainly made up of synthetic data which is not video sequence. To train our tracking model, we use our own data augmentation method by randomly changing the relative pose of one certain object from two different frames. The details could be found in `dataset/dataset_nocs.py`. We also randomly select COCO images and use as the background.


## Evaluation

Since our method is tracking which requires pose label of the first frame, for fair comparison, we place a 4cm translation noise to the given first frame pose and let our model self-adjust this initial error. Also, we run evaluation 5 times and calculated the mean score as our final evaluation results. In each frame, a sampling method similar to particle filtering is also used to improve the tracking stability.

Please run:
```
python eval.py
```

After generating the estimated pose results of each frame, please run:

```
python benchmark.py
```

Then you will get the evaluation results in four metrics: (1) <5cm5° percentage, (2) IoU25, (3) Rotation error, (4) Translation error.

**Notice**: You might already figure out that the 5cm5° metric used in NOCS is mAP instead of smaller percentage. However, the referred works in NOCS paper about this metric are [Shotton et al. CVPR2013] and [Li et al. ECCV2018] which are using the smaller percentage metric. Also, since our method is not detection-based, we could not calculate mAP results. Therefore, we re-tested the NOCS results provided by the author of NOCS in <5cm5° percentage metric and reported the highest score of all its variants.


## Trained Checkpoints
You can download our trained checkpoints used for benchmarking and real robot experiments from [Link](https://drive.google.com/drive/folders/1FWNYJ1E7nMJe7FSK_jVD8YsuMrAVhmvJ?usp=sharing).

## Inference
After preparing the data and fill in some information blanks in `inference.py`, please run:
```sh
python inference.py
```

For ROS users, you can change the `inference.py` into a TransformBroadcaster by:
```python
import rospy
import tf

br = tf.TransformBroadcaster()
br.sendTransform((-current_t[0] / 1000.0, -current_t[1] / 1000.0, current_t[2] / 1000.0),
                 tf.transformations.quaternion_from_matrix(current_r),
                 rospy.Time.now(),
                 "tracked_obj",
                 "head_rgbd_sensor_rgb_frame")
```

Each time when you want to get the current 6D pose of the object, run:
```python
lr = tf.TransformListener()
(trans, rot) = lr.lookupTransform('tracked_obj', THE_REFERENCE_FRAME, rospy.Time(0))
```

## Citations
Please cite [6-PACK](https://arxiv.org/abs/1910.10750) if you use this repository in your publications:
```
@article{wang20196-pack,
  title={6-PACK: Category-level 6D Pose Tracker with Anchor-Based Keypoints},
  author={Wang, Chen and Mart{\'\i}n-Mart{\'\i}n, Roberto and Xu, Danfei and Lv, Jun and Lu, Cewu and Fei-Fei, Li and Savarese, Silvio and Zhu, Yuke},
  booktitle={International Conference on Robotics and Automation (ICRA)},
  year={2020}
}
```

## License
Licensed under the [MIT License](LICENSE)
