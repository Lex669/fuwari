---
title: VINS-Fusion
published: 2025-05-19
updated: 2026-05-26
description: 'VINS-Fusion 视觉惯性里程计算法使用笔记'
image: ''
tags: [ROS, VINS-Fusion, VIO, SLAM]
category: 'ROS'
draft: false
---

```sh
    roslaunch vins vins_rviz.launch
    rosrun vins vins_node ~/catkin_ws/src/VINS-Fusion/config/euroc/euroc_mono_imu_config.yaml 
    (optional) rosrun loop_fusion loop_fusion_node ~/catkin_ws/src/VINS-Fusion/config/euroc/euroc_mono_imu_config.yaml 
    rosbag play YOUR_DATASET_FOLDER/MH_01_easy.bag
```

**VINS-Fusion**效果没有**VINS-MONO**好，建议使用 `VINS-Mono`
