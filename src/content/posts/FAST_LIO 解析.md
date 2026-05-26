---
title: FAST_LIO 解析
published: 2025-02-07
updated: 2026-05-26
description: 'FAST_LIO 激光惯性里程计算法解析，包括 Livox 雷达驱动配置与话题发布'
image: ''
tags: [ROS, FAST_LIO, LIO, LiDAR, SLAM]
category: 'ROS'
draft: false
---

[项目地址](https://github.com/hku-mars/FAST_LIO)
## 解析
- 如果使用的是livox雷达，需按照雷达类型使用雷达的驱动功能包
    - [livox_ros_driver2](https://github.com/Livox-SDK/livox_ros_driver2)
    - [livox_ros_driver](https://github.com/Livox-SDK/livox_ros_driver)
- 雷达的驱动功能包还对应不同的SDK
    - **[Livox-SDK](https://github.com/Livox-SDK/Livox-SDK)**
    - **[Livox-SDK2](https://github.com/Livox-SDK/Livox-SDK2)**
- fast_lio默认使用的是livox avia因此使用mid-360雷达的话需要更改fast_lio源码，即把livox-ros-driver改成livox-ros-driver2
- 在与mid-360用网口连接时，需要在设置中设置有线网连接，写入一个ip地址，这个ip地址便是host主机的host ip，在livox-ros-driver2功能包中需更改配置文件，将host ip改成设置的ip地址
- 而雷达的ip地址默认为192.168.1.100，见[livox_ros_driver2](https://github.com/Livox-SDK/livox_ros_driver2)，有时会不同，使用`ping <lidar ip>`验证是否可与雷达进行连接。
- 所有发布方![1.png](/images/1.png)![Pasted-image-20250207130715.png](/images/Pasted-image-20250207130715.png)
- PCD文件保存![Pasted-image-20250207130005.png](/images/Pasted-image-20250207130005.png)
- 话题"/cloud_registered"发布程序![Pasted-image-20250207130622.png](/images/Pasted-image-20250207130622.png)
- 话题"/cloud_registered_body"发布程序![Pasted-image-20250207130756.png](/images/Pasted-image-20250207130756.png)
- ![Pasted-image-20250207131000.png](/images/Pasted-image-20250207131000.png)
