---
title: VINS-Mono
published: 2025-05-19
updated: 2026-05-26
description: 'VINS-Mono 视觉惯性里程计算法使用笔记，包括话题输出与 PX4 视觉定位配置'
image: ''
tags: [ROS, VINS-Mono, VIO, SLAM]
category: 'ROS'
draft: false
---

[github项目地址](https://github.com/HKUST-Aerial-Robotics/VINS-Mono)

```bash
    roslaunch vins_estimator realsense_color.launch 
    roslaunch vins_estimator vins_rviz.launch
    rosbag play YOUR_PATH_TO_DATASET/MH_01_easy.bag 
```

`VINS-Mono`输出的话题中有
```sh
/vins_estimator/camera_pose
/vins_estimator/odometry
```

两个话题的消息类型均为 `nav_msgs/odometry`，因此两个话题都是里程计.
区别是 `camera_pose`的方向与实际方向差了90度，而 `odometry`的方向与实际方向相同，因此我们把 `/vins_estimator/odometry`话题输入到mavros中的视觉定位话题中。

>**注意，需要在地面站中修改 `EKF2_AID_MSK`参数**
