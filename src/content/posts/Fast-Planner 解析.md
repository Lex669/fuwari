---
title: Fast-Planner 解析
published: 2025-05-08
updated: 2026-05-26
description: 'Fast-Planner 路径规划算法解析，包括 launch 配置、话题设置与仿真启动'
image: ''
tags: [ROS, Fast-Planner, Path Planning, UAV]
category: 'ROS'
draft: false
---

[项目地址](https://github.com/HKUST-Aerial-Robotics/Fast-Planner)
## 解析
- 启动仿真
    - rviz.launch![Pasted-image-20250210135349.png](/images/Pasted-image-20250210135349.png)调用了config文件中的traj.rviz配置文件一共有三个配置文件可供选择，如图![Pasted-image-20250210135442.png](/images/Pasted-image-20250210135442.png) 
    - kino_replan.launch
        - kino_algorithm.xml
            - `<arg name="odometry_topic" value="$(arg odom_topic)"/>` `<arg name="odom_topic" value="/state_ukf/odom" />` odometry话题即里程计话题通常由LIO或VIO算法给出，可参考[[FAST_LIO]]
            - `<arg name="camera_pose_topic" value="/pcl_render_node/camera_pose"/>`
            - `<arg name="depth_topic" value="/pcl_render_node/depth"/>`
            - `<arg name="cloud_topic" value="/pcl_render_node/cloud"/>`
            - ![Pasted-image-20250210142804.png](/images/Pasted-image-20250210142804.png)注意看下面的注释，**don't set cloud_topic if you already set these ones!** 以及**don't set camera pose and depth, if you already set this one!** 这两句话需注意，*意思是如果设置了camera_pose_topic和depth_topic则不需再设置cloud_topic* ,反之*设置了cloud_topic则不需设置另外两个话题了*
            - fast_planner_node
                - fast_planner_node.cpp
                - kino_replan_fsm.cpp
                - topo_replan_fsm.cpp
                - planner_manager.cpp
        - traj_server
        - waypoint_generator
        - simulator.xml
